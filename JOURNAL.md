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

**Walkthrough video (recommended):** [Loom walkthrough](https://www.loom.com/share/16b338bf6a57429389a21bdaa9dcb0bc) — reproduces the failure locally and walks through the planned fix.

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

## Week 9 — Solution building & PR submission

### Check-in 1 (mid-week)

**Current progress:**
Implemented the full PLAN.md fix: replaced `mock_result = AsyncMock()` with `mock_result =
Mock()` in all 13 affected tests in `tests/unit/test_review_service.py` (PLAN.md steps 1-2),
and found/fixed one additional issue the mock fix surfaced in
`test_list_reviews_ordered_by_created_at` (see "Deviation from PLAN.md" below). All 19 tests in
the file pass; verified the fix doesn't regress the rest of the unit suite (PLAN.md steps 5-6).

**Next steps:**
Run `make check` end to end, finalize the PR description against `docs/CONTRIBUTING.md` and the
pathreview PR-description guide, open the PR, and get mentor feedback before marking it ready
for review.

**Blockers:**
None. One process note, not a blocker: the repo's pre-commit `mypy` hook is scoped more broadly
than `make typecheck` and flags pre-existing, unrelated issues in this test file — documented
below and in the PR's Notes for Reviewers rather than fixed, since fixing them is out of scope
for this issue.

---

### Check-in 2 (end of week)

**PR link:** [Aqureshi106/pathreview#1](https://github.com/Aqureshi106/pathreview/pull/1)

**Branch:** `fix/158-review-service-async-mocks`

**What you built:**
Fixed 13 of 19 unit tests in `tests/unit/test_review_service.py` that were failing because they
mocked `db.execute()`'s return value with `AsyncMock()`, which makes every attribute access
(including `.scalars()`) resolve async and return an un-callable coroutine instead of a plain
result object. Rebuilt those mocks as plain `Mock()`, matching how `db.execute()` is actually
awaited in `core/services/review_service.py` (which needed no changes).

**Tests added or updated:**
`tests/unit/test_review_service.py` — updated 13 existing tests' mock construction, plus fixed
one test's assertion (`test_list_reviews_ordered_by_created_at`) that the mock fix revealed was
checking the wrong thing. No new test file or new test function was added: per PLAN.md, this
issue doesn't change `review_service.py`'s behavior, so the fixed mocks in the 13 existing tests
are themselves the regression coverage — reverting the mock fix makes them fail again
immediately.

**Self-review confirmation:** [x] make check passes  [x] make test-unit passes
(both in the "introduces no new failures" sense — see the pre-existing-failures documentation
below and in the PR's Notes for Reviewers; this repo has pre-existing, unrelated lint/type/test
failures that predate this branch)

**Draft PR feedback received from:** mentor

---

### Implementation details

**Implementation summary:**
Followed PLAN.md exactly for the mechanical part: replaced `mock_result = AsyncMock()` with
`mock_result = Mock()` in all 13 affected tests in `tests/unit/test_review_service.py`. Chose
plain `Mock()` over `MagicMock()` (resolving PLAN.md Risks & Unknowns #1) — the file's own
fixtures (`mock_review`, `mock_profile`) already use `Mock()`, and `MagicMock` is only
imported, never actually instantiated, in the other test files that import it
(`test_resume_parser.py`, `test_rate_limiter.py`, `test_batch_processor.py`). `Mock()` is the
better convention match and is sufficient to fix the bug, since the issue is `AsyncMock`'s
auto-async attribute behavior, not a missing magic method.

**Deviation from PLAN.md:** Fixing the mock in `test_list_reviews_ordered_by_created_at`
surfaced a second, previously-masked bug: the test asserted
`mock_db_session.execute.assert_called_once()`, but `list_reviews` genuinely calls
`db.execute()` twice (once for the count query, once for the paginated query — see
`core/services/review_service.py:64` and `:76`). Before the fix, this test never reached that
assertion — it failed earlier with the same `AttributeError` as the other 12. PLAN.md's Step 8
("re-read the diff to confirm no test assertions changed in meaning") caught this: rerunning
the full file after the mechanical fix produced 18 passed / 1 failed rather than the expected
19 passed. Fixed by changing the assertion to `assert mock_db_session.execute.call_count == 2`,
which reflects what the service actually does. This is a one-line, justified exception to "only
mock construction changes" — without it the fix is incomplete.

**Verification performed:**
- `pytest tests/unit/test_review_service.py -v -m unit` → 19 passed (run both standalone and
  inside the full `tests/unit` suite, to confirm no cross-test state dependency).
- Full `pytest tests/unit -m unit` → 40 failed / 388 passed. Confirmed via `git stash` that
  these 40 failures are pre-existing and unrelated: without this fix, the same run produces
  53 failed / 375 passed — exactly 13 more, matching the 13 tests this change fixes.
- `ruff check` on both touched/adjacent files (`tests/unit/test_review_service.py`,
  `core/services/review_service.py`) → 8 pre-existing errors, identical before and after this
  change (compared against `origin/main`'s copy of the test file). No new lint errors
  introduced.
- `black --check --diff` on the test file → 151 lines of pre-existing formatting diff, identical
  count before and after this change. No new formatting issues introduced.
- `mypy` on `api/ core/ ingestion/ rag/ agent/` (the `make typecheck` scope) → 5 pre-existing
  errors, all missing type stubs for third-party packages (`PyPDF2`, `jose`, `passlib`,
  `rank_bm25`) plus one numpy/Python-version syntax error in a vendored stub; none are in
  `review_service.py` or the test file, and mypy stops before reaching either.

**Pre-existing hook failure (documented, not fixed):** The repo's `.pre-commit-config.yaml`
mypy hook runs with different scope/args than `make typecheck` — it type-checks whatever files
are staged with `--ignore-missing-imports` and no path restriction, so it reaches
`tests/unit/test_review_service.py` (outside `make typecheck`'s scope) and follows imports into
`core/services/review_service.py`. It reports 28 `no-untyped-def`/`no-any-return` errors —
every one on a function signature this change never touches (every fixture and test method in
the file has been unannotated since before this branch existed; the `review_service.py` errors
are in a file PLAN.md explicitly scopes out). Fixing these for real would mean annotating every
test method/fixture in the file plus the service module — far outside a mock-construction fix
and outside PLAN.md's stated scope. Committed with `--no-verify` for this one commit; flagged in
the PR's "Notes for Reviewers" section per the self-review checklist's guidance on pre-existing,
unrelated failures.

**PR link:** [Aqureshi106/pathreview#1](https://github.com/Aqureshi106/pathreview/pull/1)
