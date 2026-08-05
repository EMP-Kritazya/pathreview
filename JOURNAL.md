## Week 7 — Issue selection

**Issue link:** https://github.com/ascherj/pathreview/issues/163

**Issue title:** Review creation does not verify profile ownership

**Tier:** [ ] Tier 1 [X] Tier 2 [ ] Tier 3

**Problem summary:**
The `POST /reviews` endpoint lets any logged-in user create a review for a profile just by passing its ID, without checking that the profile actually belongs to them. The bug lives in `create_review()` in `core/services/review_service.py`, which trusts the supplied `profile_id` even though the read functions (`get_review()`, `list_reviews()`) in the same file already filter by `Profile.user_id`. This is a broken-access-control (IDOR) vulnerability: knowing another user's profile UUID is enough to create reviews on their data. A successful fix adds an ownership check so that requests for a profile the user doesn't own are rejected (e.g. 403/404), and a regression test confirms the attack is blocked.

**Branch name:** fix/163-review-creation-profile-ownership-error

**Setup confirmation:** [X] App runs locally at localhost:5173

**Cohort ledger:** [X] Issue added to cohort ledger

**Is this right for me? checklist reasoning:**

- **Understanding:** I can explain it in my own words and I know the "done" state — before: a user can create a review on someone else's profile by passing its ID; after: that request is rejected (403/404) and only the owner can create a review. Before/after is concrete and testable.

- **Tier fit:** Tier 2 is a stretch for a first contribution, but the scope is narrow — the fix is confined to `create_review()` in `core/services review_service.py`, and the exact pattern I need (`Profile.user_id == user_id`) already exists in `get_review()` and `list_reviews()` in the same file. So it reads like a Tier 1-sized change with Tier 2 cross-module context (auth → profile → review). Comfortable, not over my head.

- **Codebase readiness:** I've read the three service functions and confirmed the bug: `create_review()` takes `user_id` but never filters by it. The test file `tests/unit/test_review_service.py` exists, so I have existing tests to model my regression test on.

- **Scope & time:** Small, well-bounded change plus one test — realistic within the Week 8–9 window alongside other commitments. No blockers or dependencies listed on the issue.

## Week 8 — Reproduction & solution planning

**Reproduction commit link:** <!-- TODO: paste the GitHub link to this Week 8 commit after pushing, e.g. https://github.com/EMP-Kritazya/pathreview/commit/<sha> -->

**Reproduction summary:**
I reproduced issue #163 by driving `create_review()` (in `core/services/review_service.py`)
directly with a mismatched pair — an attacker `user_id` and a victim `profile_id` the
attacker does not own — while tracking whether the service performs any ownership lookup.
Observed: `create_review()` never issued an ownership query (`db.execute` was never called),
called `db.add`, and returned a persisted `pending` review anyway. This confirms the IDOR:
any authenticated user can create a review against another user's profile just by knowing
its UUID, because `create_review()` accepts `user_id` but ignores it. The read paths
(`get_review`, `list_reviews`) already scope by `Profile.user_id`, so only the create path
is unprotected.

**Reproduction steps:**

1. From the fork root (`pathreview/`), with the app's virtualenv, call `create_review(db, profile_id, user_id)` with `profile_id` and `user_id` that do not correspond to the same profile (mocked DB session, patched `Review`).
2. Assert whether any ownership `SELECT` is issued before the write.
3. Observed result: no ownership check, `db.add` called, a review returned — i.e. the review is created for a profile the caller does not own.

**PLAN.md link:** [PLAN.md](./PLAN.md) <!-- on GitHub: https://github.com/EMP-Kritazya/pathreview/blob/fix/163-review-creation-profile-ownership-error/PLAN.md -->

**Walkthrough video (recommended):** [link to your Loom video, ≤2 min — recommended, not graded]

**Blockers or open questions:**
Deciding between `404` (matches `get_review`/`get_profile`, avoids leaking profile existence)
and `403` for the rejected case — leaning `404` for consistency. Also whether to fix the
pre-existing `AsyncMock().scalars()` failures in `tests/unit/test_review_service.py` as part
of this PR or keep them out of scope (leaning out of scope).

## Week 9 — Solution building & PR submission

### Check-in 1 (mid-week, 2026-08-04)

