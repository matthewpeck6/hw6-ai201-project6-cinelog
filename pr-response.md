# PR Response Doc — CineLog Watchlist Feature

## AI Usage

I used Claude to curtail my pr-response document and revise and edit what i wrote

## Comment 1 — Rename

**What I did:**
Renamed `save\_to\_watchlist()` to `add\_to\_watchlist()` in `services/watchlist\_service.py` to match the project's `verb\_to\_noun` naming convention already used by `add\_to\_collection()`. Updated the one call site in `routes/watchlist/watchlist.py` (the `add\_film` route) to use the new name.

While updating the call site, I also noticed the route had no exception handling at all — not even for the existing `FilmNotFoundError`, unlike `collection.py`'s equivalent route. I added `try/except` handling there so `FilmNotFoundError` and `AlreadyInWatchlistError` (see Comment 2) return proper 404/409 responses instead of an unhandled 500.

**How I verified:**
Searched the codebase for all references to `save\_to\_watchlist` to confirm only one call site existed. Manual trace of both files to confirm names/imports line up. *(Note: full `pytest tests/ -v` run still needs to happen locally to confirm nothing broke.)*

## Comment 2 — Deduplication

**What I did:**
Added a dedup check to `add\_to\_watchlist()` mirroring the pattern in `add\_to\_collection()`: query for an existing `WatchlistEntry` with the same `user\_id`/`film\_id` before inserting, and raise an error if one exists. Defined a new `AlreadyInWatchlistError` exception locally in `watchlist\_service.py` rather than reusing `AlreadyInCollectionError` from `collection\_service.py`, since that error's message is collection-specific — this follows the same local-exception pattern `collection\_service.py` itself uses for `NotInCollectionError`.

**How I verified:**
Manual trace against `add\_to\_collection()`'s logic. Also wrote `test\_add\_to\_watchlist\_duplicate\_raises` (see Comment 3) to exercise this path directly. *(Still needs a local `pytest` run to confirm.)*

## Comment 3 — Missing test

**What I did:**
Created `tests/test\_watchlist.py`, following the fixture and assertion structure of `tests/test\_collection.py`. Included the specifically requested test, `test\_add\_to\_watchlist\_nonexistent\_film\_raises`, modeled directly on `test\_add\_to\_collection\_nonexistent\_film\_raises`. Also added `test\_add\_to\_watchlist\_creates\_entry` and `test\_add\_to\_watchlist\_duplicate\_raises` while in the file, though only the nonexistent-film case was explicitly requested.

**How I verified:**
Modeled directly on the existing `test\_collection.py` fixtures (`app`, `sample\_user`, `sample\_film`) and assertion style. *(Local `pytest tests/test\_watchlist.py -v` run still needed to confirm all three pass.)*

## Comment 4 — Default visibility

**My position:** Watchlists should default to `public=False`.

**Reasoning:** I value protecting user confidentiality by default. A watchlist's primary purpose is for a user to document what they personally want to watch — not to be a public-facing artifact. Right now, there is no toggle anywhere in the code for a user to see or change the `public` setting, and nothing in the codebase currently reads or filters on `WatchlistEntry.public` at all — the field is inert. Defaulting it to `True` under those conditions means user data is silently exposed with no way for the user to know about it or control it, which contradicts the expectation a user should reasonably have about their own data.

**Tradeoff acknowledged:** I initially leaned toward `public=True` for social-discovery reasons (helping a group of users see what's popular and decide what to watch together), but reconsidered after checking the codebase — there's no follower/social model in `models.py` and no endpoint that aggregates or surfaces other users' watchlists, so that use case doesn't exist yet. I'm accepting that, under a `False` default, a group of friends deciding what to watch together would have no shared/ranked view to work from until such a feature is built. I think that's the right tradeoff given silent exposure is a real, present cost, while the social feature is still hypothetical.

## Comment 5 — Sort order

**My position:
Reasoning:
Engagement with reviewer's point:**

<I argued for date-added via a "group decides what's popular" framing,
     but this self-contradicted (prioritizing old neglected entries vs. surfacing what's new) and didn't
     yet engage with the maintainer's "most users want recency" claim or the get\_collection() precedent
     (which already sorts date\_added.desc()), or name the alphabetical tradeoff (findability of a

## Comment 6 — Rebase

**What conflicted:
How I resolved it:
How I verified no conflict remains:**

< git rebase origin/main initially failed because .gitignore and pr-response.md
     were untracked, which blocked the checkout. Committed both separately:
       chore: add .gitignore for venv, cache, and db files
       docs: add PR response doc
     Retrying the rebase now. Expecting the real conflict to land in models.py, since main's refactor
     changed Film.id from db.Integer to db.String(36) (matching the UUID pattern already used for
     User.id and CollectionEntry.id), and downstream WatchlistEntry.film\_id / CollectionEntry.film\_id
     (currently db.Integer FKs) will need updating to db.String(36) to match, along with a stale

## PR Description

<!-- Written at the end — feature overview, design decisions, manual testing steps -->

