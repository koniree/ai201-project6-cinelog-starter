# PR Response Doc — CineLog Watchlist Feature

## AI Usage
Used AI (Claude) for several distinct purposes on this PR, each with a different amount of autonomy:

- **Comment 2 (dedup):** Asked AI to explain what `add_to_collection()`'s duplicate check does and what it returns on a duplicate — not to write the code. I wrote the `AlreadyInWatchlistError` check myself based on that explanation.
- **Comments 4 & 5 (design arguments):** I gave AI my actual position on each (keep `public=True`, keep alphabetical sort) and had it stress-test the reasoning as a devil's advocate, per the assignment's suggested workflow. It surfaced two real gaps I revised for:
  - Comment 4: I'd initially cited `get_collection()` having no privacy flag at all as precedent for defaulting watchlist to public. AI pointed out that's an unreviewed gap in the existing model, not a deliberate design precedent — I dropped that argument since it wouldn't hold up if the maintainer pushed back on it.
  - Comment 5: I'd only argued alphabetical is "faster to scan." AI raised that as a watchlist grows, a user browsing to remind themselves what they queued up benefits from recency, and that two structurally identical endpoints (`get_collection`, `get_watchlist`) returning differently-ordered lists is a real frontend integration cost, not just a code-uniformity nitpick. I added an explicit rebuttal to both points instead of ignoring them.
