# GitNexus Engineering Plan

> Task: Cache the upstream SSL context process-wide and stop the wedged-teardown reclaim from firing on teardowns that already finished (Soju06/codex-lb#2029).
> Evidence verified at commit 665e58e316ef72d05ba791669879fa5c92746773; GitNexus index fresh (built this session, `gitnexus status` up-to-date; freshness: accept).
> Evidence provenance schema 2; global dirty digest 0a9c85780067d9afcd0764f307b60891e3cee927ee11eaeb5ec7826d10fd82cd; cited-path manifest 8 sorted entries; exact generated plan path excluded.

## Objective (§1)

Remove the per-turn synchronous CA-bundle load from the event loop, and make `_reclaim_wedged_sqlite_session` a no-op (with an attributable log line) when the abandoned rollback/close has already completed by the time the reclaim runs — i.e. the event loop, not the aiosqlite worker, was starved. Both are bug fixes against the observed cascade in #2029 (74.8 s loop stall, 13/13 wedges inside lag windows, 8/13 `interrupt()` failures on already-closed connections).

## Current Behaviour (§2–3)

- `app/core/clients/http.py:104-107` `_build_ssl_context()` = `ssl.create_default_context()` + `load_verify_locations(cafile=certifi.where())`, uncached [verified]. Callers (context, exact): `_build_http_client` (http.py:176), `create_codex_session` (codex.py:435), `_socks_proxy_connector` (codex.py:520), `_probe_upstream_proxy_endpoint` (settings/api.py:570) [verified].
- `create_codex_session()` (codex.py:433-441) builds a fresh `TCPConnector(ssl=_build_ssl_context())` + `ClientSession` per call; 15 call sites use `codex_client or CodexClient(create_codex_session())` and `app/modules/proxy/**` never injects `codex_client` (0 grep hits) → every upstream turn pays the CA load on the loop [verified].
- `refresh_http_client` (http.py:350-359) and `refresh_http_client_after_network_failure` rebuild via `_build_http_client()` on the loop [verified].
- `_safe_rollback`/`_safe_close` (session.py:803-845) call `_shielded_bounded` (session.py:522-553), which returns the still-pending task after `_SQLITE_TEARDOWN_TIMEOUT_SECONDS` (5 s) of wall-clock; the caller then always runs `_reclaim_wedged_sqlite_session` (session.py:696-769): fences the session, `driver.interrupt()`, `connection.invalidate()`, and registers `_finish_abandoned_teardown` [verified]. Nothing re-checks `abandoned.done()` first, and a starved loop cannot observe completion before the deadline passes.

## Findings (§4–5)

- `gitnexus impact _build_ssl_context -d upstream --depth 1` → `risk: LOW, epistemic: exact, direct: 4` — the four callers above; all become shared-context consumers [graph, source-confirmed].
- `gitnexus impact _reclaim_wedged_sqlite_session -d upstream --depth 1` → `risk: CRITICAL, direct: 2 (_safe_rollback, _safe_close), processes_affected: 50` — every session close path; change must be additive (early return) and keep the existing wedged path byte-for-byte otherwise [graph, source-confirmed].
- Tests located: `tests/unit/test_http_client.py:327` (`test_build_ssl_context_preserves_default_roots_and_adds_certifi_bundle`, asserts the builder is called once → the builder must stay uncached), :74/:171/:224/:268 patch `_build_ssl_context` and assert `ssl_context_factory.call_count == 1` → a cache must be resettable per test [verified]. `tests/unit/test_db_session.py:1275` (wedged-rollback harness with `release_wedge` Event + interrupt spy) and :1597 (`test_reclaim_interrupts_the_real_aiosqlite_driver_without_a_spy`, passes an *already-finished* task and asserts invalidation) — the latter conflicts with the intended early return and must switch to a pending task [verified].
- OpenSpec: `openspec/specs/outbound-http-clients/spec.md:440` "Upstream connectors persist across interactive turn gaps" (keepalive/DNS TTL) is the neighbouring requirement; no requirement covers SSL-context construction or the SQLite teardown reclaim (`grep` over `openspec/specs`) [verified]. Contributing rule: behavior changes need `openspec/changes/<slug>/` [verified].
- PDG slice: not built (`freshness: accept`, no `--pdg` layer requested; the two functions are small and fully source-read) [inferred].

## Proposed Changes (§6)

1. `app/core/clients/http.py` — add `shared_ssl_context()` decorated with `functools.cache`, returning `_build_ssl_context()`; keep `_build_ssl_context` as the uncached builder. Switch `_build_http_client` to `shared_ssl_context()`. Rationale: one `SSLContext` per process is what aiohttp itself does for its default context; contexts are loop-independent and safe to share across connectors.
2. `app/core/clients/codex.py` — `create_codex_session` and `_socks_proxy_connector` import and use `shared_ssl_context` instead of `_build_ssl_context`.
3. `app/modules/settings/api.py` — `_probe_upstream_proxy_endpoint` uses `shared_ssl_context()`.
4. `app/db/session.py` — `_reclaim_wedged_sqlite_session`: first statement checks `abandoned.done()`; when done, retrieve its exception (if any, not cancelled) so nothing logs "never retrieved", log one WARNING `sqlite_teardown_bound_exceeded_but_finished phase=%s bound_seconds=%.1f elapsed_seconds=%.1f — the %s completed before the reclaim ran: the event loop was starved past the bound, no connection was held (issue #2029)`, and return without fencing, interrupting, invalidating, or registering cleanup tasks. `_safe_rollback`/`_safe_close` measure `elapsed = loop.time() - started` around `_shielded_bounded` and pass `elapsed_seconds=` (new keyword, default `None` so existing call sites and tests keep working); include `elapsed_seconds` in the existing `sqlite_wedged_teardown` line too.
5. `app/db/session.py` — in the `driver.interrupt()` except branch, classify a closed-connection failure (`ValueError` whose message contains `no active connection`, or `sqlite3.ProgrammingError` containing `closed database`) and log it as a one-line WARNING without traceback: `sqlite_wedged_teardown_interrupt_skipped phase=%s reason=connection_already_closed — ...`; everything else keeps the current `exc_info=True` warning. Invalidation still runs.
6. `openspec/changes/fix-sqlite-teardown-reclaim-false-positive/` — `proposal.md`, `tasks.md`, `specs/database-backends/spec.md` (ADDED requirement: bounded SQLite teardown reclaim MUST NOT fence/interrupt/invalidate a teardown that has completed by reclaim time and MUST log an attributable starvation line) and `specs/outbound-http-clients/spec.md` (ADDED requirement: one process-wide upstream SSL context; connectors MUST reuse it rather than loading the CA bundle per connector).

## Implementation Sequence (§7)

1. Add the OpenSpec change folder (proposal, tasks, two delta specs). Note: `openspec` CLI is not installed locally; validate via `npx -y @fission-ai/openspec@latest validate --specs` if available, else state it in the PR.
2. Tests first (red): add `test_shared_ssl_context_is_built_once_and_reused` + autouse fixture `shared_ssl_context.cache_clear()` in `tests/unit/test_http_client.py`; add `test_create_codex_session_uses_the_shared_ssl_context` (patch `app.core.clients.http.shared_ssl_context`, assert `TCPConnector(..., ssl=<shared>)`); in `tests/unit/test_db_session.py` add `test_close_session_does_not_reclaim_a_rollback_that_finished_before_the_reclaim_ran` (reuse the :1275 harness; wrap `_shielded_bounded` so that, when it returns the pending task, the wedge is released and the task awaited before returning — simulating a starved loop; assert interrupt spy not called, connection not invalidated, no wedge fence in `session.info`, the other writer succeeds, the new log line present) and `test_reclaim_logs_a_closed_connection_interrupt_without_a_traceback` (pending task + driver.interrupt raising `ValueError("no active connection")` → classified line, no `exc_info` line, connection invalidated). Run: `uv run pytest tests/unit/test_http_client.py tests/unit/test_db_session.py -q` → new tests fail.
3. Implement §6.1–6.3 (green for the SSL tests). Risk: any module importing `_build_ssl_context` directly keeps working (builder unchanged).
4. Implement §6.4–6.5; update `test_reclaim_interrupts_the_real_aiosqlite_driver_without_a_spy` to use an Event-gated pending task released after the reclaim returns. Risk (CRITICAL radius): the early return must happen before `session.info[...] = True` and before `_wedged_teardown_cleanup_tasks.add(abandoned)`; the existing wedged path below it is untouched.
5. Verify: `uv run pytest tests/unit/test_http_client.py tests/unit/test_db_session.py tests/unit/test_defer_cancellation_shield_leak.py tests/unit/test_proxy_websocket_client.py -q`; `make lint`; `make typecheck`; `uv run python scripts/check_proxy_architecture.py`.
6. Local product check on this Mac: `codex-lbctl` install from the branch build, watch `launchd-error.log` for `sqlite_teardown_bound_exceeded_but_finished` vs `sqlite_wedged_teardown`, and `sample` for `set_default_verify_paths` on the main thread (expected: absent after startup).

## Test Strategy (§8)

- Update: `tests/unit/test_db_session.py:1597` (pending task instead of finished task); `tests/unit/test_http_client.py` autouse cache reset.
- Add (owning seams): shared-context reuse across `_build_http_client` + `create_codex_session`; reclaim early-return under simulated loop starvation (asserts the writer slot was released by the normal rollback, i.e. the other writer's INSERT succeeds without invalidation); closed-connection interrupt classification.
- Regression sensitivity: the starvation test fails on current code because the reclaim invalidates the connection and sets the wedge fence; the reuse test fails because `_build_ssl_context` is called per connector.
- Commands: `uv run pytest tests/unit/test_http_client.py tests/unit/test_db_session.py -q`; `make lint`; `make typecheck`.

## Implementation Context (§11)

```yaml
implementation_context:
  task_summary: Cache the upstream SSLContext process-wide (shared_ssl_context) and make the SQLite wedged-teardown reclaim a logged no-op when the abandoned teardown already finished; classify closed-connection interrupt failures.
  acceptance_criteria:
    - No caller builds an SSLContext per connector; one context per process after startup.
    - A teardown that completes before the reclaim runs is not fenced, interrupted, or invalidated, and logs an attributable starvation line with elapsed seconds.
    - interrupt() on an already-closed aiosqlite connection logs one classified line without a traceback; invalidation still runs.
    - Existing wedged-worker behaviour (pending task at reclaim time) is unchanged.
  evidence_provenance: {
  "schema_version": 2,
  "head_commit": "665e58e316ef72d05ba791669879fa5c92746773",
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
      intended_change: use shared_ssl_context
    - file: app/modules/settings/api.py
      symbols: [_probe_upstream_proxy_endpoint]
      intended_change: use shared_ssl_context
    - file: app/db/session.py
      symbols: [_reclaim_wedged_sqlite_session, _safe_rollback, _safe_close]
      intended_change: early return when abandoned.done() with starvation log; elapsed_seconds attribution; classify closed-connection interrupt failures
    - file: openspec/changes/fix-sqlite-teardown-reclaim-false-positive/
      symbols: []
      intended_change: proposal, tasks, delta specs for database-backends and outbound-http-clients
  tests:
    - file: tests/unit/test_http_client.py
      scenarios:
        - "shared_ssl_context() twice → _build_ssl_context called once, same object returned"
        - "init_http_client after cache_clear → both TCPConnectors receive the shared context (existing assertions keep holding)"
        - "create_codex_session → TCPConnector ssl kwarg is the shared context"
    - file: tests/unit/test_db_session.py
      scenarios:
        - "rollback wedged past 0.2 s bound but finished before reclaim (starved loop simulated) → no interrupt, no invalidate, no fence, other writer INSERT succeeds, starvation WARNING logged"
        - "pending teardown + driver.interrupt raises ValueError('no active connection') → classified WARNING without traceback, connection invalidated"
        - "existing real-driver interrupt test with a pending task → still interrupts and invalidates"
  verification_commands:
    - uv run pytest tests/unit/test_http_client.py tests/unit/test_db_session.py tests/unit/test_defer_cancellation_shield_leak.py tests/unit/test_proxy_websocket_client.py -q
    - make lint
    - make typecheck
  assumptions:
    - "aiohttp accepts one SSLContext shared across TCPConnector/ProxyConnector instances — check: aiohttp does this for its own default context; verify by running the connector tests."
    - "functools.cache on a zero-arg function is thread-safe enough for the to_thread callers — check: only the loop thread calls it in production (grep shared_ssl_context callers after implementation)."
  open_questions:
    - "Whether upstream wants the elapsed_seconds attribution in the existing sqlite_wedged_teardown line or only in the new line (plan: both)."
    - "openspec CLI is not installed locally; validation may have to run in CI."
  avoid:
    - Do not repeat full repository discovery
    - Do not replace established patterns without evidence
    - Do not make _build_ssl_context itself cached (tests assert the builder is called once per call)
    - Do not inject a shared CodexClient across the 15 fallback sites in this PR (scope creep; follow-up)
    - Do not move _build_http_client off-loop in this PR (follow-up; startup-only after the cache)
```

## Assumptions and Open Questions (§12)

- [assumed] Sharing one `SSLContext` across connectors is safe (aiohttp shares its default context the same way); verified by tests + local run.
- Deferred follow-ups (not in this PR): inject the shared `CodexClient` from `app/modules/proxy` so `create_codex_session()` stops being the per-turn default; build replacement clients in `refresh_http_client*` off-loop; the `NullPool` last-connection WAL checkpoint churn on multi-GB stores.
- OpenSpec validation tool availability locally is unknown.

## Definition of Done (§13)

- New tests red on `main` @ 665e58e316ef, green on the branch; listed suites pass; `make lint` and `make typecheck` clean.
- OpenSpec change folder present with proposal, tasks, and two delta specs.
- PR opened against `Soju06/codex-lb` `main` with `Fixes #2029` (title `fix(db): ...` / conventional), template filled, test plan pasted.
- Local run on this Mac shows no `set_default_verify_paths` on the main thread after startup and no reclaim on teardowns that finished.
