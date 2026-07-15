# PR Response Doc – CineLog Watchlist Feature

## AI Usage
I used Claude Code throughout this PR, in three distinct ways, and my own position changed as a result of two of them:

1. **Orientation.** Before touching any code, I had Claude read `models.py`, `services/collection_service.py`, and `tests/test_collection.py` to establish the naming/dedup/testing patterns to mirror, and had it attempt to fetch the six `@dev-lead` review comments. It reported back that no PR actually existed on GitHub for this branch and no comment-fetching tool was available, rather than inventing plausible-sounding review text — so instead of getting handed six comments to react to, I got a table of *hypothesized* comment subjects derived from diffing the branch against `main` and the code's own gaps (e.g., "no dedup check in `save_to_watchlist`," "no test file exists"), which I then confirmed were the right things to work on.
2. **Stress-testing my design instincts (Comments 4 & 5).** For default visibility, I initially assumed public-by-default was fine since CineLog is a social app; Claude pushed back with the asymmetry argument (a watchlist reveals intent, not a completed action, so oversharing is harder to undo than under-sharing), which changed my position to private-by-default. For sort order, I initially favored alphabetical for browsability; playing devil's advocate against my own instinct surfaced that the browsability benefit was speculative while the API-consistency cost (two structurally identical endpoints behaving differently) was concrete, which is what actually changed my position to newest-first.
3. **Troubleshooting the rebase (Comment 6).** When `git rebase origin/main` conflicted in `models.py`, Claude diagnosed that the conflict wasn't a simple text disagreement but a real schema mismatch (`main` had already migrated `Film.id` to UUID, our `WatchlistEntry.film_id` was still `Integer`), and traced the fallout into files Git didn't flag as conflicted at all — stale `int` docstrings and a test fixture that would've silently passed for the wrong reason. It also caught a pre-existing bug unrelated to any of the six comments (a missing SQLAlchemy relationship on `Film` that made `get_watchlist()` crash) by writing a test that actually exercised the function for the first time.

Every code change and design position above was reviewed and approved by me before being committed — the AI proposed and drafted, but the decisions (including the final default-visibility and sort-order stances) were mine.

## Comment 1 – Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` to match the `verb_to_noun` convention used elsewhere (`add_to_collection`, `remove_from_collection`, `get_collection`). Updated the two call sites in `routes/watchlist/watchlist.py`: the import statement and the call inside `add_film()`.
**How I verified:** `grep -rn "save_to_watchlist"` across the repo returned zero matches after the edit. Ran `python -m py_compile` on both changed files, imported the app factory (`create_app`) to confirm no `ImportError`, and ran the full test suite (`pytest tests/ -v`) — all 4 existing tests still pass.

## Comment 2 – Deduplication
**What I did:** Added an `AlreadyInWatchlistError` exception and a duplicate check to `add_to_watchlist()` in `services/watchlist_service.py`, mirroring `add_to_collection()`'s pattern exactly: after confirming the film exists, query `WatchlistEntry.query.filter_by(user_id=user_id, film_id=film_id).first()` and raise `AlreadyInWatchlistError` if a row is found, before constructing or inserting the new entry.
**How I verified:** Confirmed the module still compiles (`python -m py_compile`) and the full existing test suite still passes (`pytest tests/ -v`, 4/4). A dedicated test for this behavior (`test_add_to_watchlist_duplicate_raises`) is added in Task 3/Comment 3 below.

