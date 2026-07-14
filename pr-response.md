# PR Response Doc — CineLog Watchlist Feature

![Git Log](image.png)
## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

I used Claude Code (AI assistant) at several points during this project:

1. **Codebase orientation / understanding a pattern.** Asked it to compare `get_collection()` in `collection_service.py` against `get_watchlist()` in `watchlist_service.py` and explain when the `.join(Film)` in `get_watchlist()` is actually required. It walked through the query and pointed out that the join wasn't doing any filtering/sorting work in this codebase's usage (neither `order_by` nor `filter_by` references a `Film` column) — its only real effect is turning the query into an INNER JOIN, which would silently drop orphaned `WatchlistEntry` rows if a film were ever deleted without cascading. That also surfaced an inconsistency: `get_collection()` has no equivalent guard, so it isn't protected the same way. This was purely codebase orientation — I didn't take further action on the inconsistency itself, just used the explanation to understand the existing pattern.

2. **Verifying commit format.** Asked it to check my `git log --oneline` history against Conventional Commits conventions and flag any commits bundling multiple logical changes. It flagged: a typo in one message, `.gitignore` categorized as `docs` instead of `chore`, several commits referencing PR review-comment numbers (`"complete comment #2"`) instead of describing the actual change, and one commit with no blank line between subject and body that bundled three unrelated changes (model+endpoint, an unnamed bug fix, and unnamed "more changes") into a single commit. I rebased to reword/split most of these into atomic, content-focused commits; the one bundled commit was left as-is since it was authored by another contributor, not me.

3. **Drafting the PR description and test plan.** Asked it to draft the "PR Description" section below (overview, the two design decisions, manual testing steps) and to add unit tests. While writing the manual testing steps, it caught a real gap: `routes/watchlist/watchlist.py` never caught `FilmNotFoundError`/`AlreadyInWatchlistError`, unlike the equivalent routes in `collection.py` — meaning the documented `404`/`409` test steps would have actually returned an unhandled `500`. I had it fix the route to catch those exceptions properly, then add `tests/test_watchlist_routes.py` to cover the route-level status codes end-to-end, on top of the existing service-level tests.

Comments 4 and 5 (default visibility and sort order) reflect my own reasoning, not AI drafting — I didn't use AI to generate or stress-test those arguments before writing them.

## Comment 1 — Rename
**What I did:** use the find all references feature on VSCode to identify all usages/calls to `save_to_watchlist()` and rename each references.
**How I verified:** after all refactoring, I did a global search on `save_to_watchlist` and ensure that there is no match in this repo.

## Comment 2 — Deduplication
**What I did:** add an additional query/check on the user's WatchlistEntry. If a valid data returns from the query, then this means the user has already added the same file to Watchlist, so throw an `AlreadyInWatchlistError` (newly added Exception to surface this type of error).
**How I verified:** add an unit test in the test suite to ensure the same file cannot be added to the same watchlist and the expected error is thrown.

## Comment 3 — Missing test
**What I did:** Confirm test for watchlist_service is missing in the test suite entirely, added a test class for `watchlist_service` and added the missing test case for film not found.
**How I verified:** ran the entire test suite using `pytest /tests -v` to ensure all tests are passing and there is an entry for the newly added test.

## Comment 4 — Default visibility
**My position:** Default to `public=True`
**Reasoning:** this enables public view of user's watchlist, which offer a few benefits:
1. increasing user-to-user connection/engagement via shared films in the watchlist
2. targeted marketing on specific films
3. increase user's interest on a film after knowing number of other user's having a similar interests
**Tradeoff acknowledged:** Exposal of user's film preferences and privacy, and platform targeted marketing might be aggressive for users and cause negative reactions among users.

## Comment 5 — Sort order
**My position:** Sort based on added timestamp in descending order.
**Reasoning:** The latest added films are the closest representation of user's interest and film preferences, and they will most likely want to watch those films.
**Engagement with reviewer's point:** Agree with this suggestion. If sorted by title, then this mixed various added films. By argument, user's taste might have changed since they added the first few films to the watchlist. Sorting by title will bring visibility to those older films in the list, but they might no longer be of interest to the user while increasing friction point for user to select a film of interest to start playing. 