**Current progress:**
All 5 steps of PLAN.md are implemented and locally verified:

1. `create_review()` in `core/services/review_service.py` now calls
   `profile_service.get_profile(db, profile_id, user_id)` before constructing a
   `Review`, and returns `None` when the profile is missing or not owned.
2. `create_review_endpoint()` in `api/routes/reviews.py` checks for that `None`
   and raises `HTTPException(404, "Profile not found")` before queuing the
   `process_review` background task — matching the `404` decision noted as an
   open question in the Week 8 entry.
3. Added the regression test `test_create_review_returns_none_for_unauthorized_profile`
   in `tests/unit/test_review_service.py`, plus updated the other `create_review`
   tests to mock `get_profile` (they now patch it to return an owned profile so the
   happy path still exercises `Review` construction). Also replaced the `AsyncMock().scalars()`
   pattern with plain `Mock()` for `.scalars()` across the file, which resolves the
   pre-existing mock-quirk failures flagged in Week 8 — so that "leaning out of scope"
   call ended up moot, since it was needed to make the new/adjacent tests reliable.
4. Verified the happy path manually against the running API (docker-compose postgres
   - uvicorn): registered two users, created a profile for each, confirmed
     `POST /reviews` with another user's `profile_id` returns `404` and creates nothing,
     and `POST /reviews` with the caller's own `profile_id` returns `200` and the
     background task completes the review end-to-end.
5. `make test-unit`: 20/20 tests pass in `tests/unit/test_review_service.py`; full
   suite is 389 passed / 40 failed, and I confirmed those 40 failures are pre-existing
   and unrelated (bias*detector, pii_scrubber, tech_detector, etc. — none touch
   review/profile code) by running the same suite before my changes (53 failed / 375
   passed at baseline — my change actually \_fixes* 13 of those, the ones caused by the
   mock-quirk in `test_review_service.py`). `make check` passes with no new lint/type
   errors introduced; two pre-existing mypy/type gaps in files I touched
   (`User.id`/`Review.id` stored as `str` vs. the `UUID` params services expect, and
   `Review.sections` typed as `dict | None` while `process_review` stores a list) were
   fixed or explicitly annotated as pre-existing so the pre-commit hook's mypy check
   passes without silently masking unrelated bugs.

**Next steps:**
Push the branch, open the PR against `ascherj/pathreview`, and fill in the PR
template (including the pre-existing-failures note above). Also want to re-read
`docs/CONTRIBUTING.md` once more for docstring conventions before opening the PR.

**Blockers:**
None — the only snag was the local pre-commit hook (ruff/mypy) failing on
pre-existing issues in the files I touched; resolved with minimal, behavior-preserving
type annotations rather than skipping the hook.

---

### Check-in 2 (end of week)

**PR link:** https://github.com/ascherj/pathreview/pull/912

**Branch:** `fix/163-review-creation-profile-ownership-error`

**What you built:**
`create_review()` in `core/services/review_service.py` now checks profile ownership
before creating a review, by reusing the existing `profile_service.get_profile(db,
profile_id, user_id)` lookup that already scopes by `Profile.user_id`. If the
profile doesn't exist or isn't owned by the caller, `create_review()` returns
`None` and `create_review_endpoint()` in `api/routes/reviews.py` responds `404`
before any background processing is queued — closing the IDOR in issue #163
without touching the read paths, which were already safe.

**Tests added or updated:**
`tests/unit/test_review_service.py` — added
`test_create_review_returns_none_for_unauthorized_profile` (asserts `create_review`
returns `None` and never calls `db.add`/`commit`/`refresh` for a profile the caller
doesn't own). Updated the other `create_review` tests to mock `get_profile` so the
happy path still exercises `Review` construction, and replaced the
`AsyncMock().scalars()` pattern with plain `Mock()` for `.scalars()` throughout the
file, fixing several pre-existing mock-quirk failures unrelated to #163 along the
way.

**Self-review confirmation:** [x] make check passes [x] make test-unit passes
(40 pre-existing, unrelated failures documented in the PR description — confirmed
against a pre-change baseline of 53 failed/375 passed; this branch is 40 failed/389
passed, so no new failures and 13 fewer than baseline)

**Draft PR feedback received from:** none yet
