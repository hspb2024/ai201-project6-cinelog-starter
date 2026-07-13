# PR Response Doc — CineLog Watchlist Feature

Branch: `feature/watchlist` (rebased on `main`) · Responding to the six review comments
from @dev-lead (the maintainer persona; comments appear under the repo owner's account on
PR #1 of the upstream repo).

**Summary:** Addressed all six comments — renamed the service function to match the
naming convention, added deduplication, added the missing test, documented reasoned
positions on default visibility and sort order, and rebased onto `main` to adopt the
integer→UUID film-ID refactor. Full suite: **8 passing**. History is linear with no merge
commits.

## AI Usage

I used **Claude Code** throughout, and verified everything against the actual code.

- **Orientation:** I had it summarize `models.py`, `collection_service.py`, and
  `watchlist_service.py`, and trace the route → service → model call chains before I read
  the PR comments. I confirmed each summary by reading the source.
- **Understanding the dedup pattern (Comment 2):** I asked it to walk through what
  `add_to_collection()`'s duplicate check does and what it returns on a hit, then wrote my
  own `add_to_watchlist` guard rather than have it generate the code.
- **Stress-testing my design arguments (Comments 4 & 5):** After drafting both positions,
  I asked it "what counterargument would a careful reviewer raise, and what tradeoff am I
  not acknowledging?"
  - For **Comment 4** it raised privacy-by-default / least-astonishment / data-minimization
    as the standard counter to a public default. I had gestured at privacy but hadn't made
    the mitigation concrete — so I revised to explicitly pair the public default with the
    existing per-entry toggle and an account-level "default to private" fast-follow, and to
    state I'd flip the default if user research showed surprise.
  - For **Comment 5** it noted alphabetical is genuinely better for *looking up a known
    title* in a long list. I hadn't named that tradeoff, so I added it and explained why
    lookup is the secondary use case (better served by an explicit sort control later).
  The final arguments are my own; AI only surfaced gaps to close.
- **Verification, not just generation:** while verifying the sort change I found (with a
  quick REPL check the AI helped me script) that `get_watchlist()` raised `AttributeError`
  because `WatchlistEntry` had no `Film` relationship — a real bug I then fixed.
- **Commit-format check:** I gave my `git log --oneline` to the AI and asked whether the
  messages follow the conventional-commits spec and whether any bundled multiple logical
  changes, then verified against the spec myself.

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in
`services/watchlist_service.py` to match the project's `verb_to_noun` convention (same
shape as `add_to_collection()`), and updated both call sites in
`routes/watchlist/watchlist.py` (the import and the call in `add_film()`).

**How I verified:** Ran a project-wide search (`grep -rn "save_to_watchlist"
--include="*.py" .`) before and after — three references before (definition + import +
call), zero after. `pytest tests/ -v` still passes, confirming the import chain is intact.

## Comment 2 — Deduplication
**What I did:** Added a duplicate guard to `add_to_watchlist()`, following
`add_to_collection()` exactly: after the film-exists check it queries for an existing
`WatchlistEntry` with the same `(user_id, film_id)` and raises a new
`AlreadyInWatchlistError` instead of inserting a second row. I defined
`AlreadyInWatchlistError` locally in `watchlist_service.py`, mirroring how
`collection_service.py` defines `AlreadyInCollectionError`. (The model has no DB-level
unique constraint on the watchlist, so this service-level check is what enforces
uniqueness.) I also wired the route to return **409** on this error, matching the
collection route.

