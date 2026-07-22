## Solution plan

**Issue:** [#158](https://github.com/ascherj/pathreview/issues/158) — `review_service` unit tests
misconfigure async mocks — 13 of 19 tests fail

### Understand

`tests/unit/test_review_service.py` fakes the object returned by `await db.execute(stmt)` as
`mock_result = AsyncMock()`. `AsyncMock` makes *every* attribute access on it — not just calls —
resolve to another `AsyncMock`, because it doesn't know in advance which attributes will be
awaited and which won't. So `mock_result.scalars` is itself an `AsyncMock`, and calling
`mock_result.scalars()` returns a coroutine object rather than a plain `Result`-like object.
That coroutine has no `.first()` or `.all()` method, so any test that calls
`mock_result.scalars.return_value.first.return_value = ...` or
`mock_result.scalars.return_value.all.return_value = ...` is setting up a return value that the
production code never actually reaches — `result.scalars()` blows up with `AttributeError`
before it gets there.

In real SQLAlchemy async usage, only `db.execute(stmt)` is a coroutine that must be awaited.
The `Result` object it returns is a plain, synchronous object — `.scalars()`, `.first()`, and
`.all()` are all synchronous calls on it. So the mock needs exactly one async boundary
(`db.execute`), not two.

Verified via reproduction (see JOURNAL.md Week 8 entry): running
`pytest tests/unit/test_review_service.py -v -m unit` gives 13 failed / 6 passed, with the two
distinct signatures `'coroutine' object has no attribute 'first'` (from `get_review`, which calls
`result.scalars().first()` at `core/services/review_service.py:47`) and `'coroutine' object has
no attribute 'all'` (from `list_reviews`, which calls `.scalars().all()` twice, at lines 65 and
77). The 6 tests that pass (`test_create_review_*` and
`test_review_sections_and_score_initially_none`) never call `.scalars()` on a mocked result —
they only assert against `db.add`/`db.commit`/`db.refresh` or the `Review(...)` constructor call,
which is why they're unaffected.

**Root cause:** In `tests/unit/test_review_service.py`, every test that mocks the return value of
`db.execute()` wraps it in `AsyncMock()` instead of `Mock()`/`MagicMock()`. `core/services/
review_service.py` is correct and will not be touched.

### Map

Files I expect to touch:

- `tests/unit/test_review_service.py` — the only file that changes. Specifically these 13 test
  methods, each of which currently does `mock_result = AsyncMock()`:
  - `test_get_review_returns_review_for_correct_owner` (line 81)
  - `test_get_review_returns_none_for_wrong_user` (line 98)
  - `test_list_reviews_returns_paginated_results` (line 115)
  - `test_list_reviews_page_2_returns_correct_offset` (line 132)
  - `test_list_reviews_returns_tuple` (line 150)
  - `test_get_review_uses_select_and_join` (line 201)
  - `test_list_reviews_default_pagination` (line 215)
  - `test_list_reviews_custom_page_size` (line 231)
  - `test_get_review_verifies_ownership` (line 261)
  - `test_list_reviews_counts_total` (line 276)
  - `test_list_reviews_returns_reviews_list` (line 291)
  - `test_get_review_with_valid_uuid` (line 319)
  - `test_list_reviews_ordered_by_created_at` (line 333)

Files I looked at but will **not** touch:

- `core/services/review_service.py` — `get_review()` (line 35) and `list_reviews()` (line 50) are
  correct as-is; `db.execute` is genuinely the only awaited call.
- `tests/unit/test_review_service.py`'s `mock_db_session` fixture (line 19) — this one is already
  correct: `session.execute = AsyncMock()` is right because `db.execute()` really is awaited. Only
  the *result* objects returned by execute are mismocked, not the session itself.

### Plan

1. Run `make test-unit` (or the scoped `pytest tests/unit/test_review_service.py -v -m unit`)
   before making any change, to have a clean "before" baseline (13 failed, 6 passed — already
   captured in JOURNAL.md).
2. For each of the 13 affected tests, change `mock_result = AsyncMock()` to
   `mock_result = MagicMock()`. `MagicMock` is used instead of plain `Mock` so that any incidental
   magic-method access (e.g. `__len__` if ever needed) keeps working, but the key behavior change
   is that `mock_result.scalars()` now returns a plain, non-awaitable return value by default.
3. Leave `mock_result.scalars.return_value.first.return_value = ...` and
   `mock_result.scalars.return_value.all.return_value = ...` lines unchanged — they already
   express the right chain (`.scalars().first()` / `.scalars().all()`); they just need
   `mock_result` itself to be a synchronous mock for the chain to resolve correctly.
4. Leave `mock_db_session.execute = AsyncMock(return_value=mock_result)` unchanged in every test —
   `db.execute` itself must stay async since it's genuinely awaited in the service.
5. Re-run `pytest tests/unit/test_review_service.py -v -m unit` and confirm all 19 tests pass.
6. Run the full unit suite (`make test-unit`) to confirm no other test file relies on the old
   (broken) mock shape or is otherwise affected.
7. Run `make check` (lint + format + type check) to confirm the diff is clean — this change is
   pure test-code, so I don't expect any typing/lint issues, but I'll confirm rather than assume.
8. Re-read the diff once more against the "Map" section above to confirm no test assertions
   changed in meaning — only the mock construction changed, not what's being verified.

### Inputs & outputs

**What's under test:** `get_review(db, review_id, user_id) -> Review | None` and
`list_reviews(db, user_id, page=1, page_size=20) -> tuple[list[Review], int]` in
`core/services/review_service.py` — unchanged by this fix.

**Existing (broken) mock shape, e.g. in `test_get_review_returns_review_for_correct_owner`:**

```python
mock_result = AsyncMock()
mock_result.scalars.return_value.first.return_value = mock_review
mock_db_session.execute = AsyncMock(return_value=mock_result)
```

**Fixed mock shape:**

```python
mock_result = MagicMock()
mock_result.scalars.return_value.first.return_value = mock_review
mock_db_session.execute = AsyncMock(return_value=mock_result)
```

**Expected outcome:** `pytest tests/unit/test_review_service.py -v -m unit` reports
`19 passed`, with no changes to `core/services/review_service.py` and no changes to what each
test asserts — only how the mocked `db.execute()` return value is constructed.

I'm not adding a new test for this issue: the fix doesn't change the service's behavior, it
corrects mocks so the 13 existing tests actually exercise the assertions they already contain.
The verification is that all 19 pre-existing tests pass, not a new test case.

### Risks & unknowns

1. **`Mock` vs `MagicMock`.** Plain `Mock()` would also fix the `.scalars()` issue, since the bug
   is specifically about `AsyncMock`'s auto-async attribute behavior, not about magic methods.
   I'm using `MagicMock` for consistency with the `mock_review`/`mock_profile` fixtures elsewhere
   in the file, which already use `Mock()` for leaf objects — I'll double check during
   implementation whether the codebase's convention (per `docs/CONTRIBUTING.md` and other test
   files in `tests/unit/`) prefers one over the other, and match it.
