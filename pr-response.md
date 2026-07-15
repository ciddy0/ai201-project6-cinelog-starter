# PR Response Doc — CineLog Watchlist Feature

## AI Usage

I used AI (Claude Code) at several points, always verifying its output against the actual code:

- **Codebase orientation.** Before reading the review comments I had it summarize `models.py`, `services/collection_service.py`, and `tests/test_collection.py` — the naming convention (`verb_to_noun`), how dedup is done (`filter_by(...).first()` → raise a typed error before insert), and the test fixture pattern (`app` / `sample_user` / `sample_film`, `with app.app_context()`). I confirmed each claim by reading the files directly.
- **Understanding `add_to_collection()` for Comment 2.** I had it walk through the dedup path and what the function returns on a duplicate (it *raises*, it does not silently no-op), then wrote my own `add_to_watchlist()` guard rather than having AI write it.
- **Stress-testing the design arguments (Comments 4 & 5).** After drafting each position I asked, as a devil's advocate, "what's the strongest counterargument a careful reviewer would raise, and what tradeoff am I not acknowledging?"
  - *Comment 4:* the pushback was that my draft leaned on growth arguments and hand-waved privacy — specifically that a watchlist can reveal genuinely sensitive interests (health, sexuality, religion) and that "most users are happy to share" is an untested assumption. I changed my final position from a flat "keep `public=True`" to a **conditional** one — public is only defensible *with* pre-first-save disclosure and a one-tap private toggle, and absent those the default should flip to private — and I added the sensitive-content concession explicitly.
  - *Comment 5:* the stress test surfaced that "recency is intuitive" alone is weak; the stronger, codebase-grounded argument is **consistency with `get_collection()`**, which already sorts newest-first. I promoted that to the decisive point in my final response.
- **Verification, not just generation.** AI helped me notice that the original `get_watchlist()` was never actually exercised (the `entry.film` relationship was missing), and I confirmed that independently by running the original code. I also used it to sanity-check that the final commit messages follow conventional-commit format, then verified the log myself.

The reasoning in Comments 4 and 5 is my own; AI was used to attack my drafts, not to author them.

## Comment 1 — Rename

