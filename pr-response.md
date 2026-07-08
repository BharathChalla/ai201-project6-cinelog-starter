# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used Claude Code (Anthropic) throughout this project as a pair-programming/orientation tool, primarily for the mechanical and research pieces of the workflow rather than the design reasoning:
- **Codebase orientation (Milestone 1):** Had it read `models.py`, `services/collection_service.py`, and `tests/test_collection.py` before touching any review comment, and summarize the `verb_to_noun` naming convention, the `AlreadyInCollectionError`-style deduplication pattern, and the fixture/assertion structure `test_collection.py` uses. I verified this against the actual code before writing `add_to_watchlist()`'s deduplication check and `tests/test_watchlist.py` — the summary matched the code, so I followed the pattern directly rather than inventing my own.
- **Retrieving the review comments:** Used it to fetch the PR's inline review comments and conversation comments from the GitHub API (`pulls/1/comments` and `issues/1/comments` on the upstream repo) since forking doesn't carry over the PR itself, and to map the six raw comments onto the doc's six numbered slots by content.
- **Git mechanics:** Used it to run the `git rebase origin/main` conflict resolution and the scripted interactive rebase (`git rebase -i`) that squashed/reworded commits into the final conventional-commit history — the *plan* for which commits to squash/reword/keep was mine (see the rebase plan I approved before it ran), and it executed it and reported the diagnostic that a `.gitignore` add/add conflict and a silently-dropped `WatchlistEntry` model needed manual fixes.
- **Comments 4 & 5 (design decisions):** I did not ask AI to draft these arguments. Both `pr-response.md` sections above were written from directly reading the codebase (e.g., noticing `CollectionEntry` has no visibility field at all, or that a watchlist has no natural upper bound the way a "recent activity" feed does) rather than generic reasoning about privacy defaults or sort orders. I did not run a separate devil's-advocate pass on these with a second AI call in this session; the tradeoff/engagement paragraphs above represent my own attempt to anticipate the maintainer's counterarguments directly.

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` to match the project's `verb_to_noun` convention used by `add_to_collection()`. Updated the one call site in `routes/watchlist/watchlist.py` (both the import and the function call).
**How I verified:** Ran a project-wide search (`grep -rn "save_to_watchlist"`) across the repo to confirm there was exactly one call site outside this file, then updated it. Ran `pytest tests/ -v` to confirm nothing else referenced the old name and the suite still passed.

## Comment 2 — Deduplication
**What I did:** Followed `add_to_collection()`'s pattern exactly: before creating a `WatchlistEntry`, query for an existing entry with the same `user_id`/`film_id` pair, and raise a new `AlreadyOnWatchlistError` (mirroring `AlreadyInCollectionError`) if one exists, instead of silently inserting a duplicate. I also added the corresponding `except` clause in the `/add` route so it returns `409 Conflict` with an error message, matching how `routes/collection.py` handles `AlreadyInCollectionError`. I didn't add a `UniqueConstraint` at the model level for this PR — `add_to_collection()` uses both the constraint and the query check together, but adding a schema migration wasn't part of this comment's ask, and the query check alone matches the observable behavior the reviewer asked for.
**How I verified:** Added `test_add_to_watchlist_duplicate_raises` in `tests/test_watchlist.py`, following the same shape as `test_add_to_collection_duplicate_raises`: add once, assert the second add raises, then assert the row count in the DB is still 1 (not just that an exception was raised). Ran `pytest tests/ -v` — all tests pass.

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py` and wrote `test_add_to_watchlist_nonexistent_film_raises`, modeled directly on `test_add_to_collection_nonexistent_film_raises` in `tests/test_collection.py`: same `app`/`sample_user` fixtures, same shape (call with a film id that doesn't exist, assert `FilmNotFoundError` is raised via `pytest.raises`). I also carried over the `app`/`sample_user`/`sample_film` fixtures from `test_collection.py` so the whole watchlist test file follows the existing structure, not just this one test.
**How I verified:** Ran `pytest tests/test_watchlist.py -v` to confirm the new test passes on its own, then `pytest tests/ -v` to confirm the full suite (collection + watchlist) still passes together.

## Comment 4 — Default visibility
**My position:** Keep `public=True` as the default for `WatchlistEntry`.
**Reasoning:** CineLog describes itself as a community film-tracking app, and a watchlist is only useful as a *social* signal if it's actually visible — it lets other users see what you're planning to watch, which is the kind of thing that drives discovery on a community platform (comparable to how public watchlists function on Letterboxd, the closest real-world analogue to this app). If the default were private, the overwhelming majority of users would never flip it to public — defaults are sticky, most users don't revisit settings after the fact — and the feature would quietly become a private to-do list instead of the community feature it was designed to be. I also looked at `CollectionEntry` (the sibling "already watched" model) for precedent: it has no visibility field at all, meaning the app currently treats a user's film activity as implicitly non-private everywhere else. Defaulting `WatchlistEntry.public` to `False` would introduce the first privacy boundary anywhere in the app, inconsistently, on the one model where the reviewer happened to add a boolean for it.
**Tradeoff acknowledged:** The real cost of `public=True` is that a user's watchlist — which can reveal genre/franchise interests, or a want-to-watch queue for something like a film they're a fan of but haven't disclosed — gets exposed before they've made an affirmative choice to share it. That's a real privacy tradeoff, not a hypothetical one, and "everyone else defaults this way" isn't itself a justification. To address it without abandoning the default, I implemented the stretch feature that adds an explicit `public` parameter to `add_to_watchlist()` (see the stretch section below) so a privacy-conscious caller — or a future client that wants a "private by request" flow — can opt out per-entry at creation time instead of only being able to change it after the fact.

## Comment 5 — Sort order
**My position:** I'm pushing back and keeping `get_watchlist()` sorted alphabetically (`Film.title.asc()`) rather than switching to `date_added` descending.
**Reasoning:** Collection and watchlist look like the same shape of data (a user, a film, a timestamp) but they answer different questions. `CollectionEntry` is a historical log — "what have I watched, and when" — so newest-first is the right default there; it behaves like a diary, and diaries are read most-recent-entry-first. `WatchlistEntry` is a *standing set of intent* — every film on it is equally "still wanted" regardless of when it was added, and nothing prunes an entry automatically just because time passed (it stays until the user watches it and removes it themselves). That means a watchlist tends to grow for as long as a user keeps it, and if we sort by `date_added` descending, films added early get pushed toward the bottom of a list that has no natural upper bound — effectively burying titles the user still wants to watch, which directly undermines the point of a watchlist (surfacing things to watch). Alphabetical order doesn't have that decay problem: every title stays equally easy to find no matter how long it's been on the list.
**Engagement with reviewer's point:** @dev-lead's argument is "most users want to see what they added recently," and that's a reasonable intuition for an activity feed — but a watchlist isn't a feed, it's closer to a reference list you scan to answer "is X on here" or "what should I watch tonight." I'd rather not default to an ordering that quietly penalizes older, still-valid entries just because they're old. That said, I don't think the reviewer's underlying need is wrong — some users probably do want to see what they just added, especially right after adding it. Rather than resolve that by changing the default, I think the better long-term answer is a `?sort=` query param on `GET /watchlist/<user_id>` (`date_added` vs `title`) so both use cases are supported explicitly instead of picking one winner — but that's beyond the scope of this comment/PR, so for now I'm keeping the existing alphabetical default and flagging the query-param idea as a natural follow-up.

## Stretch Features

**`remove_from_watchlist(user_id, film_id)`:** Added in `services/watchlist_service.py`, following the exact naming and structure of `remove_from_collection()`: look up the entry by `user_id`/`film_id`, raise a new `NotOnWatchlistError` (mirroring `NotInCollectionError`) if it isn't found, otherwise delete and commit, returning `True`. Wired up as `DELETE /watchlist/<user_id>/remove` in `routes/watchlist/watchlist.py`, mirroring `routes/collection.py`'s `remove_film` (404 on `NotOnWatchlistError`). Covered by `test_remove_from_watchlist_deletes_entry` and `test_remove_from_watchlist_not_present_raises` in `tests/test_watchlist.py`.

**Second test (edge case):** Added `test_get_watchlist_returns_alphabetical_regardless_of_add_order`, which adds two films to a watchlist in reverse-alphabetical order and asserts `get_watchlist()` still returns them alphabetically. I chose this case specifically because it's the one most likely to silently break the Comment 5 decision: an implementation could easily "accidentally" sort by insertion order or primary key order and still pass a test that only adds one film, or adds films in alphabetical order by coincidence. Writing this test is also what surfaced a real bug: `get_watchlist()` calls `entry.film.to_dict()`, but `models.py` never defined a `film` backref on `WatchlistEntry` (only `CollectionEntry` had one via `Film.collection_entries`). I added `Film.watchlist_entries = db.relationship("WatchlistEntry", backref="film", lazy=True)` to fix it — without this, `GET /watchlist/<user_id>` would throw an `AttributeError` on every call, so `get_watchlist()` was actually unreachable before this PR.

**Visibility toggle (`public` param):** Added an optional `public` parameter to `add_to_watchlist(user_id, film_id, public=True)`, and exposed it in `POST /watchlist/<user_id>/add` via an optional `"public"` key in the request body (defaults to `True` if omitted, preserving the Comment 4 default). This directly supports the Comment 4 position: callers who want privacy can opt out explicitly at creation time instead of the API only ever producing public entries. Covered by `test_add_to_watchlist_respects_public_false`.

## Comment 6 — Rebase
**What conflicted:** I fetched `upstream/main` (which had merged `refactor: migrate film IDs from integer to UUID`, changing `Film.id` from `db.Integer` to `db.String(36)` with a `generate_uuid` default, and updating `CollectionEntry.film_id` to match) and ran `git rebase origin/main`. Git reported one textual conflict, in `.gitignore`: my branch added `.gitignore` from scratch during Milestone 1 setup, while `main` had already merged a `.gitignore` from a separate PR (`chore: add .gitignore for generated files`) — an add/add conflict on the same path with different content. More importantly, git *silently* mis-merged `models.py`: my branch's first replayed commit added the whole `WatchlistEntry` class (with `id`/`user_id` as UUID strings, but `film_id` still `db.Integer`, since it predated the refactor) directly after `CollectionEntry`, in the same region of the file the refactor commit had also touched. Git's three-way merge resolved that region without flagging a conflict — but the resolution it picked dropped the `WatchlistEntry` class entirely. `pytest` failed immediately after the rebase finished with `ImportError: cannot import name 'WatchlistEntry' from 'models'`, which is what caught it; there was no conflict marker in the file to notice by eye.
**How I resolved it:**
1. For `.gitignore`, I merged both versions by hand — kept `.pytest_cache/` (from `main`'s version) alongside `.venv/`, `*.db`, `__pycache__/`, etc. (from my version) — and removed the `<<<<<<<`/`=======`/`>>>>>>>` markers before continuing the rebase.
2. For the silently-dropped model, once the rebase finished I re-added the `WatchlistEntry` class to `models.py` (after `CollectionEntry`, matching the file's existing layout), but with `film_id = db.Column(db.String(36), db.ForeignKey("film.id"), ...)` instead of the old `db.Integer`, so it lines up with the post-refactor `Film.id` type — matching exactly how the refactor commit updated `CollectionEntry.film_id`. I also updated the `film_id` docstrings in `services/watchlist_service.py` and the request-body comments in `routes/watchlist/watchlist.py` from `int` to UUID `str`, and changed `tests/test_watchlist.py`'s nonexistent-film test to use a UUID-shaped placeholder (`"00000000-0000-0000-0000-000000000000"`) instead of an integer, matching `test_collection.py`'s pattern.
**How I verified no conflict remains:** Ran `git status` to confirm no unmerged paths remained, then `git rebase --continue`, which completed cleanly (`Successfully rebased and updated refs/heads/feature/watchlist`). Ran `pytest tests/ -v` afterward — all 11 tests (4 collection + 7 watchlist) pass, including the watchlist tests that create/query/delete `WatchlistEntry` rows against the now-UUID `film_id` column, which would fail immediately if the type mismatch weren't actually fixed. Confirmed with `git log --oneline` that `feature/watchlist` now sits on top of `origin/main` with no merge commits (`git rebase`, not `git merge`, was used throughout).

## PR Description

### What this feature does
Adds a watchlist to CineLog so users can save films they intend to watch, separate from their collection (films already watched). It introduces:
- `WatchlistEntry` model (`user_id`, `film_id`, `date_added`, `public`), UUID-keyed to match the post-refactor `Film`/`User`/`CollectionEntry` models.
- `add_to_watchlist(user_id, film_id, public=True)` — adds a film to a user's watchlist, rejecting nonexistent films (`FilmNotFoundError`, 404) and duplicate entries (`AlreadyOnWatchlistError`, 409).
- `remove_from_watchlist(user_id, film_id)` — removes a film from the watchlist, 404 (`NotOnWatchlistError`) if it isn't there.
- `get_watchlist(user_id)` — returns a user's watchlist sorted alphabetically by title.
- Routes: `GET /watchlist/<user_id>`, `POST /watchlist/<user_id>/add`, `DELETE /watchlist/<user_id>/remove`.

### Design decisions
- **Default visibility (`public=True`):** Watchlists default to public because CineLog is a community app and a watchlist only functions as a discovery/social signal if it's visible; `CollectionEntry` has no visibility concept at all, so this keeps the app's implicit privacy posture consistent instead of introducing an inconsistent boundary. Callers who need privacy can now pass `"public": false` explicitly via the new `public` parameter/request field. See Comment 4 for the full reasoning and the acknowledged tradeoff.
- **Sort order (alphabetical, not date-added):** `get_watchlist()` stays sorted alphabetically rather than switching to newest-first. A watchlist is a standing set of "still want to watch" intent, not an activity log — sorting by recency would bury older, equally-valid entries at the bottom of a list with no natural size limit. See Comment 5 for the full argument and direct engagement with the maintainer's recency-based reasoning.

### How to manually test
1. Set up the environment and run the app:
   ```
   python -m venv .venv
   source .venv/Scripts/activate   # or .venv\Scripts\activate.bat on Windows cmd
   pip install -r requirements.txt
   python app.py
   ```
2. Create a user and a film via the existing endpoints (or use fixtures/seed data), noting their UUIDs.
3. Add a film to the watchlist (defaults to public):
   ```
   curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
     -H "Content-Type: application/json" \
     -d '{"film_id": "<film_id>"}'
   ```
4. Add a second film as private, to exercise the visibility toggle:
   ```
   curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
     -H "Content-Type: application/json" \
     -d '{"film_id": "<film_id_2>", "public": false}'
   ```
5. Confirm duplicate rejection — repeat step 3 with the same `film_id` and confirm a `409` with an `AlreadyOnWatchlistError` message.
6. View the watchlist and confirm alphabetical ordering by title, regardless of the order the films were added:
   ```
   curl http://127.0.0.1:5000/watchlist/<user_id>
   ```
7. Remove a film and confirm it disappears from the list:
   ```
   curl -X DELETE http://127.0.0.1:5000/watchlist/<user_id>/remove \
     -H "Content-Type: application/json" \
     -d '{"film_id": "<film_id>"}'
   ```
8. Confirm removing a film not on the list returns `404` with a `NotOnWatchlistError` message.
9. Run the automated test suite: `pytest tests/ -v` — all 11 tests (4 collection, 7 watchlist) should pass.

### Commit history
`feature/watchlist`, rebased on `origin/main`, no merge commits — 10 conventional commits below, plus one final commit (not shown here, since a screenshot taken before a commit exists can never include that commit) that adds the `git log --oneline` screenshot itself:
```
(this commit) docs: add pr-response.md with review responses and design decisions
36298bd fix: update WatchlistEntry film_id to UUID after main branch refactor
f03988e test: verify get_watchlist sorts alphabetically regardless of add order
1d92908 fix: add missing Film.watchlist_entries relationship for entry.film access
a3e0ace feat: add public parameter to add_to_watchlist for explicit visibility control
8517157 feat: add remove_from_watchlist following collection removal pattern
6c32af8 test: add test for nonexistent film_id in add_to_watchlist
65a8c78 fix: add deduplication check to prevent duplicate watchlist entries
ec731d7 fix: rename save_to_watchlist to add_to_watchlist per naming convention
9aa0da2 feat: add watchlist model and add_to_watchlist endpoint
```
See the screenshot below (added in the follow-up commit) for the real, final `git log --oneline` output.