## Comment 3 – Missing test
**What I did:** Created `tests/test_watchlist.py`, modeled directly on `tests/test_collection.py` (same `app`/`sample_user`/`sample_film` fixtures, same `with app.app_context():` structure). Added three tests covering `add_to_watchlist()`: happy path (`test_add_to_watchlist_creates_entry`), duplicate handling (`test_add_to_watchlist_duplicate_raises`, matching Comment 2's new `AlreadyInWatchlistError`), and a nonexistent film id (`test_add_to_watchlist_nonexistent_film_raises`) — matching CONTRIBUTING.md's "at least 3 tests for a new service function" requirement.
**How I verified:** Ran `pytest tests/test_watchlist.py -v` — all 3 new tests passed. Then ran the full suite with `pytest tests/ -v` — all 7 tests (4 existing + 3 new) passed with no regressions.

## Comment 4 – Default visibility
**My position:** Watchlist entries should default to private (`public=False`), not public. Implemented by flipping `WatchlistEntry.public`'s column default in `models.py`, with a test (`test_add_to_watchlist_defaults_to_private`) asserting the default.
**Reasoning:** A watchlist reveals *intent* — what a user is curious about before they've committed to it — which is a more sensitive signal than a collection (a retrospective record of films already watched). This follows the standard privacy-by-default principle: the harm from unwanted exposure (an unintentionally public list surfacing something sensitive) is asymmetric with the cost of opting in to share (flipping one boolean). Defaulting private doesn't remove the social/discovery feature, it just makes sharing an explicit choice instead of an implicit one.
**Tradeoff acknowledged:** Fewer watchlists will be publicly visible out of the box, which mutes the discovery/virality effect a community film-tracking app is generally trying to encourage. We're trading some default social engagement for user control.

## Comment 5 – Sort order
**My position:** Conceded — switched `get_watchlist()` from alphabetical-by-title (`Film.title.asc()`) to newest-first (`WatchlistEntry.date_added.desc()`), matching `get_collection()`. Added `test_get_watchlist_returns_newest_first`, mirroring `test_get_collection_returns_newest_first`.
**Reasoning:** Newest-first surfaces what a user just decided they wanted to watch, which is more actionable than an alphabetical scan as the list grows, and it removes the need for a client to special-case ordering per endpoint.
**Engagement with reviewer's point:** I initially considered alphabetical order defensible — a watchlist can function like a list you return to and scan repeatedly, where a stable, browsable order helps (e.g., "did I already add this?"). But that benefit is speculative, whereas the inconsistency between two structurally similar endpoints (`/collection/<user_id>` and `/watchlist/<user_id>`) is a concrete API contract problem: an engineer building against both should not have to remember that one sorts by name and the other by time. The reviewer's consistency argument wins here. If per-list browsing order turns out to matter later, the right fix is a `?sort=` query param on both endpoints, not a permanently different default on one.

## Comment 6 – Rebase
**What conflicted:** `main` had already merged commit `07ca580` ("refactor: migrate film IDs from integer to UUID") before this branch was rebased, which changed `Film.id` and `CollectionEntry.film_id` from `Integer` to `String(36)`. Our branch's `WatchlistEntry` model (added after this branch diverged) still declared `film_id` as `Integer`, so `git rebase origin/main` produced a real conflict in `models.py` when replaying the "default watchlist entries to private" commit — Git couldn't auto-merge because the `WatchlistEntry` class didn't exist yet on the `main` side.
**How I resolved it:** Kept `main`'s post-refactor structure and re-added the `WatchlistEntry` class on top of it, changing its `film_id` column from `db.Column(db.Integer, ...)` to `db.Column(db.String(36), ...)` to match the new `Film.id` type — the same change `CollectionEntry.film_id` already went through in the refactor. After the rebase completed, I also swept `services/watchlist_service.py`'s docstring, `routes/watchlist/watchlist.py`'s request-body docstring, and the `fake_film_id` test fixture in `tests/test_watchlist.py` — all three still said/assumed `int`, which was accurate before the refactor but stale afterward. (Separately, an untracked local `.gitignore` was blocking the rebase from starting; its contents were a strict subset of `main`'s tracked `.gitignore`, so I removed the local copy with no loss of content.)
**How I verified no conflict remains:** `grep -rn "<<<<<<<\|=======\|>>>>>>>"` across all `.py` files returned nothing, `git status` shows a clean rebase with no unmerged paths, and `pytest tests/ -v` passes all 9 tests (4 collection + 5 watchlist) against the rebased, UUID-consistent schema.

## PR Description

### What this adds

This PR adds a **watchlist** feature to CineLog: users can save films they want to watch later, view their saved list, and control whether each saved film is visible to others. It follows the same shape as the existing collection feature (model + service + routes), with its own duplicate-prevention and error handling.

- `POST /watchlist/<user_id>/add` — save a film to a user's watchlist (`{"film_id": "<uuid>"}`)
- `GET /watchlist/<user_id>` — view a user's watchlist, newest-added first
- `add_to_watchlist(user_id, film_id)` raises `FilmNotFoundError` for an unknown film and `AlreadyInWatchlistError` for a duplicate save, matching `add_to_collection()`'s behavior

### Design decisions

- **Default visibility (`public=False`):** watchlist entries are private by default. A watchlist reveals what a user is curious about, not what they've done — a more sensitive signal than a collection — so sharing is opt-in rather than opt-out. See Comment 4 above for the full reasoning and the tradeoff (reduced default social discoverability) we accepted.
- **Sort order (newest-first):** `get_watchlist()` sorts by `date_added` descending, matching `get_collection()`, instead of alphabetically by title. This keeps the two structurally similar endpoints consistent for anyone building against both. See Comment 5 above for the full discussion.

### How to test manually

1. Start the app: `python app.py` (starts on `http://localhost:5000`, using local `cinelog.db`).
2. Create a user and a film (or use existing seed data) so you have a `user_id` and `film_id` (both UUIDs).
3. Save a film to the watchlist:
   ```bash
   curl -X POST http://localhost:5000/watchlist/<user_id>/add \
     -H "Content-Type: application/json" \
     -d '{"film_id": "<film_id>"}'
   ```
   Confirm the response is `201` and the returned entry has `"public": false`.
4. Repeat the same request with the same `user_id`/`film_id` — confirm it now returns a 409-style error (`AlreadyInWatchlistError`) instead of creating a second entry.
5. Add a second film with an earlier manual timestamp if you want to verify ordering, or just add two films back-to-back, then:
   ```bash
   curl http://localhost:5000/watchlist/<user_id>
   ```
   Confirm the most recently added film appears first in the response.
6. Try adding a nonexistent `film_id` (any random UUID) — confirm it returns a not-found error rather than a raw database exception.