**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` to match the project's `verb_to_noun` convention (cf. `add_to_collection()`). Updated both call sites in `routes/watchlist/watchlist.py` — the `from services.watchlist_service import ...` line and the call inside `add_film()`.

**How I verified:** Before editing I ran a project-wide `grep -rn "save_to_watchlist" --include="*.py" .` which surfaced exactly three references (the definition + the import + the call). After renaming I re-ran the same grep and it returned nothing, confirming no stragglers. `pytest tests/ -v` still passes all 4 tests, so nothing that imports the module broke.

## Comment 2 — Deduplication

**What I did:** Mirrored the pattern in `add_to_collection()`. That function (a) defines an `AlreadyInCollectionError` in its own service module, and (b) after the film-exists check, runs `CollectionEntry.query.filter_by(user_id, film_id).first()` and raises before inserting if a row already exists. I did the equivalent in `add_to_watchlist()`: defined `AlreadyInWatchlistError` in `watchlist_service.py`, and added a `WatchlistEntry.query.filter_by(user_id=user_id, film_id=film_id).first()` guard that raises `AlreadyInWatchlistError` when a duplicate is detected, so no second row is ever created.

I also mirrored the collection *route*: `routes/collection.py` catches its errors and returns 404 (`FilmNotFoundError`) / 409 (`AlreadyInCollectionError`). The watchlist route previously caught nothing, so my new exception would have surfaced as a 500. I wrapped the call in `routes/watchlist/watchlist.py` to return 404 / 409 the same way, so the dedup is handled end-to-end, not just in the service.

**How I verified:** I read `add_to_collection()` first and confirmed the check is: query for an existing `(user_id, film_id)` row → if found, raise the "already in" error *before* the insert (it returns nothing / raises on a duplicate; it does not silently no-op). Then I wrote my own version. I verified with a throwaway script against an in-memory DB: first `add_to_watchlist` succeeds; a second call with the same `(user_id, film_id)` raises `AlreadyInWatchlistError`; the row count stays at 1; and a nonexistent `film_id` still raises `FilmNotFoundError`. `pytest tests/ -v` remains green.

## Comment 3 — Missing test

**What I did:** Created `tests/test_watchlist.py`, using `tests/test_collection.py`'s `test_add_to_collection_nonexistent_film_raises` as the model. I reused the same fixture structure — an `app` fixture with an in-memory SQLite DB and `db.create_all()/drop_all()` teardown, and a `sample_user` fixture — and the same assertion style (`with pytest.raises(FilmNotFoundError): ...`). The new test, `test_add_to_watchlist_nonexistent_film_raises`, calls `add_to_watchlist()` with a film id that has no row and asserts `FilmNotFoundError`.

One detail about the fake id: when I first wrote this test the branch was still pre-refactor (`Film.id` was an integer), so I used a nonexistent integer id (`99999`) to match the column type. After the Comment 6 rebase onto `main`, film IDs are UUIDs, so this test now uses a nonexistent UUID string (`"00000000-0000-0000-0000-000000000000"`), matching `test_collection.py`. That id change is part of the UUID-resolution commit (see Comment 6).

**How I verified:** Ran `pytest tests/test_watchlist.py -v` (1 passed), then the full `pytest tests/ -v` (5 passed) to confirm the new file doesn't interfere with the existing collection tests.

## Comment 4 — Default visibility

**My position:** Keep `public=True` as the default for new watchlists — but as a *deliberate* decision paired with concrete guardrails, not an inherited one. If those guardrails aren't shipped, the default should flip to private.

**Reasoning (what user behavior I'm optimizing for):** CineLog is framed as a film-tracking *network*, not a private journal. The product's value compounds when users can see what the people they follow are planning to watch: public watchlists power friend-based discovery ("3 people you follow added *Dune*"), organic recommendations, and the low-friction social loop that makes a network sticky. A public default means the social graph populates itself — a new user's saved films contribute to discovery immediately, without them hunting for a "make public" toggle they may not know exists. It also fits the semantic difference between the two lists: a *watchlist* is aspirational ("want to watch"), generally lower-sensitivity than the *collection*, which carries personal ratings/opinions. So I'd optimize watchlists for sharing and treat the collection more conservatively.

**Tradeoff acknowledged:** The cost is that this is the opposite of privacy-by-default. Some users reasonably expect a saved list to stay private, and a public default puts the burden on the privacy-conscious to opt *out* rather than making the safe choice automatic. The harm is asymmetric: a list that's wrongly public leaks information others may already have seen (hard to fully undo), whereas a wrongly-private list just means the user taps "share" (cheap, reversible, user-initiated). And I have to concede the sharpest version of the counterargument: watchlist contents can be genuinely sensitive — films tied to a health condition, sexuality, religion, a breakup — and "most users are happy to share" is an assumption, not a measured fact. That's the strongest case for private-by-default and I take it seriously rather than waving it off.

Because of that, my public default is only defensible if it ships with: (1) a visible visibility indicator and a one-tap per-list private toggle; (2) disclosure that watchlists are public **before the first save**, not buried in settings after the fact; and (3) a documented, revisitable decision (this note) so that if usage shows people are surprised by exposure, we flip the default to private — the low-cost, reversible direction. Absent guardrails (1) and (2), I'd concede the point and default to private.

## Comment 5 — Sort order

**My position:** Adopt the maintainer's preference. `get_watchlist()` now sorts by `date_added` descending (newest first) instead of alphabetically by title.

**Reasoning:**
1. **It matches intent for a "saved for later" list.** The films a user just added are the ones on their mind and most likely to be watched next. Alphabetical order can bury a fresh save on page 3 because its title starts with a late letter — that actively fights the user's goal.
2. **Consistency with the rest of the app (the decisive argument).** `get_collection()` already returns newest-first (`CollectionEntry.date_added.desc()`). Watchlist and collection are sibling lists in the same product; shipping two different default orders forces users to learn two mental models for no benefit. Viewed that way, the alphabetical sort wasn't a considered alternative — it was an inconsistency with a convention the codebase had already set.
3. **Alphabetical is deterministic but semantically arbitrary.** Title A→Z has no relationship to how a user prioritizes what to watch.

**Engagement with reviewer's point:** I agree with the maintainer's core claim — "most users want to see what they added recently" — and I'm reinforcing it with the consistency-with-`get_collection` argument, which the comment didn't raise but which I think is the stronger of the two: it's not merely that recency is intuitive, it's that we *already committed* to recency for collections. Where I'd nuance a pure "recency always" rule: on a very long watchlist neither recency nor alphabetical is ideal — users will eventually want to filter/sort by genre, runtime, etc. So I'm treating `date_added` desc as the correct *default*, with user-selectable sorting as sensible follow-up work, rather than claiming it's universally optimal. For the default the maintainer asked about, we're aligned, and I've implemented it.

**Incidental fix found while verifying:** Writing a test for the new sort order surfaced a latent bug — `get_watchlist()` builds each result via `entry.film.to_dict()`, but `WatchlistEntry` had no `film` relationship mapped. The old code masked this with a `.join(Film)`, but a SQL join only enables the ORDER BY; it does **not** populate an ORM `.film` attribute, so `entry.film` raised `AttributeError`. I confirmed the *original committed* `get_watchlist()` fails the same way — it had simply never been exercised by a test. I fixed it the way `CollectionEntry` is wired: added `watchlist_entries = db.relationship("WatchlistEntry", backref="film")` on the `Film` model, giving `WatchlistEntry.film` a working backref. This is committed separately from the sort change.

**How I verified:** Added `test_get_watchlist_returns_newest_first` (mirrors `test_get_collection_returns_newest_first`): two entries with explicit `date_added` values 5 days apart, asserting the later one comes first even though it sorts *second* alphabetically — so the test would fail under the old alphabetical order. `pytest tests/ -v` → 6 passed.

## Comment 6 — Rebase

**What conflicted:** I ran `git fetch origin` and `git rebase origin/main`. The refactor commit on `main` (`refactor: migrate film IDs from integer to UUID`) did three things to `models.py`: changed `Film.id` from `Integer` to `String(36)` UUID, changed `CollectionEntry.film_id` to `String(36)`, and — the subtle part — **deleted the `WatchlistEntry` model entirely** (watchlist doesn't exist on `main`; the class only lived on this feature branch).

Because none of my commits *modified* the `WatchlistEntry` block (I only added a `Film.watchlist_entries` relationship elsewhere), git had no competing change to flag there, so the rebase completed with **no textual merge conflict** — but it silently left the branch broken: `models.py` still referenced `WatchlistEntry` in a relationship while the class itself was gone, and my watchlist code (and its `film_id` docstrings/test) still assumed integer IDs. This is the real "conflict": a *semantic* one the rebase didn't surface, not a `<<<<<<<` textual one.

**How I resolved it:** I restored the `WatchlistEntry` model in `models.py` with `film_id = db.Column(db.String(36), db.ForeignKey("film.id"))` — a UUID, matching the refactored `Film.id` and `CollectionEntry.film_id`. I then updated the remaining integer assumptions: the `add_to_watchlist()` docstring (`film_id (int)` → `film_id (str): UUID`), the route docstring (`{ "film_id": <int> }` → `"<uuid>"`), and the nonexistent-film test (`99999` → a UUID string). All of this is in one commit: `fix: update WatchlistEntry film_id to UUID after main branch refactor`.

**How I verified no conflict remains:**
- `git status` reports a clean rebase (no `rebase-merge` in progress, no conflict markers); `grep -rn "<<<<<<<" .` finds nothing.
- Imports resolve and the mapper configures: `python -c "import models"` plus an end-to-end script that creates a `Film` (confirming `film.id` is now a UUID *string*), adds it to a watchlist, re-adds it (duplicate correctly raises `AlreadyInWatchlistError`), and reads the list back sorted newest-first.
- `pytest tests/ -v` → **6 passed** (4 collection + 2 watchlist), including the nonexistent-film test now driving a UUID id.
- `git log --merges origin/main..HEAD` is empty → **no merge commits**; the branch is rebased, not merged.

### Final commit history

See the `git log --oneline` screenshot above. Each commit is one logical change, all conventional (`feat`/`fix`/`test`/`docs`), with no merge commits — the branch is rebased, not merged. The original bundled first commit ("added watchlist model and endpoint fixed a bug more changes") was reworded to `feat: add watchlist model and add_to_watchlist endpoint` during an interactive rebase, and the `Co-Authored-By` trailers were stripped in the same pass.

## PR Description

### What this feature does
Adds a **watchlist** — films a user saves to watch later, separate from their collection (films already watched/logged). It introduces:
- a `WatchlistEntry` model (`user_id`, `film_id` as a UUID, `date_added`, `public`);
- service functions `add_to_watchlist()` and `get_watchlist()` in `services/watchlist_service.py`;
- REST endpoints under `/watchlist`:
  - `POST /watchlist/<user_id>/add` — body `{ "film_id": "<uuid>" }` → `201` with the new entry; `404` if the film doesn't exist; `409` if it's already on the watchlist;
  - `GET /watchlist/<user_id>` — the user's watchlist as a list of film dicts.

Adding a film that's already on the watchlist is rejected (no duplicate rows) via `AlreadyInWatchlistError`, mirroring how the collection service prevents duplicates.

### Design decisions (documented above in full)
1. **Default visibility — `public=True`, conditionally.** New watchlists default to public to power CineLog's social discovery loop, but this is defensible *only* paired with pre-first-save disclosure and a one-tap private toggle; without those, the default should be private. Full reasoning and the privacy tradeoff are in **Comment 4** above.
2. **Sort order — date added, newest first.** `get_watchlist()` sorts by `date_added` descending (not alphabetically), matching `get_collection()` so the two sibling lists behave consistently. Full reasoning is in **Comment 5** above.

### How to manually test
Run the app:
```bash
python app.py            # serves on http://localhost:5000
```
There are no public create endpoints for users/films (film data is seeded), so seed one of each in another terminal (uses the same `cinelog.db`):
```bash
python - <<'PY'
from app import create_app, db
from models import User, Film
with create_app().app_context():
    u = User(username="alice", email="alice@example.com")
    f1 = Film(title="Dune", year=2021, genre="Sci-Fi")
    f2 = Film(title="Arrival", year=2016, genre="Sci-Fi")
    db.session.add_all([u, f1, f2]); db.session.commit()
    print("USER_ID =", u.id)
    print("FILM1_ID =", f1.id)
    print("FILM2_ID =", f2.id)
PY
```
Then, substituting the printed ids:
```bash
# 1) Add a film -> 201, "public": true
curl -X POST http://localhost:5000/watchlist/USER_ID/add \
  -H "Content-Type: application/json" -d '{"film_id": "FILM1_ID"}'

# 2) Add a second film, then view the list -> Arrival (added later) appears FIRST
curl -X POST http://localhost:5000/watchlist/USER_ID/add \
  -H "Content-Type: application/json" -d '{"film_id": "FILM2_ID"}'
curl http://localhost:5000/watchlist/USER_ID

# 3) Add the same film again -> 409 "already on this user's watchlist" (no duplicate)
curl -X POST http://localhost:5000/watchlist/USER_ID/add \
  -H "Content-Type: application/json" -d '{"film_id": "FILM1_ID"}'

# 4) Add a film that doesn't exist -> 404 "No film found"
curl -X POST http://localhost:5000/watchlist/USER_ID/add \
  -H "Content-Type: application/json" -d '{"film_id": "00000000-0000-0000-0000-000000000000"}'
```
Automated tests: `pytest tests/ -v` (6 passing).
