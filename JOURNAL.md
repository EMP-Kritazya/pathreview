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
