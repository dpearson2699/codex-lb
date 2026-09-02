# GitNexus Engineering Plan

> Task: Cache the upstream SSL context process-wide and stop the wedged-teardown reclaim from firing on teardowns that already completed successfully (Soju06/codex-lb#2029).
> Evidence verified at commit 93247be4e80834173c283d579d71fbbef04c4b53 (branch fix/ssl-context-cache-teardown-reclaim; code identical to upstream main 665e58e3, the only delta is this plan file); GitNexus index built this session at 665e58e3, 1 docs-only commit behind, refresh skipped (no code changed); freshness: accept.
> Evidence provenance schema 2; global dirty digest 0a9c85780067d9afcd0764f307b60891e3cee927ee11eaeb5ec7826d10fd82cd; cited-path manifest 10 sorted entries; exact generated plan path excluded.
> Revision 2 (Deepen): folds the cross-model plan critiques — Codex gpt-5.6-sol (xhigh, ephemeral, 2026-09-02) and ChatGPT Pro (GitHub connector, conversation 6a97a482…) — both returned "needs-changes"; arbitration recorded in §12.

## Objective (§1)

Remove the per-turn synchronous CA-bundle load from the event loop, and make the bounded SQLite teardown skip the reclaim only when the abandoned rollback/close has already **completed successfully** by the time the reclaim would run — i.e. the event loop, not the aiosqlite worker, was late. Failed, cancelled, and still-pending teardowns keep the existing reclaim unchanged. Both are bug fixes against the observed cascade in #2029 (74.8 s loop stall, 13/13 wedges inside lag windows, 8/13 `interrupt()` failures on already-closed connections).

## Current Behaviour (§2–3)

- `app/core/clients/http.py:104-107` `_build_ssl_context()` = `ssl.create_default_context()` + `load_verify_locations(cafile=certifi.where())`, uncached [verified]. Callers (context, exact): `_build_http_client` (http.py:176), `create_codex_session` (codex.py:435, function-local import), `_socks_proxy_connector` (codex.py:520, function-local import), `_probe_upstream_proxy_endpoint` (settings/api.py:570, module-level import at api.py:22) [verified].
- `create_codex_session()` (codex.py:432-441) builds a fresh `TCPConnector(ssl=_build_ssl_context())` + `ClientSession` per call; **16** call sites use `codex_client or CodexClient(create_codex_session())` and `app/modules/proxy/**` never injects `codex_client` (0 grep hits), all on the loop thread → every upstream turn pays the CA load on the loop [verified]. Startup `init_http_client()` (http.py:340-347) runs in lifespan before serving [verified].
- `_safe_rollback`/`_safe_close` (session.py:803-845) call `_shielded_bounded` (session.py:522-553), which returns the still-pending task once the wall-clock deadline passes; the caller then always runs `_reclaim_wedged_sqlite_session` (session.py:696-769): fences the session, registers the task, `driver.interrupt()`, `connection.invalidate()`, and attaches `_finish_abandoned_teardown` [verified]. A worker completion delivered via `call_soon_threadsafe` can be queued behind the timeout callback, so an immediate `done()` check at reclaim entry can still read `False` although the worker already closed the connection [inferred, both critiques].
- SQLAlchemy `SessionTransaction.close()` clears `session._transaction` before iterating held connections; an exception mid-close can leave a connection that `_safe_close` cannot rediscover, and `NullPool.dispose()` is a no-op, so a failed/cancelled teardown must keep the existing reclaim [graph/inferred, Codex-verified against the installed SQLAlchemy].

## Findings (§4–5)

- `gitnexus impact _build_ssl_context -d upstream --depth 1` → `risk: LOW, epistemic: exact, direct: 4` [graph, source-confirmed].
- `gitnexus impact _reclaim_wedged_sqlite_session -d upstream --depth 1` → `risk: CRITICAL, direct: 2 (_safe_rollback, _safe_close), processes_affected: 50`; the change must be a guarded early return in the callers, leaving the reclaim function's wedged path untouched [graph, source-confirmed].
- Tests located [verified]: `tests/unit/test_http_client.py:60` `test_init_http_client_creates_tcp_connector_with_limits` (patches `_build_ssl_context`, asserts `call_count == 1` and context identity — the only call-count assertion); `:153`, `:201`, `:250` patch the builder without asserting; `:327` `test_build_ssl_context_preserves_default_roots_and_adds_certifi_bundle` (builder stays uncached). `tests/unit/test_codex_client.py:802` `test_socks_websocket_uses_proxy_connector_and_closes_session` (patches `ProxyConnector`, asserts `ssl` present). `tests/integration/test_settings_api.py:779` `test_upstream_proxy_endpoint_test_probes_socks_proxy` (patches `app.modules.settings.api.ProxyConnector`, captures kwargs). `tests/unit/test_db_session.py:1275` (wedged-rollback harness: `release_wedge` Event, interrupt spy, `other_writer`), `:1597` `test_reclaim_interrupts_the_real_aiosqlite_driver_without_a_spy` (task is *pending* at reclaim entry — `ensure_future` does not run eagerly; keep as-is, add `assert not abandoned.done()`).
- OpenSpec [verified]: no requirement covers SSL-context construction or the SQLite teardown reclaim; `openspec/specs/outbound-http-clients/spec.md:440` (connector persistence) and `openspec/specs/database-backends/spec.md:162` (file-backed SQLite engines) are the neighbouring requirements. `openspec` CLI is available via `npx -y @fission-ai/openspec@latest` (1.11.0) [verified].
- PDG slice: not built (`freshness: accept`; both central functions fully source-read) [inferred].

## Proposed Changes (§6)

1. `app/core/clients/http.py` — add `shared_ssl_context()` (`functools.cache`, returns `_build_ssl_context()`); `_build_ssl_context` stays the uncached builder/test seam. `_build_http_client` uses `shared_ssl_context()`. The context is immutable after publication (no runtime code mutates it); trust-root changes take effect on restart — both stated in the OpenSpec context.
2. `app/core/clients/codex.py` — `create_codex_session` and `_socks_proxy_connector` import `shared_ssl_context` (keep the function-local import so patching `app.core.clients.http.shared_ssl_context` keeps working).
3. `app/modules/settings/api.py` — import and use `shared_ssl_context` (module-level binding: tests patch `app.modules.settings.api.shared_ssl_context`).
4. `app/db/session.py` — new helper `_teardown_completed_after_bound(task) -> bool`: after `_shielded_bounded` returned the pending task, wait on it once more for `_SQLITE_TEARDOWN_GRACE_SECONDS = 0.25` inside `anyio.CancelScope(shield=True)` via `asyncio.wait`, so a worker completion queued behind the timeout callback is observed; return `True` only when `task.done() and not task.cancelled() and task.exception() is None`. `_safe_rollback`/`_safe_close` record `started = loop.time()` immediately before `_shielded_bounded`; when the helper returns `True` they log one WARNING `sqlite_teardown_bound_elapsed_but_completed phase=%s bound_seconds=%.1f elapsed_seconds=%.1f — the %s completed before the reclaim ran; nothing is held (issue #2029)` and return **without** calling `_reclaim_wedged_sqlite_session` (no fence, no registry entry, no interrupt, no invalidate; `close_session` then runs the normal `_safe_close`). Otherwise call the existing reclaim, passing `elapsed_seconds` (required keyword) so the existing `sqlite_wedged_teardown` line also carries `elapsed_seconds=`.
5. `app/db/session.py` — **no** closed-connection classifier (dropped after arbitration, see §12): the `interrupt()` except branch keeps its traceback warning, and the invalidate branch's outcome is logged at WARNING when `connection.invalidate()` raises (was DEBUG) so #1981-style holds stay diagnosable.
6. `openspec/changes/prevent-ssl-starvation-false-reclaim/` — `proposal.md` (one incident chain: trigger + recovery), `tasks.md`, `specs/outbound-http-clients/spec.md` (ADDED: during normal startup the process SHALL construct one process-scoped outbound SSL context before serving; shared HTTP/WebSocket connectors, Codex direct sessions, Codex SOCKS sessions and settings SOCKS probes SHALL receive that exact instance; runtime code SHALL NOT call the uncached builder or mutate the published context; trust-root changes apply after restart), `specs/database-backends/spec.md` (ADDED: when an abandoned rollback/close has completed successfully before reclaim begins, reclaim SHALL NOT fence, register, interrupt, or invalidate it and SHALL emit one warning with phase, bound and elapsed seconds; a failed or cancelled terminal task SHALL NOT receive the exemption; a task still pending SHALL keep the existing fence/registry/interrupt/invalidate/late-cleanup behaviour).

## Implementation Sequence (§7)

1. OpenSpec change folder (§6.6); validate: `npx -y @fission-ai/openspec@latest validate prevent-ssl-starvation-false-reclaim --strict` and `npx -y @fission-ai/openspec@latest validate --specs`.
2. Tests first (red where marked R): `tests/unit/test_http_client.py` — autouse fixture clearing `shared_ssl_context` **before and after** each test; `test_shared_ssl_context_builds_once_across_client_generations` (R: `init_http_client` then `refresh_http_client` → builder called once, all four connectors get the same identity); `test_create_codex_session_uses_the_shared_ssl_context` (R) and `test_socks_proxy_connector_uses_the_shared_ssl_context` (R) asserting the exact sentinel; `tests/integration/test_settings_api.py::test_upstream_proxy_endpoint_test_probes_socks_proxy` extended to patch `app.modules.settings.api.shared_ssl_context` and assert the exact sentinel (R). `tests/unit/test_db_session.py` — `test_close_session_skips_reclaim_when_the_rollback_completes_after_the_bound` (R; production `_shielded_bounded` + `close_session`; bound 0.2 s; the wedged rollback is released by a `threading.Timer` via `loop.call_soon_threadsafe` while the test blocks the loop with `time.sleep(0.4)`, so the worker completion is queued behind the timeout; assert no interrupt, no invalidate, no fence, no registry entry, `other_writer` INSERT succeeds, normal close ran, the new log line present); same for phase=close (R); `test_reclaim_still_runs_for_a_rollback_that_failed_after_the_bound` and `..._cancelled_after_the_bound` (guards, may pass on main); elapsed propagation test with a patched monotonic clock (R); `assert not abandoned.done()` added to `:1597`.
3. Implement §6.1–6.3 (SSL tests green).
4. Implement §6.4–6.5 (teardown tests green). Constraint: the early return lives in `_safe_rollback`/`_safe_close` before `_reclaim_wedged_sqlite_session` is entered; the reclaim body keeps its fence-before-first-await ordering.
5. Verify: `uv run pytest tests/unit/test_http_client.py tests/unit/test_db_session.py tests/unit/test_codex_client.py tests/unit/test_defer_cancellation_shield_leak.py tests/unit/test_proxy_websocket_client.py tests/integration/test_settings_api.py -q`; `make lint`; `make typecheck`; `uv run pre-commit run local-ci --hook-stage manual --all-files`.
6. Local product check on this Mac: build + `codex-lbctl` install from the branch, watch `launchd-error.log` for `sqlite_teardown_bound_elapsed_but_completed` vs `sqlite_wedged_teardown`, and `sample` the main thread for `set_default_verify_paths` (expected: absent after startup).
7. PR against `Soju06/codex-lb` main: `fix(db): share one upstream SSL context and skip reclaim for completed SQLite teardowns`, body per template, `Fixes #2029`; then current-head gates: CI Required green, actionable CodeRabbit threads addressed, `mergeable=CLEAN`.

## Test Strategy (§8)

- Update: `tests/unit/test_http_client.py` (cache reset fixture), `tests/unit/test_db_session.py:1597` (`assert not abandoned.done()`), `tests/integration/test_settings_api.py:779` (sentinel assertion).
- Add: the scenarios in §7.2. Not every guard test is red on main (done-with-exception, cancelled); they exist to prevent `done()` from becoming the semantic test.
- Commands: as in §7.5.

## Implementation Context (§11)

```yaml
implementation_context:
  task_summary: Add shared_ssl_context() (functools.cache) and route the four connector factories through it; in _safe_rollback/_safe_close, after the bound expires, observe the abandoned task for a short shielded grace and skip _reclaim_wedged_sqlite_session only when it completed successfully, logging phase/bound/elapsed; pass elapsed_seconds into the existing reclaim log; OpenSpec change prevent-ssl-starvation-false-reclaim.
  acceptance_criteria:
    - During normal application startup one process-scoped outbound SSLContext is built before serving, and _build_http_client, create_codex_session, _socks_proxy_connector and the settings SOCKS probe all receive that exact instance (one builder call across client generations).
    - A rollback/close that completed successfully before the reclaim would run is not fenced, registered, interrupted, or invalidated; one warning carries phase, bound_seconds, elapsed_seconds; normal close still follows a rollback.
    - A failed or cancelled terminal task, and a task still pending after the grace, take the existing reclaim path unchanged.
    - interrupt()/invalidate() failures keep traceback-bearing diagnostics (no classifier).
  evidence_provenance: {
  "schema_version": 2,
  "head_commit": "93247be4e80834173c283d579d71fbbef04c4b53",
  "generated_plan_path": "docs/plans/2026-09-02-gitnexus-plan-ssl-context-cache-reclaim.md",
  "global_dirty_digest": {
    "algorithm": "sha256",
    "canonicalization": "gitnexus-evidence-provenance-v2 NUL-framed UTF-8 records",
    "value": "0a9c85780067d9afcd0764f307b60891e3cee927ee11eaeb5ec7826d10fd82cd"
  },
  "cited_path_manifest": [
    {
      "path": "app/core/clients/codex.py",
      "object_kind": {
        "head": "regular",
        "index": "regular",
        "worktree": "regular",
        "untracked": "absent"
      },
      "state": "clean",
      "rename_from": null,
      "rename_to": null,
      "head_digest": "sha256:99f908360a82a8bf3e4358aa46fb83d6cf68e69ca423ea72916e46259977c3be",
      "index_digest": "sha256:99f908360a82a8bf3e4358aa46fb83d6cf68e69ca423ea72916e46259977c3be",
      "worktree_digest": "sha256:99f908360a82a8bf3e4358aa46fb83d6cf68e69ca423ea72916e46259977c3be",
      "untracked_digest": "absent"
    },
    {
      "path": "app/core/clients/http.py",
      "object_kind": {
        "head": "regular",
        "index": "regular",
        "worktree": "regular",
        "untracked": "absent"
      },
      "state": "clean",
      "rename_from": null,
      "rename_to": null,
      "head_digest": "sha256:5c1ea55505f962556cee88b30462272d43f086d29da3f10bd4f3b8bec8de8258",
      "index_digest": "sha256:5c1ea55505f962556cee88b30462272d43f086d29da3f10bd4f3b8bec8de8258",
      "worktree_digest": "sha256:5c1ea55505f962556cee88b30462272d43f086d29da3f10bd4f3b8bec8de8258",
      "untracked_digest": "absent"
    },
    {
      "path": "app/db/session.py",
      "object_kind": {
        "head": "regular",
        "index": "regular",
        "worktree": "regular",
        "untracked": "absent"
      },
      "state": "clean",
      "rename_from": null,
      "rename_to": null,
      "head_digest": "sha256:c027bf6b4cdb4fbf9aae57ebf3f99db47d50ea5676c6dbc39165a2a87c2793a9",
      "index_digest": "sha256:c027bf6b4cdb4fbf9aae57ebf3f99db47d50ea5676c6dbc39165a2a87c2793a9",
      "worktree_digest": "sha256:c027bf6b4cdb4fbf9aae57ebf3f99db47d50ea5676c6dbc39165a2a87c2793a9",
      "untracked_digest": "absent"
    },
    {
      "path": "app/modules/settings/api.py",
      "object_kind": {
        "head": "regular",
        "index": "regular",
        "worktree": "regular",
        "untracked": "absent"
      },
      "state": "clean",
      "rename_from": null,
      "rename_to": null,
      "head_digest": "sha256:0a7e98c3a032214fa6e5d2bf46190da5394fd4bbe618f38290be1ba212b45bde",
      "index_digest": "sha256:0a7e98c3a032214fa6e5d2bf46190da5394fd4bbe618f38290be1ba212b45bde",
      "worktree_digest": "sha256:0a7e98c3a032214fa6e5d2bf46190da5394fd4bbe618f38290be1ba212b45bde",
      "untracked_digest": "absent"
    },
    {
      "path": "openspec/specs/database-backends/spec.md",
      "object_kind": {
        "head": "regular",
        "index": "regular",
        "worktree": "regular",
        "untracked": "absent"
      },
      "state": "clean",
      "rename_from": null,
      "rename_to": null,
      "head_digest": "sha256:a1cc83bd43809f0a559a6bcbd581c810d96756c9904ffe1410c620e586b481a3",
      "index_digest": "sha256:a1cc83bd43809f0a559a6bcbd581c810d96756c9904ffe1410c620e586b481a3",
      "worktree_digest": "sha256:a1cc83bd43809f0a559a6bcbd581c810d96756c9904ffe1410c620e586b481a3",
      "untracked_digest": "absent"
    },
    {
      "path": "openspec/specs/outbound-http-clients/spec.md",
      "object_kind": {
        "head": "regular",
        "index": "regular",
        "worktree": "regular",
        "untracked": "absent"
      },
      "state": "clean",
      "rename_from": null,
      "rename_to": null,
      "head_digest": "sha256:22e5e653677329d579fe536e930e4adf74fe71e3d5c3d38657b1039e110ce1ff",
      "index_digest": "sha256:22e5e653677329d579fe536e930e4adf74fe71e3d5c3d38657b1039e110ce1ff",
      "worktree_digest": "sha256:22e5e653677329d579fe536e930e4adf74fe71e3d5c3d38657b1039e110ce1ff",
      "untracked_digest": "absent"
    },
    {
      "path": "tests/integration/test_settings_api.py",
      "object_kind": {
        "head": "regular",
        "index": "regular",
        "worktree": "regular",
        "untracked": "absent"
      },
      "state": "clean",
      "rename_from": null,
      "rename_to": null,
      "head_digest": "sha256:b334020c4bb9b7cddb20a4cf67706fa825407b043a16d1ff9fbc277cacea9a21",
      "index_digest": "sha256:b334020c4bb9b7cddb20a4cf67706fa825407b043a16d1ff9fbc277cacea9a21",
      "worktree_digest": "sha256:b334020c4bb9b7cddb20a4cf67706fa825407b043a16d1ff9fbc277cacea9a21",
      "untracked_digest": "absent"
    },
    {
      "path": "tests/unit/test_codex_client.py",
      "object_kind": {
        "head": "regular",
        "index": "regular",
        "worktree": "regular",
        "untracked": "absent"
      },
      "state": "clean",
      "rename_from": null,
      "rename_to": null,
      "head_digest": "sha256:8277b713f9dfb587e4d76e0b27ec7fad62782ac469b1744f977cba59af1eeec5",
      "index_digest": "sha256:8277b713f9dfb587e4d76e0b27ec7fad62782ac469b1744f977cba59af1eeec5",
      "worktree_digest": "sha256:8277b713f9dfb587e4d76e0b27ec7fad62782ac469b1744f977cba59af1eeec5",
      "untracked_digest": "absent"
    },
    {
      "path": "tests/unit/test_db_session.py",
      "object_kind": {
        "head": "regular",
        "index": "regular",
        "worktree": "regular",
        "untracked": "absent"
      },
      "state": "clean",
      "rename_from": null,
      "rename_to": null,
      "head_digest": "sha256:958214e698df238598666c941926d80017e7b22488400b79b40f9b2cc7fd3c02",
      "index_digest": "sha256:958214e698df238598666c941926d80017e7b22488400b79b40f9b2cc7fd3c02",
      "worktree_digest": "sha256:958214e698df238598666c941926d80017e7b22488400b79b40f9b2cc7fd3c02",
      "untracked_digest": "absent"
    },
    {
      "path": "tests/unit/test_http_client.py",
      "object_kind": {
        "head": "regular",
        "index": "regular",
        "worktree": "regular",
        "untracked": "absent"
      },
      "state": "clean",
      "rename_from": null,
      "rename_to": null,
      "head_digest": "sha256:537a7bb460fd681b964b7985f8a29e120f280513576fd0eb3510a6cfe7f7131a",
      "index_digest": "sha256:537a7bb460fd681b964b7985f8a29e120f280513576fd0eb3510a6cfe7f7131a",
      "worktree_digest": "sha256:537a7bb460fd681b964b7985f8a29e120f280513576fd0eb3510a6cfe7f7131a",
      "untracked_digest": "absent"
    }
  ]
}
  files_to_modify:
    - file: app/core/clients/http.py
      symbols: [shared_ssl_context, _build_http_client]
      intended_change: add functools.cache accessor; _build_http_client uses it
    - file: app/core/clients/codex.py
      symbols: [create_codex_session, _socks_proxy_connector]
      intended_change: use shared_ssl_context via function-local import
    - file: app/modules/settings/api.py
      symbols: [_probe_upstream_proxy_endpoint]
      intended_change: import and use shared_ssl_context
    - file: app/db/session.py
      symbols: [_teardown_completed_after_bound, _safe_rollback, _safe_close, _reclaim_wedged_sqlite_session]
      intended_change: grace observation + successful-completion exemption in the callers; elapsed_seconds required keyword on the reclaim; invalidate failure logged at WARNING
    - file: openspec/changes/prevent-ssl-starvation-false-reclaim/
      symbols: []
      intended_change: proposal, tasks, delta specs (outbound-http-clients, database-backends)
  tests:
    - file: tests/unit/test_http_client.py
      scenarios:
        - "init_http_client then refresh_http_client with _build_ssl_context patched → builder called once; all four connectors receive the same sentinel"
        - "create_codex_session → TCPConnector ssl is the exact shared sentinel (patch app.core.clients.http.shared_ssl_context)"
        - "_socks_proxy_connector → ProxyConnector ssl is the exact shared sentinel"
        - "existing test_build_ssl_context_preserves_default_roots_and_adds_certifi_bundle unchanged (builder uncached)"
    - file: tests/integration/test_settings_api.py
      scenarios:
        - "SOCKS probe → captured connector kwargs ssl is the exact sentinel patched at app.modules.settings.api.shared_ssl_context"
    - file: tests/unit/test_db_session.py
      scenarios:
        - "rollback wedged past 0.2 s bound, released from a thread while the loop is blocked → after the loop resumes the grace observation sees success: no interrupt, no invalidate, no fence, no registry entry, other writer INSERT succeeds, normal close ran, sqlite_teardown_bound_elapsed_but_completed logged"
        - "same for phase=close"
        - "rollback that raises after the bound → existing reclaim path (fence + invalidate) still runs"
        - "rollback task cancelled after the bound → existing reclaim path still runs"
        - "elapsed_seconds measured from before _shielded_bounded reaches both log lines (patched loop clock)"
        - "test_reclaim_interrupts_the_real_aiosqlite_driver_without_a_spy: assert the task is pending at entry; behaviour unchanged"
  verification_commands:
    - npx -y @fission-ai/openspec@latest validate prevent-ssl-starvation-false-reclaim --strict
    - npx -y @fission-ai/openspec@latest validate --specs
    - uv run pytest tests/unit/test_http_client.py tests/unit/test_db_session.py tests/unit/test_codex_client.py tests/unit/test_defer_cancellation_shield_leak.py tests/unit/test_proxy_websocket_client.py tests/integration/test_settings_api.py -q
    - make lint
    - make typecheck
    - uv run pre-commit run local-ci --hook-stage manual --all-files
  assumptions:
    - "aiohttp 3.14.3 and aiohttp_socks treat a supplied SSLContext as caller-owned, immutable configuration — check: no repository code mutates it after _build_ssl_context returns (grep check_hostname/verify_mode/load_verify_locations/set_ciphers/set_alpn outside http.py)."
    - "All shared_ssl_context callers run on the loop thread, so functools.cache's first-miss duplication is unreachable — check: grep callers after implementation; startup init_http_client primes it."
    - "A 0.25 s shielded grace is enough for a queued worker completion to be delivered once the loop runs — check: the queued-completion test."
  open_questions:
    - "Whether upstream prefers the grace constant derived from the busy timeout (like the bound) or a literal."
  avoid:
    - Do not repeat full repository discovery
    - Do not replace established patterns without evidence
    - Do not make _build_ssl_context itself cached (tests assert the builder is called once per call)
    - Do not treat Task.done() as success; only a not-cancelled task whose exception() is None skips the reclaim
    - Do not add a closed-connection interrupt classifier (masks the #1981 permanent-hold signature)
    - Do not inject a shared CodexClient across the 16 fallback sites in this PR (deferred follow-up)
    - Do not move _build_http_client off-loop in this PR (deferred follow-up)
```

## Assumptions and Open Questions (§12)

- Arbitration of the two critiques: adopted Codex C1 (grace observation for the queued completion), C2/P1 (success-only exemption), C3/P7 (neutral wording, required elapsed), C6 (keep the real-driver test, assert pending), C7/P4/P5 (all four consumers + cross-generation reuse), C8/P2/P10 (precise MUST scenarios, mandatory strict validation, local-ci and cloud gates), C10 (16 sites, one call-count assertion), P6 (clear before and after, patch consumer-bound names), P9 (slug covers both halves, immutability and restart semantics in context). Divergence: Codex wanted the closed-connection classifier removed, Pro wanted a strict allow-list with near-miss tests; the root removed it — with the success-only exemption the false-positive tracebacks disappear on their own, and an allow-list adds surface for no reachable case.
- [assumed] see `assumptions` in §11.
- Deferred follow-ups (not in this PR): inject the shared `CodexClient` from `app/modules/proxy`; build replacement clients in `refresh_http_client*` off-loop; the `NullPool` last-connection WAL checkpoint churn on multi-GB stores; loop-lag attribution in the log line (needs measured lag evidence at the seam).

## Definition of Done (§13)

- Red-marked tests fail on `main` @ 665e58e3 and pass on the branch; the suites in §7.5 pass; `make lint`, `make typecheck`, and `uv run pre-commit run local-ci --hook-stage manual --all-files` clean; both `openspec validate` commands clean.
- OpenSpec change folder `prevent-ssl-starvation-false-reclaim` present with proposal, tasks, two delta specs.
- PR opened against `Soju06/codex-lb` `main` with `Fixes #2029`, template filled; current-head CI Required green, actionable CodeRabbit threads addressed, `mergeable=CLEAN`.
- Local run on this Mac shows no `set_default_verify_paths` on the main thread after startup and no `sqlite_wedged_teardown` for teardowns that completed.