2. **Other test files might have the same pattern.** Since this is a copy-paste-style mistake, it
   seemed possible other files in `tests/unit/` mock a `db.execute()` result the same broken way.
   Checked with `grep -rn "mock_result = AsyncMock()" tests/unit/` — the only match is
   `tests/unit/test_review_service.py`, so this fix is confirmed self-contained; no other test
   file needs the same change.
3. **`test_list_reviews_page_2_returns_correct_offset` currently passes despite being logically
   thin** — it only asserts `len(calls) > 0`, not that the offset was actually computed correctly.
   Fixing the mock might be a good moment to ask whether this test should be strengthened, but
   I'll treat that as out of scope for #158 unless asked, since the issue is specifically about
   the mocks breaking, not about weak assertions.

### Edge cases

- Tests where `.scalars().first()` should resolve to `None` (`test_get_review_returns_none_for_wrong_user`,
  several `get_review` tests with `first.return_value = None`) — must continue to resolve to
  `None`, not a `Mock`, after the fix.
- Tests where `.scalars().all()` should resolve to an empty list (`test_list_reviews_*` with
  `all.return_value = []`) — must continue to resolve to `[]`, not a `Mock`.
- `test_list_reviews_counts_total` and `test_list_reviews_returns_paginated_results` both mock
  `all.return_value = mock_reviews` (a list of 5 `Mock()` objects) — need to confirm `len()` on
  that list still works correctly once `mock_result` is no longer an `AsyncMock` (it will, since
  `all.return_value` is just a plain Python list already; the fix only changes how `mock_result`
  itself behaves, not the configured return values).