- **Comment 6 (rebase):** Had AI drive the rebase mechanics (fetch, rebase, diagnosing what main's UUID-migration commit changed, restoring the deleted `WatchlistEntry` model, updating stale integer-ID references) and run the verification. This was mechanical/systematic work, not a design judgment call, so I delegated it directly rather than doing it by hand.
- **Milestone 4 (commit history cleanup):** Asked AI to review the branch's commit history against Conventional Commits format and CONTRIBUTING.md's "one logical change per commit" rule. It found one commit (`2d3dc26`, the pre-existing `db.session.get` fix) that bundled two unrelated changes — a blueprint import-path fix in `app.py` and the `Query.get()` → `db.session.get()` migration across two service files — and split it into two commits. It also caught that my earlier `feat: add deduplication check...` commit should have been `fix:` per CONTRIBUTING.md's own definition (bug fix, not a new feature/endpoint), and that my docs commit message undersold its actual scope (it described itself as covering "Comments 4-6" when the diff showed it introduced the entire file, Comments 1-6). I verified each of these against the diffs myself before accepting them rather than taking the AI's word for it.

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py`. Searched the codebase for every call site with a project-wide search for `save_to_watchlist` and found exactly one, in `routes/watchlist/watchlist.py` (both the import statement and the call inside `add_film()`), and updated it to `add_to_watchlist`. Re-ran the same search afterward and confirmed zero remaining references to the old name.
**How I verified:** `pytest tests/ -v` — all tests passed. Committed as `refactor: rename save_to_watchlist to add_to_watchlist` (final hash `e28032d` after both the Comment 6 rebase and the Milestone 4 history cleanup rewrote it — see the commit log at the bottom of this doc).

## Comment 2 — Deduplication
**What I did:** Added a duplicate check to `add_to_watchlist()`, modeled on `add_to_collection()` in `services/collection_service.py`. Before creating a `WatchlistEntry`, the function now queries `WatchlistEntry.query.filter_by(user_id=user_id, film_id=film_id).first()`; if an entry already exists it raises a new `AlreadyInWatchlistError` (defined at the top of `watchlist_service.py`, mirroring `AlreadyInCollectionError`) instead of silently inserting a duplicate row. I read `add_to_collection()` first to understand the check and what it returns on a duplicate (an exception, not a boolean), then wrote the watchlist version myself rather than asking AI to generate it.
**How I verified:** `pytest tests/ -v` — all 5 existing tests still pass. Manually traced the logic: existing entry → `AlreadyInWatchlistError` raised, no `db.session.add`/`commit` reached; no existing entry → falls through to normal insert, same as before.

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py` and wrote `test_add_to_watchlist_nonexistent_film_raises`, using `test_add_to_collection_nonexistent_film_raises` in `tests/test_collection.py` as the model. Reused the same fixture structure (`app`, `sample_user`, `sample_film` — in-memory SQLite via `create_app`), the same fake-UUID pattern for a nonexistent film, and the same `pytest.raises(FilmNotFoundError)` assertion, adapted to call `add_to_watchlist` instead of `add_to_collection`.
**How I verified:** `pytest tests/test_watchlist.py -v` — passed. Then `pytest tests/ -v` — all 5 tests passed. Committed as `test: add test for add_to_watchlist with nonexistent film` (final hash `848dc0a` after both the Comment 6 rebase and the Milestone 4 history cleanup rewrote it — see the commit log at the bottom of this doc).

## Comment 4 — Default visibility
**My position:** Keep `public=True` as the default on `WatchlistEntry`.

**Reasoning:** CineLog is, per its own README, "a community film tracking app" — the product's value comes from social visibility, not just personal logging. A watchlist that's private by default launches into an empty social graph: nobody sees anyone's plans, so the exact behavior the feature exists to enable — "see what your friends are planning to watch," "get reminded a film is worth watching because someone you follow queued it" — never happens for the overwhelming majority of users, who don't go looking for a privacy toggle before they've even used the feature once. Defaults are what most users experience; an opt-in social feature is functionally a feature nobody uses.

**Tradeoff acknowledged:** A watchlist reveals *intent*, not just history, and that's a meaningfully different privacy surface than a collection entry (a completed, past action). Someone queuing a documentary about a health condition, a breakup movie, or anything they're not ready to have surfaced to their whole follower list doesn't get a chance to make that call before the first save — most users won't discover the `public` flag exists until after they've already been exposed once. That's a real cost of the default, not a hypothetical one. I don't think it outweighs the network-effect argument above, but it's worth the team considering a lightweight mitigation later (e.g., a one-time visibility prompt on a user's first watchlist add) rather than treating the toggle as sufficient on its own.

*(Note: I initially also argued that `get_collection()` has no privacy control at all, so defaulting watchlist to public was "consistent" with existing behavior. I dropped that after stress-testing it — the collection endpoint having zero visibility control looks like an unreviewed gap in the current model, not a deliberate precedent, so it's not solid ground to build a new argument on.)*

## Comment 5 — Sort order
**My position:** Keep alphabetical (by film title), rather than switching to date-added.

**Reasoning:** `get_collection()` and `get_watchlist()` look structurally similar but serve different purposes. A collection is an activity log — "what did I watch and when" — where recency is the meaningful axis, so newest-first makes sense. A watchlist is a working list a user returns to repeatedly for two tasks: deciding what to watch next, and checking whether a film is already queued before adding it again. Both of those are lookup tasks, not "what did I do recently" tasks, and alphabetical-by-title is the faster path to a specific answer, especially once the list has grown past a handful of entries a user can hold in memory chronologically.

**Engagement with reviewer's point:** The maintainer's consistency argument is legitimate and I don't want to wave it away: two endpoints with the same shape (`GET /<user_id>` returning a list of film dicts) returning differently-ordered results is a real cost, not just a nitpick — a frontend consuming both endpoints has to remember that one is newest-first and the other is alphabetical, which is exactly the kind of inconsistency that causes bugs when someone reuses a list component across both views. I also considered the opposite of my own argument: as a watchlist grows, a user may be scanning to remind themselves what they queued up recently rather than searching for a specific title, and recency genuinely helps that use case. Both of those are fair pushback. On balance I still land on alphabetical because the primary watchlist interaction (find/check a specific film) is more common than the recall-what-I-added-recently one, but I'd be comfortable revisiting this if usage data said otherwise — this is a preference, not a correctness issue, so I'm not going to die on this hill if the maintainer feels strongly post-review.

## Comment 6 — Rebase
**What conflicted:** No textual merge conflicts occurred — `git rebase origin/main` applied cleanly. The real conflict was semantic: main's `refactor: migrate film IDs from integer to UUID` commit (`07ca580`) changed `Film.id` and `CollectionEntry.film_id` from `Integer` to `String(36)` UUIDs, and — because the watchlist feature didn't exist on main yet — that same commit deleted the `WatchlistEntry` model entirely from `models.py`. Since my branch never modified `models.py` directly, the rebase silently replayed main's version (without `WatchlistEntry`) with no conflict markers, which would have broken every import in `services/watchlist_service.py` and `routes/watchlist/watchlist.py` if left unaddressed.

**How I resolved it:** Re-added the `WatchlistEntry` class to `models.py`, this time with `film_id` as `db.Column(db.String(36), db.ForeignKey("film.id"), ...)` to match the new UUID type on `Film.id` and follow the same pattern as `CollectionEntry.film_id`. Updated the stale integer-ID docstring in `add_to_watchlist()` (`services/watchlist_service.py`) from `film_id (int)` to `film_id (str): UUID of the film`, and fixed the matching `{ "film_id": <int> }` example in the route docstring (`routes/watchlist/watchlist.py`) to `<uuid>`. Also had to move aside an untracked local `.gitignore` before rebasing, since main's `chore: add .gitignore` commit now tracks a superset of the same file and the checkout would have collided with it; confirmed the tracked version covered everything the local one had and discarded the local copy.

**How I verified no conflict remains:** `pytest tests/ -v` — all 5 tests pass post-rebase. Grepped the whole repo for any remaining integer-style film ID usage (`film_id = <digit>`, `Film(id=<digit>`) and found none outside the expected UUID string fixtures. Confirmed `git log --merges origin/main..HEAD` is empty, meaning the rebase itself introduced no merge commits (the one merge commit visible in `git log --merges` belongs to main's own inherited history, not to this branch). The UUID fix was committed separately as `fix: migrate watchlist to UUID film IDs after rebase onto main`.

## Commit History (Milestone 4)

Cleaned up with `git rebase -i origin/main` (7 original commits → 8, after splitting one that bundled two unrelated fixes). Final `git log --oneline origin/main..HEAD`:

```
301cf95 docs: add pr-response.md with responses to all six review comments
2a04d65 fix: migrate watchlist to UUID film IDs after rebase onto main
eae313a fix: add deduplication check to add_to_watchlist
848dc0a test: add test for add_to_watchlist with nonexistent film
e28032d refactor: rename save_to_watchlist to add_to_watchlist
1d023bd fix: use db.session.get instead of deprecated Query.get in collection and watchlist services
9f70d72 fix: correct watchlist blueprint import path in app.py
777a73b feat: add watchlist model, service, and endpoint
```

(Note: the top commit's own hash above will be slightly stale — amending this file's content necessarily changes that commit's hash, so it can never fully describe itself. Run `git log --oneline origin/main..HEAD` locally to see the exact current hash; the other seven are stable.)

`git log --merges origin/main..HEAD` is empty — no merge commits on the branch.

**[SCREENSHOT NEEDED]** — The assignment asks for a screenshot of this output; the text above is accurate but paste an actual screenshot here before submitting, since I can't generate an image.

## PR Description

### What this feature does
Adds a watchlist to CineLog: users can save films they intend to watch later (`POST /watchlist/<user_id>/add`) and view their saved list (`GET /watchlist/<user_id>`), sorted alphabetically by title. Each entry tracks when it was added and whether it's visible to other users (`public`, defaults to `True`).

### Design decisions
- **Default visibility (`public=True`):** Watchlists default to visible to other users, optimizing for CineLog's social/discovery use case over individual privacy-by-default. Full reasoning and the acknowledged tradeoff (watchlists reveal intent, not just history) are in Comment 4 above.
- **Sort order (alphabetical by title):** `get_watchlist()` sorts alphabetically rather than by date-added, because a watchlist is a lookup/reference list (find or check a film) rather than an activity log where recency is the primary axis. Full reasoning and engagement with the alternative are in Comment 5 above.

### How to manually test
1. `pip install -r requirements.txt && python app.py` (starts on `http://localhost:5000`)
2. Create a user and a film via the existing endpoints (or seed the DB directly), noting their UUIDs.
3. Add a film to the watchlist:
   ```
   POST /watchlist/<user_id>/add
   Body: { "film_id": "<film-uuid>" }
   ```
   Expect `201` with the new entry (including `public: true`).
4. Retry the same request with the same `user_id`/`film_id` — expect a duplicate-entry error (`AlreadyInWatchlistError`), not a second row.
5. Retry with a nonexistent `film_id` (e.g. `00000000-0000-0000-0000-000000000000`) — expect a `FilmNotFoundError`, not a raw DB error.
6. `GET /watchlist/<user_id>` — expect the film(s) just added, sorted alphabetically by title, each with `date_added` and `public`.
7. `pytest tests/ -v` — all tests should pass (5/5).