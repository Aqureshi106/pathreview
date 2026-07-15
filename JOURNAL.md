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

**Branch name:** fix/158-review-service-async-mocks

**Setup confirmation:** [x] App runs locally at localhost:5173

**Cohort ledger:** [ ] Issue added to cohort ledger
