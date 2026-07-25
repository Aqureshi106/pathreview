# Contribution Journal

## Week 7 — Issue selection

**Issue link:** [https://github.com/ascherj/pathreview/issues/158](https://github.com/ascherj/pathreview/issues/158)

**Issue title:** review_service unit tests misconfigure async mocks — 13 of 19 tests fail

**Tier:** [x] Tier 1  [ ] Tier 2  [ ] Tier 3

**Problem summary:**
The unit tests for `core/services/review_service.py` build their fake database result as
`AsyncMock()`, which means every attribute pulled off of it — including `.scalars()` — is
also automatically treated as async. Since `.scalars()` is actually a synchronous call in
SQLAlchemy's async API, calling it on the mock returns a coroutine object instead of a
result object, and that coroutine has no `.first()` or `.all()` method, so 13 of the 19
tests in `tests/unit/test_review_service.py` fail with `AttributeError`. The service code
itself in `core/services/review_service.py` is correct and does not need to change — only
the test mocks do. A successful fix rebuilds the mocked result object as a plain `Mock`/
`MagicMock` (keeping `AsyncMock` only on the awaited `db.execute` call) so all 19 tests pass.

**Selection notes ("Is this right for me?" checklist reasoning):**

*Part 1 — Understanding the issue:* Confirmed I could restate the bug without looking at the
issue body: the tests fake the DB result with `AsyncMock()`, so `.scalars()` on it returns a
coroutine instead of a plain object, breaking `.first()`/`.all()`. Verified this is real by
running `pytest tests/unit/test_review_service.py -v -m unit` myself — got 13 failed, 6 passed,
matching the issue exactly. Confirmed the affected file (`tests/unit/test_review_service.py`)
exists and read the whole thing, plus the service module it tests
(`core/services/review_service.py`), so I know the fix is test-only.

*Part 2 — Tier fit:* This is my first time working in a codebase this size, so I deliberately
stayed in Tier 1 rather than reaching for a Tier 2/3 issue to "challenge myself." I also looked
at issue #119 (Tier 2, docstrings across 3 files in `core/services/`) as a comparison — it's a
reasonable issue but a bigger, more diffuse scope (4–6 hrs, subjective "done" criteria across
multiple files) than #158's single-file, objectively-verifiable fix (tests pass or they don't).

*Part 3 — Codebase readiness:* Read `create_review`, `get_review`, and `list_reviews` in
`review_service.py` along with every test in `test_review_service.py`, including the shared
`mock_db_session` fixture and how individual tests override it. I can already sketch the fix:
replace each test's `mock_result = AsyncMock()` with a plain `Mock()`/`MagicMock()`, keeping
`AsyncMock` only where `db.execute` itself is awaited.

*Part 4 — Scope and time:* Checked the issue's comments and the cohort ledger before claiming —
#158 had only 2 prior claims, versus 5–11 on most other open Tier-1 issues (e.g. #146, #147,
#154, #155), so it's comparatively uncrowded. No `blocked by` references in the issue. Estimated
2–3 hours given the fix is a mechanical, repeated pattern across ~13 tests — comfortably within
the Week 8–9 window alongside my other coursework.

**Branch name:** fix/158-review-service-async-mocks

**Setup confirmation:** [x] App runs locally at localhost:5173

**Cohort ledger:** [x] Issue added to cohort ledger

## Week 8 — Reproduction & solution planning

**Reproduction commit link:** [9cc2587](https://github.com/Aqureshi106/pathreview/commit/9cc25872e8c4bf9d0e0dc39c794790a51942d66a)

**Reproduction summary:**
Ran `python -m pytest tests/unit/test_review_service.py -v -m unit` and got **13 failed, 6
passed**, matching the issue exactly, with both failure signatures present:
`AttributeError: 'coroutine' object has no attribute 'first'` (from `get_review`, at
`core/services/review_service.py:47`) and `'...has no attribute 'all'` (from `list_reviews`, at
line 65). Root cause: every failing test mocks the `db.execute()` return value as
`mock_result = AsyncMock()`, which makes `.scalars()` itself resolve to an async attribute and
return a coroutine instead of a plain result object — `core/services/review_service.py` is
correct and doesn't need to change.

**PLAN.md link:** [PLAN.md](https://github.com/Aqureshi106/pathreview/blob/fix/158-review-service-async-mocks/PLAN.md)

**Walkthrough video (recommended):** Not recorded yet.

**Blockers or open questions:**
None blocking. Open question for Week 9: whether to use `Mock()` or `MagicMock()` as the
replacement — leaning `MagicMock` for consistency with other fixtures in the file, but will
confirm against `docs/CONTRIBUTING.md` conventions before implementing (see PLAN.md Risks &
Unknowns #1).

<details>
<summary>Detailed reproduction steps (expand)</summary>

1. From the repo root, ran:
   ```
   python -m pytest tests/unit/test_review_service.py -v -m unit
   ```
2. Result: **13 failed, 6 passed** — matches the issue report and my Week 7 restatement exactly.
3. Confirmed both failure signatures named in the issue:
   - `get_review` tests fail with `AttributeError: 'coroutine' object has no attribute 'first'`
     at `core/services/review_service.py:47` (`result.scalars().first()`).
   - `list_reviews` tests fail with `AttributeError: 'coroutine' object has no attribute 'all'`
     at `core/services/review_service.py:65` (`count_result.scalars().all()`).
4. Root cause confirmed by reading `tests/unit/test_review_service.py`: every failing test builds
   `mock_result = AsyncMock()`. Because `AsyncMock` treats *every* attribute access as async by
   default, `mock_result.scalars` resolves to an `AsyncMock` too, so calling `.scalars()` returns
   a coroutine instead of a plain result object — and that coroutine has no `.first()`/`.all()`.
   The 6 passing tests (`test_create_review_*`, `test_review_sections_and_score_initially_none`)
   never touch `.scalars()`, which is why they're unaffected.
5. Confirmed `core/services/review_service.py` itself is correct and needs no change — `db.execute`
   is genuinely awaited (real SQLAlchemy async sessions), but `result.scalars()` is a synchronous
   call on the result object. The bug is entirely in how the tests fake that result.

No source files were modified to reproduce the bug — it reproduces on the branch as-is.

</details>
