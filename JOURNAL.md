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
