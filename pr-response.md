# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Review and edit this to match your own honest account. -->
I used AI tools (Claude Code) in the following ways during this project:

- **Codebase orientation:** summarizing what `models.py`, `services/collection_service.py`,
  and `tests/test_collection.py` do, and how `add_to_collection()` handles film lookup and
  deduplication — so I understood the existing patterns before touching the watchlist code.
- **Implementation support:** applying the rename, mirroring the collection deduplication
  pattern into `add_to_watchlist()`, and writing the watchlist test in the same fixture
  style as the collection tests. I verified every change by running `pytest`.
- **Rebase guidance:** working through the UUID rebase conflict step by step (see Comment 6).
- **Stress-testing my design arguments (Comments 4 & 5):** after I decided my positions,
  I asked the AI what counterarguments a reviewer might raise. The *decisions* — private-by-default
  visibility and date-added sort order — are my own, grounded in how CineLog's collection and
  watchlist differ. Where the AI surfaced a counterpoint I hadn't addressed (e.g. that
  private-by-default weakens the community discovery the app is built around), I folded the
  tradeoff into my written response rather than letting the argument stay one-sided.

---

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in
`services/watchlist_service.py` to match the project's `verb_to_noun` convention
(`add_to_collection()`), and updated the one call site plus the import in
`routes/watchlist/watchlist.py`.
**How I verified:** Ran a project-wide search (`grep -rn "save_to_watchlist" --include="*.py"`)
to confirm no references remained, then ran the full test suite — 4 tests still passed.

