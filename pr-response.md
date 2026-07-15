# PR Response Doc — CineLog Watchlist Feature

## Commit History
<img width="1342" height="194" alt="image" src="https://github.com/user-attachments/assets/7e8c3cd8-b121-42e4-bc2b-60795b360617" />

## AI Usage

I used an AI assistant (Claude) in a few specific, bounded ways during this project:

- **Codebase orientation.** Before reading the review comments, I had the AI summarize
  `models.py`, `services/collection_service.py`, and `tests/test_collection.py` — what each
  file is responsible for and what the existing patterns are (naming, deduplication, test
  fixtures). I verified each summary against the actual code; where I relied on a claim (e.g.
  that `add_to_collection()` raises on an existing `(user_id, film_id)` before inserting), I
  confirmed it by reading the function directly.
- **Stress-testing my design arguments (Comments 4 and 5).** After I decided my positions —
  keep `public=True`, and switch the watchlist sort to date-added, I asked the AI to play
  devil's advocate and raise the counterarguments a careful maintainer would. For Comment 4
  this surfaced a real gap: my original wording implied users could "easily toggle" visibility,
  but that opt-out mechanism isn't actually built yet (the model field exists, but nothing
  exposes it). I revised my response to acknowledge that honestly and flag the toggle as a
  follow-up, rather than overclaim. The core reasoning and the final wording are mine; the AI
  only pushed me to close the gap.
