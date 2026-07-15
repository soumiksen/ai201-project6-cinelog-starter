# PR Response Doc – CineLog Watchlist Feature

## AI Usage
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
**What conflicted:**
**How I resolved it:**
**How I verified no conflict remains:**

## PR Description
