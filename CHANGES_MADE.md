# CHANGES_MADE.md — AURA Prime Windows Setup Fixes

> All changes are Windows-only compatibility fixes and stale-comment removal.
> No production logic, risk, execution, accounting, or live-trading code was touched.
> No test was skipped, weakened, or xfail'd. No threshold was lowered.

---

## Fix 1 — Remove stale `# noqa` comments in test_telegram_alerts.py

**File:** `tests/test_telegram_alerts.py`

**Lines changed:** 188, 197

**Why:** Ruff rule `RUF100` flags `noqa` directives for rules that are not enabled in the
project's ruff config (`pyproject.toml` has `[tool.ruff]` with no `extend-select`).
Rule `DTZ001` is not enabled, so these two comments were stale leftovers from an
earlier development phase. Ruff blocked the Setup verification step.

**Before:**
```python
alert(created_at=datetime(2026, 1, 1))  # noqa: DTZ001 - invalid input under test
completed_at=datetime(2026, 1, 1),  # noqa: DTZ001 - invalid input under test
```

**After:**
```python
alert(created_at=datetime(2026, 1, 1))
completed_at=datetime(2026, 1, 1),
```

**Before test result:** Setup blocked — `ruff check` exited non-zero (RUF100 ×2)
**After test result:** `ruff check aura tests examples` → `All checks passed!`

---

## Fix 2 — Windows WinError 10053 retry in test_web_command_center_v2.py

**File:** `tests/test_web_command_center_v2.py`

**Lines changed:** 25–46 (`_request_json` function)

**Why:** `ConnectionAbortedError: [WinError 10053] WSAECONNABORTED` is a Windows-only
TCP race condition. When `ThreadingHTTPServer` starts in a daemon thread, Windows may
abort the first connection before the server's accept loop is ready. The test already
handled this in the `_service()` warmup loop (lines ~66-84) for GET requests but not
for the POST requests made via `_request_json`. Added a 5-attempt retry (×50ms sleep)
for `ConnectionAbortedError` / `ConnectionResetError` only — `HTTPError` is still
caught on the first attempt so all status-code assertions (401, 422, 202) are unaffected.
This pattern is identical to the existing `_service()` warmup loop.

**Before test result:**
```
FAILED tests/test_web_command_center_v2.py::test_v2_http_owner_token_and_session_only_pwa
ConnectionAbortedError: [WinError 10053]
```
**After test result:** `1 passed in 0.99s` ✅

---

## Fix 3 — Windows WinError 10053 retry + server warmup in test_web_command_center.py

**File:** `tests/test_web_command_center.py`

**Lines changed:** 22–49 (`_request_json`), 58–69 (`_running_service`)

**Why:** Same root cause as Fix 2, but the V1 test file had two vulnerabilities:
1. `_request_json` had no retry (same fix as Fix 2 applied here)
2. `_running_service` had **no warmup loop at all** — it called `thread.start()` and
   returned immediately. The V2 `_service()` already had a 50-retry warmup loop that
   this V1 helper was missing entirely. Added identical warmup pattern (50 × 10ms).

**Before test result:**
```
FAILED tests/test_web_command_center.py::test_http_rejects_missing_and_wrong_owner_token
ConnectionAbortedError: [WinError 10053]
```
**After test result:** `11 passed in 2.61s` ✅

---

## Fix 4 — Regenerate stale audit snapshot artifacts

**Files updated (auto-generated, not manually edited):**
- `artifacts/governance/repo_audit.json`
- `artifacts/governance/test_inventory.json`
- `artifacts/governance/module_map.md`

**Command run:** `.venv\Scripts\python.exe -m aura.ops.refresh_software_evidence`

**Why:** After Fixes 1–3 changed 3 source files, the pre-generated audit snapshots
(which contain per-file checksums) became stale. The test
`test_generated_phase_zero_artifacts_are_current` checks these hashes and failed.
Running the project's own refresh command regenerated them. No manual editing.

**Before test result:**
```
FAILED tests/test_repository_audit.py::test_generated_phase_zero_artifacts_are_current
AssertionError: stale generated audit artifact: artifacts\governance\repo_audit.json
```
**After test result:** All audit tests pass ✅

---

## Summary

| # | File | Type | Before | After |
|---|------|------|--------|-------|
| 1 | `tests/test_telegram_alerts.py` | Stale comment removal | ruff FAIL | ruff PASS |
| 2 | `tests/test_web_command_center_v2.py` | Windows retry loop | 1 FAILED | PASS |
| 3 | `tests/test_web_command_center.py` | Windows retry + warmup | 1 FAILED | 11 PASS |
| 4 | `artifacts/governance/*.json/.md` | Audit snapshot refresh | stale | current |

**No secrets in this file. No production code modified.**