## Comment 2 — Deduplication
**What I did:** Added a duplicate-entry check to `add_to_watchlist()`, following the exact
pattern in `add_to_collection()`: query for an existing `WatchlistEntry` with the same
`user_id`/`film_id` and raise a new `AlreadyInWatchlistError` if one exists, before creating
the entry. I added the `AlreadyInWatchlistError` class at the top of the service module,
mirroring how `AlreadyInCollectionError` is defined in `collection_service.py`.
**How I verified:** Read `add_to_collection()` to confirm the dedup happens *after* the
film-existence check and *before* the insert, and reproduced that ordering. Confirmed the
module imports cleanly and the full suite still passed.

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py` with
`test_add_to_watchlist_nonexistent_film_raises`, the direct equivalent of
`test_add_to_collection_nonexistent_film_raises`. It reuses the same fixture structure
(`app` with an in-memory SQLite DB, `sample_user`) and the same assertion style
(`pytest.raises(FilmNotFoundError)` for a nonexistent film id).
**How I verified:** `pytest tests/test_watchlist.py -v` passes; the full suite is now 5 passing.

## Comment 4 — Default visibility
**My position:** `WatchlistEntry.public` should default to **`False` (private)**, not `True`.

**Reasoning:** CineLog models two different lists. `CollectionEntry` records films a user has
*already watched* — and notably it has no `public` field at all. `WatchlistEntry` is the one
that introduced visibility, and it represents *intent* — films a user is planning to watch.
Intent is arguably more sensitive than history: a watchlist signals what someone is about to
do and what they're currently interested in, which can be personal (a run of films tied to a
health topic, a breakup, a job search). Defaulting that to public means the very first time a
user saves a film — before they've formed any mental model of CineLog's social features — it's
silently broadcast. That violates the principle of least surprise. A community product earns
trust by making sharing an intentional act, so users should have to *opt in* to publishing a
watchlist rather than opt out.

**Tradeoff acknowledged:** CineLog is explicitly a *community* film-tracking app, and
public-by-default maximizes the shared content that fuels discovery, recommendations, and
network effects — with zero friction. Private-by-default means far fewer public watchlists,
because most users never flip a toggle, so the community/discovery surface grows more slowly.
That's a real cost. I accept it because an accidental privacy exposure is far more damaging to
a community's trust than slower growth of public content, and the discovery value can be
recovered with good opt-in UX (a visible "share your watchlist" affordance produces
*intentionally curated* public lists, which are higher-signal for discovery than lists that
are public only because nobody changed the default).

**Implementation:** I shipped this rather than leaving it as an argument. Flipping the default
to `False` on its own would make every watchlist permanently private (nothing could ever be
shared), so I paired it with an explicit `public` opt-in parameter on `add_to_watchlist()` and
the `/watchlist/<user_id>/add` endpoint (the visibility-toggle stretch feature). Private is now
the safe default *and* sharing stays possible via `{"public": true}`. See the Stretch Features
section below.

## Comment 5 — Sort order
**My position:** Change `get_watchlist()` to sort by **`date_added` descending (newest first)**,
matching `get_collection()`. (Implemented.)

**Reasoning:** Two things point the same way. First, recency is the more useful default for a
watchlist specifically — a watchlist is a queue of intent, and the film a user just added is
the one most on their mind, so surfacing recent additions supports the core "I just heard about
this, saved it, what's on my list" loop. Alphabetical order optimizes for a *lookup* task ("is
X already saved?"), which is secondary and better served by search/filter than by the default
sort. Second, the previous alphabetical ordering sorted by `Film.title`, so the list was
organized around film metadata rather than around anything the user actually did.

**Engagement with reviewer's point:** The maintainer is right on both counts — consistency with
`get_collection()` matters, and date-added is the better default. `get_collection()` already
returns newest-first (`date_added.desc()`), and that behavior is locked by
`test_get_collection_returns_newest_first`. Having two sibling list views order differently
(collection newest-first, watchlist A–Z) makes the product feel incoherent. I adopted
`date_added.desc()` specifically (not ascending) so the two views are *truly* consistent, not
just "both sorted by date." I considered oldest-first as a "watch these first" queue but
rejected it to preserve exact consistency with the collection and because newest-first matches
the moment-of-adding mental model. If lookup later becomes a common need, the right answer is
an optional sort parameter, not changing this default.

## Comment 6 — Rebase
**What conflicted:** My `feature/watchlist` branch was based on the pre-refactor commit, where
film IDs were integers. `main` had since migrated film IDs from `Integer` to UUID
`String(36)`. Rebasing onto `main` produced two issues:
1. **`models.py`** — Git's 3-way auto-merge resolved the file to `main`'s version and, in doing
   so, *silently dropped* my `WatchlistEntry` class (it was an addition that lost out to main's
   copy of the surrounding file — no conflict markers, but the model was gone).
2. **`.gitignore`** — an add/add conflict, because both my branch and `main` had added one.

**How I resolved it:**
- `.gitignore`: `git rebase --skip` on my now-redundant `.gitignore` commit, since `main`
  already provides an equivalent one. (The course PDF is kept out of git locally via
  `.git/info/exclude`.)
- `models.py`: re-added the `WatchlistEntry` model with `film_id` as `String(36)` UUID
  (matching `CollectionEntry`) instead of the pre-refactor `Integer`, and updated the stale
  `film_id (int)` docstrings in `watchlist_service.py` and the route. Committed as
  `fix: update WatchlistEntry film_id to UUID after main branch refactor`.

**How I verified no conflict remains:** `git status` is clean; `git log --oneline` shows a
linear history rebased on top of `main` with **no merge commit**; the app boots and registers
all three blueprints; `pytest tests/ -v` passes all 5 tests; and a `grep` confirmed no
remaining integer `film_id` references in the watchlist code.

---

## Stretch Features

**Visibility toggle (`public` parameter):** Added an optional `public` parameter to
`add_to_watchlist(user_id, film_id, public=False)` and the `POST /watchlist/<user_id>/add`
endpoint (`{"film_id": "<uuid>", "public": true}`). This lets callers set visibility
explicitly instead of relying on the default, and is what makes the private-by-default
decision (Comment 4) workable — private is the safe default, sharing is an explicit opt-in.

**Second test (edge case of my choice):** Beyond the required nonexistent-film test, I added
two tests in `tests/test_watchlist.py`:
- `test_add_to_watchlist_defaults_to_private` — verifies a new entry is `public is False` when
  the caller doesn't specify, locking in the Comment 4 default. I chose this case because the
  default is a *silent* behavior no other test would catch — a future change flipping it back
  to public would otherwise pass CI unnoticed.
- `test_add_to_watchlist_respects_public_flag` — verifies `public=True` actually produces a
  public entry, so the opt-in toggle can't silently no-op.

---

## Follow-up: Automated Review (Copilot)

After opening the PR, GitHub Copilot's review flagged three issues beyond the six human
comments. All three were valid and are addressed:

1. **`WatchlistEntry` had no `film`/`user` relationship** — `get_watchlist()` calls
   `entry.film`, so `GET /watchlist/<user_id>` crashed with `AttributeError` (my tests missed
   it because none exercised `get_watchlist()`). Added `watchlist_entries` relationships on
   `User` and `Film` (mirroring `CollectionEntry`), plus a `UniqueConstraint(user_id, film_id)`
   for DB-level dedup safety under concurrency. Verified `GET /watchlist` now returns 200.
2. **Route returned 500 for invalid requests** — the endpoint didn't catch `FilmNotFoundError`
   or `AlreadyInWatchlistError`. Added the same `try/except` the collection route uses, so a
   nonexistent film returns 404 and a duplicate returns 409.
3. **Missing test coverage** — added `test_add_to_watchlist_duplicate_raises` and
   `test_get_watchlist_returns_newest_first` (mirroring `test_collection.py`). The sort test
   also guards against a regression of issue #1. Suite is now 9 passing tests.

---

## Git Log Screenshot

`git log --oneline` on `feature/watchlist` after the interactive rebase — 8 conventional
commits, one logical change each, rebased on `main` with no merge commits:

![git log --oneline showing 8 conventional commits with no merge commits](git-log.png)

---

## PR Description

**What the feature does:** Adds a **watchlist** — a list of films a user intends to watch,
separate from their `collection` (films already watched).

- **Model** `WatchlistEntry` (`models.py`): UUID primary key, `user_id`, `film_id` (UUID),
  `date_added`, and a `public` visibility flag.
- **Service** (`services/watchlist_service.py`):
  - `add_to_watchlist(user_id, film_id)` — validates the film exists (`FilmNotFoundError`),
    prevents duplicates (`AlreadyInWatchlistError`), then creates the entry.
  - `get_watchlist(user_id)` — returns the user's watchlist sorted by date added, newest first.
- **Endpoints** (`routes/watchlist/watchlist.py`, registered at `/watchlist`):
  - `GET  /watchlist/<user_id>` — list the watchlist.
  - `POST /watchlist/<user_id>/add` — body `{ "film_id": "<uuid>" }`.

**Design decisions made:**
1. **Default visibility → private (`public=False`)** — sharing should be an explicit opt-in
   (see Comment 4).
2. **Sort order → date-added, newest first** — consistent with `get_collection()` and better
   matched to how a watchlist is used (see Comment 5).

**How to manually test the feature:**
```bash
# 1. Start the app
python app.py            # serves at http://127.0.0.1:5000

# 2. Seed a user and a film, and grab their UUIDs (in a second terminal)
python - <<'PY'
from app import create_app, db
from models import User, Film
app = create_app()
with app.app_context():
    u = User(username="ashok", email="ashok@example.com")
    f = Film(title="Dune: Part Two", year=2024, genre="Sci-Fi")
    db.session.add_all([u, f]); db.session.commit()
    print("USER_ID =", u.id)
    print("FILM_ID =", f.id)
PY

# 3. Add the film to the watchlist (use the printed IDs)
curl -X POST http://127.0.0.1:5000/watchlist/<USER_ID>/add \
     -H "Content-Type: application/json" \
     -d '{"film_id": "<FILM_ID>"}'
# → 201 with the new entry (note "public": false)

# 4. Adding the same film again is rejected (deduplication)
curl -X POST http://127.0.0.1:5000/watchlist/<USER_ID>/add \
     -H "Content-Type: application/json" \
     -d '{"film_id": "<FILM_ID>"}'

# 5. View the watchlist (newest first)
curl http://127.0.0.1:5000/watchlist/<USER_ID>
```