## Comment 6 — Rebase
**What conflicted:** `.gitignore` appeared as the only conflict. The difference between local and remote main was `pytest_cache` and 2 packages (`venv` and `.venv/`).
**How I resolved it:** Keep the current version since my branch has all the contents as the remote, plus the `pytest_cache` which was applicable due to tests. 
**How I verified no conflict remains:** Able to run the `git add .gitignore` and `git rebase --continue` successfully after resolving conflicts. Then, I ran the entire test suite with `pytest /tests -v` to ensure all tests should be passing as before. However, tests with `WatchlistEntry` references started to break, since the remote main doesn't have this model added. To resolve the test failures, I have bought back the `WatchlistEntry` class and all tests are currently passing as before.

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->

### Overview

This PR adds the **watchlist** feature: a way for users to save films they intend to watch later, separate from their `collection` (films they've already watched and rated). It introduces:

- `WatchlistEntry` model (`models.py`) — tracks `user_id`, `film_id`, `date_added`, and `public`, with a unique constraint on `(user_id, film_id)` to prevent duplicate entries at the DB level.
- `services/watchlist_service.py` — `add_to_watchlist(user_id, film_id)` and `get_watchlist(user_id)`, raising `FilmNotFoundError` for an unknown film and `AlreadyInWatchlistError` for a duplicate add.
- `routes/watchlist/watchlist.py` — registered at `/watchlist`:
  - `GET /watchlist/<user_id>` — returns the user's watchlist as a list of film dicts, each annotated with `date_added` and `public`.
  - `POST /watchlist/<user_id>/add` — body `{ "film_id": <id> }`, adds a film to the watchlist and returns the new entry (`201`), or `404`/`409` on the error cases above.

### Design decisions

1. **Visibility default — `public=True`.** New watchlist entries default to public. This supports user-to-user engagement around shared film interests (e.g. seeing what others want to watch) and gives visibility into aggregate interest in a film. Tradeoff: this exposes a user's film preferences by default, which not every user may want — see Comment 4 above for the full reasoning and the acknowledged privacy tradeoff.
2. **Sort order — `date_added` descending.** `get_watchlist()` returns the most recently added films first, since the newest additions are the closest reflection of a user's current taste and what they're most likely to want to watch next. See Comment 5 above for why this was chosen over sorting by title.

### Manual testing

Start the app (`python app.py` or however it's run locally) and confirm a `user` and at least one `film` already exist in the DB (needed to satisfy the foreign keys below). Then, using `curl` or Postman:

1. **Add a film to the watchlist (happy path)**
   ```
   POST /watchlist/<user_id>/add
   Body: { "film_id": <existing_film_id> }
   ```
   Expect `201` with the new entry, including `"public": true` and a `date_added` timestamp.

2. **Add a second, different film to the same watchlist**
   ```
   POST /watchlist/<user_id>/add
   Body: { "film_id": <another_existing_film_id> }
   ```
   Expect `201`.

3. **View the watchlist and check sort order**
   ```
   GET /watchlist/<user_id>
   ```
   Expect both films back, newest-added film first (i.e. the film added in step 2 appears before the one from step 1).

4. **Attempt to re-add the same film (deduplication)**
   ```
   POST /watchlist/<user_id>/add
   Body: { "film_id": <film_id_from_step_1> }
   ```
   Expect `409` with an error message stating the film is already in the watchlist. Confirm via `GET /watchlist/<user_id>` that no duplicate entry was created.

5. **Attempt to add a nonexistent film**
   ```
   POST /watchlist/<user_id>/add
   Body: { "film_id": "00000000-0000-0000-0000-000000000000" }
   ```
   Expect `404` with an error message stating no film was found for that id.

6. **Attempt to add without a `film_id`**
   ```
   POST /watchlist/<user_id>/add
   Body: {}
   ```
   Expect `400` with `{"error": "film_id is required"}`.

7. **Confirm the `public` field is present on every entry**
   Re-run `GET /watchlist/<user_id>` and check each film dict includes `"public": true`, matching the default-visibility design decision above.