- **Rebase safety and verification.** The integer→UUID rebase had a non-obvious trap: a plain
  `git rebase main` silently dropped the `WatchlistEntry` model (main's refactor had no such
  model, and Git resolved the overlap by taking main's side with no conflict marker). Working
  through it with the AI helped me catch that the model needed to be explicitly re-added with a
  UUID `film_id`, and to verify the result with `pytest` and `git log --merges` rather than
  trusting the rebase "succeeded."
- **Commit-history hygiene.** I used the AI to sanity-check that my final commit messages
  followed conventional-commit format and that no single commit was bundling unrelated changes,
  then verified the history myself with `git log --oneline`.

What the AI did *not* do: it did not make the design decisions in Comments 4 and 5, and it did
not write the deduplication or rename logic for me, those follow the existing
`collection_service.py` patterns, which I applied myself after reading them.

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in
`services/watchlist_service.py` to match the project's `verb_to_noun` naming convention
(consistent with `add_to_collection()` in the collection service). Updated the docstring
summary from "Save a film…" to "Add a film…" to match.

**How I verified:** Ran a repo-wide search (`grep -rn "save_to_watchlist" --include="*.py"`)
to find every call site. There were exactly two, both in `routes/watchlist/watchlist.py` —
the import on line 8 and the call on line 32 — which matches what the reviewer noted. Updated
both, re-ran the search to confirm zero remaining references, then ran `pytest tests/ -v`
(all passing).

## Comment 2 — Deduplication
**What I did:** Added a duplicate check to `add_to_watchlist()`. Before creating the entry,
it now queries for an existing `WatchlistEntry` with the same `(user_id, film_id)` and raises
`AlreadyInWatchlistError` if one exists. I followed the exact pattern from
`add_to_collection()` in `services/collection_service.py`, which does the same lookup and
raises `AlreadyInCollectionError`. I added a matching `AlreadyInWatchlistError` exception
class in the watchlist service, mirroring how the collection service defines its own
exceptions.

**How I verified:** Read `add_to_collection()` to confirm the pattern: it uses
`.filter_by(user_id=..., film_id=...).first()` and raises if truthy, before the
`db.session.add`. I then reproduced the scenario directly in a Python shell, added a film,
then added the same `(user_id, film_id)` again — and confirmed the second call raises
`AlreadyInWatchlistError` and that only one row persists (`.count()` == 1). Full suite still
passes.

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py` and added
`test_add_to_watchlist_nonexistent_film_raises`, the watchlist equivalent of
`test_add_to_collection_nonexistent_film_raises` in `tests/test_collection.py`. It asserts
that calling `add_to_watchlist()` with a film_id that isn't in the database raises
`FilmNotFoundError`.

**How I verified:** Used `test_add_to_collection_nonexistent_film_raises` as the model —
same `app` / `sample_user` fixtures (in-memory SQLite), same `pytest.raises(...)` assertion
structure, same fake-id approach. Ran `pytest tests/test_watchlist.py -v` (passes) and the
full `pytest tests/ -v` (all pass).

## Comment 4 — Default visibility
**My position:** Keep the `public=True` default.

**Reasoning:** CineLog is meant to be a community-driven film tracking app, so the
social/discovery features only work if watchlists are visible by default. If the default
were private, most users would stay private (some without even realizing it), and the
community layer would just be hollow, with social features with nothing to populate them. Public
by default optimizes for frictionless discovery, while privacy stays available to anyone who
wants to make a given list private.

**Tradeoff acknowledged:** A user could expose their watchlist without intending to. I think
this is acceptable because watchlist entries are low-sensitivity (films you intend to watch,
nothing especially private), and the `public` field already exists on the model to support
per-entry privacy controls. The one honest caveat is that the mechanism to flip that field,
an actual opt-out on the endpoint, isn't built yet. The field supports it but nothing
currently exposes it to callers. I'd treat adding that visibility control as the natural
follow-up (it's the "visibility toggle" stretch item). Even so, the downside here is small
and reversible, while the upside, a functional community, is central to the product.

## Comment 5 — Sort order
**My position:** Agree with the maintainer — switch the watchlist to date-added order (newest
first).

**Reasoning:** `get_collection()` already sorts by `date_added.desc()`, and there's a test
enforcing it too (`test_get_collection_returns_newest_first`). The watchlist was the
inconsistent one, sorting alphabetically by title. Matching the collection's convention keeps
behavior predictable across the two sibling features.

**Engagement with reviewer's point:** The maintainer's "most users want to see what they
added recently" reasoning is consistent with the existing collection behavior, so this isn't
just a preference because it's aligning the watchlist with a pattern the codebase already
established. Implemented: changed `.order_by(Film.title.asc())` →
`.order_by(WatchlistEntry.date_added.desc())` in `get_watchlist()`, and added
`test_get_watchlist_returns_newest_first` so the new order is actually enforced.

## Comment 6 — Rebase
**What conflicted:** Two things. (1) `.gitignore`: `main` had gained its own `.gitignore`
(from the same refactor series), so my separate `chore: add .gitignore` commit became an
add/add conflict — and redundant. (2) `models.py`: this was the real one. The UUID refactor on
`main` changed `Film.id` and `CollectionEntry.film_id` from `Integer` to `String(36)` (UUID),
while my feature branch's first commit had authored `models.py` in the *pre-refactor* state
(integer IDs) and appended the new `WatchlistEntry` with an `Integer` `film_id`. So my branch
was effectively trying to revert main's refactor while adding the watchlist model on top of the
old schema.

**How I resolved it:**
- Dropped the redundant `.gitignore` commit (`git rebase --skip`) since `main` already provides one.
- For `models.py`, resolved to main's post-refactor version (UUIDs throughout) **plus** my
  `WatchlistEntry` model, with its `film_id` changed from `db.Integer` to `db.String(36)` so it
  matches the new UUID `Film.id`. I also updated the `add_to_watchlist()` docstring, which had
  described `film_id` as an integer, to say UUID.
- **Important gotcha I hit:** a plain `git rebase main` did *not* raise a textual conflict on
  `models.py` — Git silently took main's side, which had no `WatchlistEntry`, so the model was
  dropped entirely and the whole feature broke (`ImportError: cannot import name
  'WatchlistEntry'`). I caught this because my test suite failed to even import. The fix was to
  explicitly re-add the `WatchlistEntry` model (with UUID `film_id`) as part of resolving the
  rebase, not to trust the auto-merge. The service logic itself needed no change — it uses
  `db.session.get(Film, film_id)` and `filter_by(film_id=...)`, which are type-agnostic.

**How I verified no conflict remains:**
- `git log --merges main..HEAD` returns nothing → no merge commits; the branch is a clean
  linear rebase on top of `main`.
- `pytest tests/ -v` → all tests pass on the rebased branch.
- Ran an end-to-end check creating a `Film` (whose `id` is now a 36-char UUID) and adding it to
  a watchlist — `add_to_watchlist()` and `get_watchlist()` both work with the UUID, confirming
  the integer→UUID migration is complete in the watchlist code.

## PR Description

### What this feature does
Adds a **watchlist** to CineLog, a list of films a user wants to watch (distinct from the
collection, which is films they've already watched). The feature includes:
- A `WatchlistEntry` model (`user_id`, `film_id` as UUID, `date_added`, `public`).
- A watchlist service with `add_to_watchlist(user_id, film_id)` and `get_watchlist(user_id)`.
- REST endpoints: `POST /watchlist/<user_id>/add` (body `{ "film_id": "<uuid>" }`) and
  `GET /watchlist/<user_id>`.
- Deduplication: adding a film already on the watchlist raises `AlreadyInWatchlistError`
  instead of creating a duplicate, following the same pattern as `add_to_collection()`.

### Design decisions
1. **Default visibility (`public=True`).** Watchlists are public by default. CineLog is a
   community film-tracking app, and the social/discovery features only work if lists are
   visible by default; a private default would leave most lists (and the community layer)
   empty. The tradeoff — a user could expose a watchlist unintentionally — is acceptable
   because entries are low-sensitivity and the `public` field is designed to support a
   per-entry privacy control (adding the actual opt-out endpoint is a natural follow-up).
2. **Sort order (date-added, newest first).** `get_watchlist()` returns entries sorted by
   `date_added` descending, matching the existing `get_collection()` convention (which has a
   test enforcing it). This aligns the two sibling features and reflects that most users want
   to see what they recently added. Changed from the original alphabetical-by-title sort.

### How to manually test
1. Set up and run:
   ```bash
   python -m venv .venv && source .venv/bin/activate
   pip install -r requirements.txt
   python app.py            # serves at http://127.0.0.1:5000 (no frontend; use curl)
   ```
2. You'll need a valid user UUID and film UUID (film IDs are UUIDs after the refactor). Create
   or look them up via `flask shell` if the DB isn't pre-seeded.
3. Add a film to the watchlist:
   ```bash
   curl -s -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
     -H "Content-Type: application/json" -d '{"film_id": "<film_uuid>"}'
   ```
   Expect `201` with the new entry.
4. Fetch the watchlist:
   ```bash
   curl -s http://127.0.0.1:5000/watchlist/<user_id>
   ```
   Expect the film, with `date_added` and `public` fields; most-recently-added first.
5. Add the **same** film again → the service raises `AlreadyInWatchlistError` (no duplicate).
6. Add a film UUID that doesn't exist → `FilmNotFoundError`.
7. Run the test suite: `pytest tests/ -v` (all pass, including the watchlist tests).
