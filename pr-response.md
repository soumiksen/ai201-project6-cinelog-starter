# PR Response Doc – CineLog Watchlist Feature

## AI Usage
## Comment 1 – Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` to match the `verb_to_noun` convention used elsewhere (`add_to_collection`, `remove_from_collection`, `get_collection`). Updated the two call sites in `routes/watchlist/watchlist.py`: the import statement and the call inside `add_film()`.
**How I verified:** `grep -rn "save_to_watchlist"` across the repo returned zero matches after the edit. Ran `python -m py_compile` on both changed files, imported the app factory (`create_app`) to confirm no `ImportError`, and ran the full test suite (`pytest tests/ -v`) — all 4 existing tests still pass.

## Comment 2 – Deduplication
**What I did:**
**How I verified:**

## Comment 3 – Missing test
**What I did:**
**How I verified:**

## Comment 4 – Default visibility
**My position:**
**Reasoning:**
**Tradeoff acknowledged:**

## Comment 5 – Sort order
**My position:**
**Reasoning:**
**Engagement with reviewer's point:**

## Comment 6 – Rebase
**What conflicted:**
**How I resolved it:**
**How I verified no conflict remains:**

## PR Description