**How I verified:** Studied `add_to_collection()`'s check first, then confirmed mine with
an isolated in-memory script — adding the same `(user, film)` twice raised
`AlreadyInWatchlistError` and the entry count stayed **1**. Covered by
`test_add_to_watchlist_duplicate_raises`.

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py`, modeled on `tests/test_collection.py`
(same `app` / `sample_user` / `sample_film` fixtures). Added
`test_add_to_watchlist_nonexistent_film_raises` as the direct equivalent of
`test_add_to_collection_nonexistent_film_raises`, plus basic-add, duplicate, and
date-added-ordering tests.

**How I verified:** `pytest tests/test_watchlist.py -v` → 4 passed; `pytest tests/ -v` →
**8 passed** (4 existing collection + 4 new).

## Comment 4 — Default visibility
**My position:** Keep the default at `public=True`, as a deliberate product decision. (No
code change; the reasoning is recorded here and in the PR description, which is what the
comment asked for.)

**Reasoning:** CineLog is a *social* film-tracking network — its value loop is discovery:
seeing what friends are watching and planning to watch fuels recommendations and
"let's watch this together." I'm optimizing for **frictionless social discovery**: a new
user who saves a film contributes to that network effect without first hunting for a
toggle. It's also consistent with the app's existing social framing (the collection
feature has no privacy flag at all).

**Tradeoff acknowledged:** The cost is **privacy / principle of least astonishment**. Some
users treat a watchlist as a private wishlist and would be surprised — or exposed — to
find their "want to watch" intent public. Privacy-by-default (`public=False`) is the
safer, more standard posture (data minimization, GDPR-friendly, harder to regret). I'm
accepting that cost because CineLog is explicitly a social network, not a private journal
— and mitigating it: the `public` flag already exists **per entry** so users can mark
individual films private today, and I'd recommend a fast-follow account-level "default new
lists to private" setting plus a clear visibility indicator at add-time. If user research
showed surprise or complaints, I'd flip the default — the decision is intentional, not
permanent.

## Comment 5 — Sort order
**My position:** I agree with the maintainer — changed the default from alphabetical to
**date added, newest first** (`get_watchlist()` now uses
`order_by(WatchlistEntry.date_added.desc())`).

**Reasoning:** A watchlist is a *recency-oriented queue* — it answers "what should I watch
next / what did I just add," not "where's the film starting with M." Newest-first puts the
most top-of-mind items where the user looks first.

**Engagement with reviewer's point:** The maintainer's argument ("most users want to see
what they added recently") is right, and there's a reinforcing reason: `get_collection()`
**already** sorts by `date_added.desc()`, so the alphabetical watchlist was an
*inconsistency* between two sibling "my films" surfaces. Date-added makes it both more
user-aligned and consistent. The tradeoff I'm accepting: alphabetical is better for
*looking up a specific known title* in a long list — but that's the secondary use case and
is better served by an explicit sort/filter control later than by degrading recency
scanning for everyone. (While implementing this I also fixed a latent bug: `get_watchlist`
accessed `entry.film` but `WatchlistEntry` had no `Film` relationship, so it raised
`AttributeError` on any non-empty list. I added `Film.watchlist_entries`, mirroring
`Film.collection_entries`.)

## Comment 6 — Rebase
**What conflicted:** `main` had migrated `Film.id` and `CollectionEntry.film_id` from
integer to UUID (`db.String(36)`). My branch was written against the pre-refactor integer
IDs: `WatchlistEntry.film_id` was `db.Integer`, and the service/route docstrings described
`film_id` as an int. So the conflict was in `models.py` (my integer column/docstring vs.
main's UUID column/docstring) plus the now-stale `int` references in
`services/watchlist_service.py` and `routes/watchlist/watchlist.py`.

**How I resolved it:** I ran `git rebase origin/main`. The text-level auto-merge was
misleading — it kept main's UUID definitions but silently *dropped the `WatchlistEntry`
class* while leaving the `Film.watchlist_entries` relationship pointing at a now-missing
model (a semantic conflict a line-based merge won't catch). To resolve it correctly and
end with a clean linear history, I rebuilt the branch on top of `origin/main` so every
commit is UUID-correct: `WatchlistEntry.film_id` is now `db.String(36)` with a
`ForeignKey("film.id")`, the `Film.watchlist_entries` relationship is intact, and the
`int` references in the service/route are updated to UUID/str.

**How I verified no conflict remains:** `git log --oneline` shows a linear history with
**no merge commits**; `git log --merges origin/main..HEAD` is empty. A search for
`Integer`/`int` in the watchlist code returns nothing; all of `Film.id`,
`CollectionEntry.film_id`, and `WatchlistEntry.film_id` are `String(36)`. The app imports
and `db.create_all()` succeeds under UUID, and the full suite (**8 tests**) passes,
including the nonexistent-film test that uses a UUID-style id.

## Commit History

Linear on top of `main`, conventional format, one logical change per commit, no merge
commits:

```
fix:  return 404/409 from watchlist add route for not-found and duplicate
test: add watchlist tests including nonexistent film and sort order
fix:  sort watchlist by date added instead of alphabetically
fix:  add deduplication check to prevent duplicate watchlist entries
fix:  rename save_to_watchlist to add_to_watchlist per naming convention
feat: add watchlist model, service, and endpoint
```
(plus this `docs:` commit adding the PR response doc — newest first as shown by
`git log --oneline`.)

## PR Description

### What the watchlist feature does
Lets a user save films they want to watch later and view that list.

- `GET /watchlist/<user_id>` — returns the user's watchlist as a list of film dicts (with
  `date_added` and `public`), **sorted newest-first**.
- `POST /watchlist/<user_id>/add` with JSON `{ "film_id": "<uuid>" }` — adds a film; returns
  the created entry (**201**). Returns **404** if the film doesn't exist and **409** if the
  film is already on the watchlist (deduplicated).

Backed by the `WatchlistEntry` model (UUID `film_id`, `date_added`, `public`) and a
`Film.watchlist_entries` relationship.

### Design decisions
1. **Default visibility = `public=True`** — optimizing for CineLog's social-discovery loop.
   Tradeoff: weaker privacy-by-default; mitigated by the per-entry `public` flag and a
   recommended account-level default-private setting. (See Comment 4.)
2. **Sort order = date added, newest first** — a watchlist is a recency queue, and this
   matches `get_collection()`. Tradeoff: alphabetical is better for lookup in long lists,
   deferred to a future explicit sort control. (See Comment 5.)

### How to manually test
```bash
# 1. Create a user and a film, and print their UUIDs
python -c "
from app import create_app, db
from models import User, Film
app = create_app()
with app.app_context():
    u = User(username='ada', email='ada@example.com')
    f = Film(title='Metropolis', year=1927)
    db.session.add_all([u, f]); db.session.commit()
    print('USER', u.id); print('FILM', f.id)
"

# 2. Run the app
python app.py   # serves http://127.0.0.1:5000

# 3. Add the film to the watchlist  -> 201 + the created entry
curl -X POST http://127.0.0.1:5000/watchlist/<USER_ID>/add \
     -H "Content-Type: application/json" -d '{"film_id": "<FILM_ID>"}'

# 4. View the watchlist  -> [ { ...film..., "date_added": ..., "public": true } ]
curl http://127.0.0.1:5000/watchlist/<USER_ID>

# 5. Add the SAME film again  -> 409 "already on this user's watchlist"
curl -X POST http://127.0.0.1:5000/watchlist/<USER_ID>/add \
     -H "Content-Type: application/json" -d '{"film_id": "<FILM_ID>"}'

# 6. Add a nonexistent film  -> 404 "No film found with id ..."
curl -X POST http://127.0.0.1:5000/watchlist/<USER_ID>/add \
     -H "Content-Type: application/json" -d '{"film_id": "00000000-0000-0000-0000-000000000000"}'
```
Automated: `pytest tests/ -v` (8 passing).
