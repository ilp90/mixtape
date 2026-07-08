# Mixtape Bug Hunt — Submission

## AI Usage

I used an AI coding assistant (Claude, via Claude Code) throughout, but deliberately in a "reader and rubber-duck" role rather than a "find-and-fix-the-bug" role. The honest split:
AI was most valuable for **navigation and comprehension**, occasionally useful for
**confirming a single fact**, and something I intentionally *did not* trust for
**diagnosis** — every root cause below I confirmed by reading the code and running it
myself. Below is where it actually helped, and where it didn't.

**Codebase navigation (Milestone 1).** I had the AI summarize each service file's
responsibility and trace two call chains (rate-a-song and view-a-playlist) from route →
service → model. This built the codebase map quickly. I verified each summary against the
actual files — the model correctly identified the strict route→service delegation pattern
and the `playlist_entries.position` ordering column, both of which I confirmed by reading
`models.py` and the route files rather than taking on faith.

**Reproduction (Milestone 2).** The most important thing AI did here was help me *not*
chase a phantom bug. My first-pass plan included Issue #3 (search duplicates). Rather than
trust the "outerjoin causes duplicates" theory, I reproduced it — `GET /songs/search?q=Crown`
returned `count: 1`, and all `test_search.py` tests passed. I then confirmed *why*: on
SQLAlchemy 2.0, legacy `Query(Song).all()` auto-de-duplicates full entities by primary key,
so the fan-out never surfaces. This is a concrete case where the plausible-sounding
diagnosis was wrong and only running the code revealed it — so I swapped #3 for #1.

**Per-bug debugging (Milestone 3):**
- **#1 (streak):** I found and read the over-constrained `if` branch myself. I used AI only
  to confirm one narrow fact — that Python's `datetime.weekday()` maps Sunday to `6` (vs
  `isoweekday()`'s Sunday=7) — which pinned `!= 6` as specifically a Sunday exclusion. I
  verified the fix by re-running the streak tests, checking both the Sunday-increments and
  skipped-day-still-resets cases.
- **#5 (playlist):** No AI needed for diagnosis. The `songs[:-1]` slice is unambiguous on a
  plain read; AI added nothing a careful read didn't.
- **#4 (notification):** I used the "compare two similar blocks" pattern — I asked the AI to
  contrast `add_to_playlist` and `rate_song`. It confirmed my own reading that the trailing
  guarded `create_notification` call exists in the former and is simply absent from the
  latter. I wrote the fix to mirror the existing pattern and verified with HTTP scenarios
  (non-sharer notifies, self-rating doesn't, re-rating notifies again).

**Where AI was incomplete or would have misled me.** (1) The #3 "outerjoin duplicates"
theory is exactly the kind of plausible-but-wrong answer that surfaces when you ask AI to
diagnose before reproducing — the ORM's auto-uniquing behavior is version-specific context
the theory ignored, and only running the query exposed it. (2) AI has no view into the live
database or the seeded state, so every reproduction (tag counts, playlist entry counts,
sharer/rater IDs) I gathered by querying the DB and hitting real endpoints myself. The
workflow I stuck to was: **I find/read the code → AI helps me understand or confirm one fact
→ I verify by running it.** Fixing before verifying was never on the table.

## Milestone 1: Codebase Map

Mixtape is a Flask + SQLAlchemy social music app. Friends share songs, rate them,
build collaborative playlists, keep listening streaks, and see what friends are
listening to. It's a JSON API only (no HTML frontend) — every response is JSON, and
there is no root `/` route, so `GET /` returns 404. The live endpoints all sit under
the four blueprint prefixes (`/songs`, `/playlists`, `/users`, `/feed`).

The architecture is a clean three-layer split:

```
HTTP request → routes/ (parse + format)  → services/ (business logic) → models.py (DB)
```

### Main files and their responsibilities

**`app.py`** — Application factory. `create_app()` builds the Flask app, configures the
SQLite database (`sqlite:///mixtape.db`), initializes `db = SQLAlchemy()`, registers the
four blueprints under their URL prefixes, and calls `db.create_all()`. It also holds the
shared `db` object that every other module imports. This is why the app must be started
with `FLASK_APP=app:create_app flask run` and **not** `python app.py` — the latter
imports `app` twice (once as `__main__`, once as `app`) and SQLAlchemy raises a
double-registration error on the models.

**`models.py`** — Defines the data model. Five entity models plus three association tables:

- **Entities:** `User`, `Song`, `ListeningEvent`, `Rating`, `Playlist`, `Notification`, `Tag`.
- **Association tables:**
  - `friendships` — many-to-many, symmetric-ish user↔user relationship. `User.friends`
    is a `lazy="dynamic"` self-referential relationship.
  - `song_tags` — many-to-many songs↔tags.
  - `playlist_entries` — many-to-many playlist↔song **with extra columns**: `position`
    (explicit ordering — songs have a defined position, not just insertion order),
    `added_by`, and `added_at`. This is the analog of a join table with payload.
- Notable: there is **no separate "rating notification" wiring** — ratings live in the
  `Rating` table (unique per `(user_id, song_id)`), and streak state (`listening_streak`,
  `last_listened_at`) lives directly on `User`. Every model has a `to_dict()` used for
  JSON serialization.

**`routes/`** — Thin HTTP layer. Each file is a blueprint that parses request args/JSON,
calls exactly one service function, and formats the JSON response (including turning a
service `ValueError` into a 400/404). No business logic lives here.

- `routes/songs.py` — `/songs/search`, `/songs/<id>`, `/songs/<id>/rate` (POST),
  `/songs/<id>/listen` (POST).
- `routes/playlists.py` — create playlist, get metadata, get songs, add song (POST).
- `routes/users.py` — get user, get streak, list notifications, mark notification read.
- `routes/feed.py` — friends-listening-now feed, activity feed.

**`services/`** — Where all business logic (and all five bugs) live:

- `streak_service.py` — records listening events and updates the consecutive-day streak
  (`update_listening_streak`). Streak increments on consecutive calendar days, resets if a
  day is skipped.
- `feed_service.py` — `get_friends_listening_now` (friends active within a 24h cutoff,
  deduped to one most-recent song per friend) and `get_activity_feed` (most recent N events).
- `search_service.py` — `search_songs` matches title/artist with `ILIKE`, outer-joining
  `song_tags`; `get_song` fetches one by ID.
- `notification_service.py` — `create_notification` helper, `add_to_playlist` (adds song +
  notifies the sharer), `rate_song` (upserts a rating), plus `get_notifications` /
  `mark_as_read`.
- `playlist_service.py` — `create_playlist`, `get_playlist_songs` (ordered by `position`),
  `get_playlist`, `get_user_playlists`.

**`seed_data.py`** — Populates the DB with test users (e.g. `nova`), songs, friendships,
listening events, playlists, and ratings.

**`tests/`** — `test_streaks.py`, `test_search.py`, `test_playlists.py` (pytest).

### Data flow trace — user rates a song (notification feature)

1. Client sends `POST /songs/<song_id>/rate` with JSON `{"user_id", "score"}`.
2. `routes/songs.py::rate()` parses the body, validates both fields are present, then calls
   `notification_service.rate_song(user_id, song_id, int(score))`.
3. `rate_song()` validates the score is 1–5, loads the `Song` and rating `User` (raising
   `ValueError` → 400 if either is missing), then **upserts**: if a `Rating` already exists
   for that `(user_id, song_id)` it updates the score in place; otherwise it inserts a new
   `Rating`. It commits and returns the `Rating`.
4. The route serializes `rating.to_dict()` and returns `201`.

Compare with the **add-song-to-playlist** flow, which *does* produce a notification:
`POST /playlists/<id>/songs` → `add_to_playlist()` appends the song to the playlist and, if
the adder isn't the original sharer, calls `create_notification(..., "song_added_to_playlist", ...)`
for `song.shared_by`. So the app already knows how to notify a sharer — the pattern exists
in `add_to_playlist` but the equivalent notify step is what Issue #4 is about for rating.

### Patterns I noticed

- **Strict route→service delegation.** Every route does only parse + call + format.
  Business logic and DB writes live entirely in `services/`. Services signal errors by
  raising `ValueError`; routes translate those into HTTP 400/404. This is why the README
  says "if an endpoint is broken, trace it back to the service it calls."
- **Shared `db` singleton.** `db` is defined once in `app.py` and imported everywhere;
  models and services never create their own SQLAlchemy instance.
- **`to_dict()` everywhere.** Serialization is a model responsibility, not a route/service one.
- **Ordering is explicit, not implicit.** `playlist_entries.position` means playlist song
  order is stored, and `get_playlist_songs` sorts by it — relevant to Issue #5.
- **Time is UTC + calendar-day based.** Streaks and the feed cutoff both use
  `datetime.now(timezone.utc)`; streaks compare `.date()` values (calendar days), the feed
  uses a rolling 24h `timedelta`.

## The Five Open Issues (all read)

| # | Symptom | Service | Initial suspicion (to confirm in later milestones) |
|---|---------|---------|-----------------------------------------------------|
| 1 | Listening streak keeps resetting | `streak_service.py` | `update_listening_streak` has a `today.weekday() != 6` (Sunday) special-case that wrongly resets the streak on Sundays. |
| 2 | Friends Listening Now shows people from yesterday | `feed_service.py` | 24h `timedelta` cutoff behaves like "yesterday too" rather than a true calendar/short window — needs a closer look. |
| 3 | Same song shows up twice in search | `search_service.py` | `outerjoin(song_tags)` fans out one row per tag; a song with 2+ tags is returned multiple times (no `DISTINCT`/grouping). |
| 4 | Notified when friend adds my song to a playlist, but not when they rate it | `notification_service.py` | `rate_song` never calls `create_notification` — the notify step present in `add_to_playlist` is missing here. |
| 5 | Last song in a playlist never shows up | `playlist_service.py` | `get_playlist_songs` returns `songs[:-1]`, dropping the final element. |

### Rough plan — which three to fix

Leaning toward **#3 (search dedup)**, **#4 (rating notification)**, and **#5 (playlist
off-by-one)** first: each has a clear, well-isolated root cause and existing test files
(`test_search.py`, `test_playlists.py`) to validate against. **#1 (streak Sunday bug)** is a
strong backup — the `weekday() != 6` clause is an obvious planted defect with
`test_streaks.py` coverage. **#2 (feed recency)** needs the most investigation into intended
semantics, so it's lowest priority. Final choice of three to be confirmed after diagnosing
each in the next milestone.

---

## Milestone 2: Reproduce the Chosen Bugs

**Chosen three: #1 (streak), #4 (rating notification), #5 (playlist last song).**

> **Plan change — dropped #3, picked up #1.** I could not reproduce #3 (search
> duplicates). The `search_songs` query does an `outerjoin` on `song_tags`, which *should*
> fan out one row per tag — but SQLAlchemy 2.0's legacy `Query(Song).all()` automatically
> de-duplicates full-entity results by primary key, so the fan-out collapses back to one
> row per song. Verified two ways: (a) `GET /songs/search?q=Crown` for "Crown Heights
> Anthem" (3 tags) returns `count: 1`, not 3; (b) all five tests in `test_search.py` pass,
> including `test_search_no_duplicates_multi_tag_song` (whose comment says "bug causes it to
> be 3"). The defect is latent — real, but masked by ORM uniquing on this version — so it's
> not reproducible as reported. Per the milestone's guidance to switch bugs when one can't
> be reproduced, I replaced it with #1, which reproduces cleanly.

Environment note: reproductions were run against the running app (`FLASK_APP=app:create_app
flask run`, hitting `http://127.0.0.1:5000`) and against the pytest suite. Seed IDs are
regenerated on each `python seed_data.py`, so the commands below fetch live IDs rather than
hardcoding them. **No code was changed in this milestone.**

### Bug #1 — Listening streak resets on Sunday (`streak_service.py`)

- **Condition needed:** two listens on *consecutive calendar days* where the second day is a
  Sunday (`weekday() == 6`).
- **How I reproduced it:** ran `pytest tests/test_streaks.py`. `test_streak_increments_on_sunday`
  listens on Sat 2024-06-15 then Sun 2024-06-16 and expects the streak to go 1 → 2. It fails
  with `assert 1 == 2` — the streak reset instead of incrementing. The other four streak
  tests (new user, consecutive weekday, same-day, skipped-day) all pass, which isolates the
  trigger to the Sunday case specifically.
- **Observed vs expected:** observed streak = 1 (reset); expected = 2 (increment).
- **Root cause (confirmed by reproduction path):** the increment branch is guarded by
  `days_since_last == 1 and today.weekday() != 6`, so a legitimate consecutive-day listen
  falls through to the `else: streak = 1` reset whenever "today" is a Sunday.

### Bug #4 — No notification when a friend rates your shared song (`notification_service.py`)

- **Condition needed:** a user who is *not* the song's original sharer submits a rating.
- **How I reproduced it (HTTP):**
  1. `GET /users/<sharer_id>/notifications` → baseline `count = 1` (one pre-existing
     `song_added_to_playlist` notification from the seed data).
  2. `POST /songs/<song_id>/rate` with `{"user_id": <non-sharer_id>, "score": 5}` → returns
     `201` and the rating is saved (confirmed in the JSON response).
  3. `GET /users/<sharer_id>/notifications` again → still `count = 1`, still only the
     `song_added_to_playlist` type. No `song_rated` notification was created.
- **Observed vs expected:** observed = no new notification for the sharer; expected = the
  sharer gets notified that their song was rated.
- **Contrast that pinpoints it:** the *add-to-playlist* path (`add_to_playlist`) does create a
  notification for the sharer — that's the `song_added_to_playlist` entry we see. The
  *rating* path (`rate_song`) saves the `Rating` and commits but never calls
  `create_notification`. The notify step that exists in one sibling function is simply
  missing from the other.

### Bug #5 — Last song in a playlist never shows up (`playlist_service.py`)

- **Condition needed:** any non-empty playlist.
- **How I reproduced it (two ways):**
  - **HTTP:** playlist "Late Night Vibes" has **7** entries in the DB (verified by counting
    `playlist_entries` rows). `GET /playlists/<id>/songs` returns `count: 6` — the 7th
    (last-by-position) song is dropped.
  - **Tests:** `pytest tests/test_playlists.py` — `test_playlist_returns_all_songs` fails
    (5-song playlist returns 4) and `test_playlist_returns_songs_in_order` fails (list is
    missing "Track 5"). `test_empty_playlist_returns_empty_list` passes, so the bug only bites
    non-empty playlists.
- **Observed vs expected:** observed = N-1 songs; expected = all N.
- **Root cause (confirmed by reproduction path):** `get_playlist_songs` returns
  `[song.to_dict() for song in songs[:-1]]` — the `[:-1]` slice discards the final element.
  (On an empty list `[][:-1]` is still `[]`, which is why the empty-playlist test passes.)

---

## Milestone 3: Root Cause Analyses

### RCA #1 — "My listening streak keeps resetting" (`streak_service.py`)

**How I reproduced it.** Ran `pytest tests/test_streaks.py`.
`test_streak_increments_on_sunday` listens on Sat 2024-06-15 then Sun 2024-06-16 and
expects the streak to go 1 → 2; it failed with `assert 1 == 2`. The other four streak
tests (new user, consecutive weekday, same-day double-listen, skipped-day reset) all
passed. That contrast is the whole diagnosis in miniature: consecutive-day logic works on
every weekday *except* when the second day is a Sunday.

**How I found the root cause.** Started from the reproduction: the failing test calls
`update_listening_streak` directly, so I opened
[services/streak_service.py](services/streak_service.py) and read that function
top-down. The branch that decides increment-vs-reset is:

```python
elif days_since_last == 1 and today.weekday() != 6:
    user.listening_streak += 1
else:
    user.listening_streak = 1
```

The moment of confidence: `days_since_last == 1` already *fully* describes "listened
yesterday, listening today" — a consecutive day. The extra `and today.weekday() != 6`
serves no purpose in a calendar-day streak. Since Python's `datetime.weekday()` returns
`6` for Sunday, any consecutive-day listen that lands on a Sunday fails the `and`, falls
through to `else`, and resets to 1. That exactly matches "streak keeps resetting" — it
resets every Sunday even when the user never missed a day.

**The root cause.** The increment condition was over-constrained. A consecutive-day
listen is correctly identified by `days_since_last == 1` alone. The additional clause
`today.weekday() != 6` (where `weekday() == 6` means Sunday) excludes Sundays from
incrementing, so a user who listens Saturday and again Sunday is wrongly treated as having
broken their streak and is reset to 1. There is no calendar reason a streak should reset
on the seventh day of the week — a streak is about *consecutive days*, not *weeks*.

**AI usage.** I located and read the branch myself, then used AI to confirm one narrow
fact I wanted to be certain of: that Python's `datetime.weekday()` maps Monday=0 … Sunday=6
(as opposed to `isoweekday()`'s Monday=1 … Sunday=7). That confirmed `!= 6` is specifically
a Sunday exclusion. I verified the fix by reading and re-running the tests, not by trusting
the explanation.

**Fix and side-effect check.** Removed the `and today.weekday() != 6` clause so the branch
is simply `elif days_since_last == 1:`. This is the smallest change that addresses the root
cause — one boolean condition, no restructuring. Boundary check (both sides): re-ran all of
`test_streaks.py` — 5/5 pass. `test_streak_increments_on_sunday` now passes (Sat→Sun gives
2), and critically `test_streak_resets_after_skipped_day` still passes (Mon→Wed still resets
to 1), so removing the clause did not weaken the genuine reset-on-gap behavior. Same-day
(`days_since_last == 0`) and new-user (`last_listened_at is None`) paths are untouched.

### RCA #5 — "The last song in a playlist never shows up" (`playlist_service.py`)

**How I reproduced it.** Two independent ways. (1) HTTP: the seeded playlist "Late Night
Vibes" has 7 rows in `playlist_entries` (confirmed by counting the association table), but
`GET /playlists/<id>/songs` returned `count: 6` — the last-by-position song was missing.
(2) Tests: `pytest tests/test_playlists.py` — `test_playlist_returns_all_songs` failed
(5-song playlist returned 4) and `test_playlist_returns_songs_in_order` failed (the returned
titles were missing "Track 5"). `test_empty_playlist_returns_empty_list` passed, telling me
the bug only affects non-empty playlists.

**How I found the root cause.** The symptom ("last song missing", off by exactly one at the
end) pointed straight at the retrieval function. I opened
[services/playlist_service.py](services/playlist_service.py) and read
`get_playlist_songs`. The query itself is correct — it joins `playlist_entries`, filters by
`playlist_id`, and orders ascending by `position`. The last line is where it goes wrong:

```python
return [song.to_dict() for song in songs[:-1]]
```

The `[:-1]` slice was the "aha" moment — it's not a query bug or an ordering bug, it's a
Python slice that deliberately drops the final element after a correctly-ordered fetch.
Because the query orders by `position` ascending, "the final element" is always the
last song in the playlist, which is exactly the reported symptom.

**The root cause.** `get_playlist_songs` fetched all N songs in correct position order, then
serialized `songs[:-1]` — a slice that returns every element *except the last*. So every
non-empty playlist returned N-1 songs with the highest-position (last) song silently
dropped. On an empty playlist the slice is harmless (`[][:-1] == []`), which is why only
non-empty playlists exhibited the bug.

**AI usage.** None needed for diagnosis — the `[:-1]` slice is unambiguous on a plain read.
(General Python knowledge that `list[:-1]` excludes the last element; no AI query made.)

**Fix and side-effect check.** Changed `songs[:-1]` to `songs`, so all fetched songs are
serialized. Smallest possible fix — one slice removed, query and ordering untouched.
Boundary check (both sides): re-ran `test_playlists.py` — 3/3 pass. Full playlist now
returns all 5 in order (`test_playlist_returns_all_songs`, `..._in_order`), and the empty
playlist still returns `[]` without error (`test_empty_playlist_returns_empty_list`), so the
fix doesn't introduce an index/None error at the lower boundary.

### RCA #4 — "Notified when a friend added my song to a playlist but not when they rated it" (`notification_service.py`)

**How I reproduced it.** HTTP sequence: (1) `GET /users/<sharer_id>/notifications` →
baseline `count: 1` (a pre-existing `song_added_to_playlist` notification from the seed).
(2) `POST /songs/<song_id>/rate` with `{"user_id": <non-sharer_id>, "score": 5}` → returned
`201`, and the `Rating` was persisted (confirmed in the JSON body). (3) `GET
/users/<sharer_id>/notifications` again → still `count: 1`, still only
`song_added_to_playlist`. The rating succeeded but the sharer was never notified.

**How I found the root cause.** The issue title itself names the comparison: one
interaction (add-to-playlist) notifies, a sibling interaction (rate) doesn't — and both
live in [services/notification_service.py](services/notification_service.py). I read the
two functions side by side. `add_to_playlist` ends with a guarded notify step:

```python
if song.shared_by != added_by_user_id:
    create_notification(user_id=song.shared_by,
                        notification_type="song_added_to_playlist", body=...)
```

`rate_song`, by contrast, validated the score, upserted the `Rating`, committed, and
returned — with **no** `create_notification` call anywhere. That structural diff was the
moment of confidence: the notify step isn't broken, it's simply absent from `rate_song`.
The routes and models were never the issue — `routes/songs.py` correctly calls `rate_song`,
and the `Notification` model and `get_notifications` retrieval both already work (proven by
the `song_added_to_playlist` notification showing up).

**The root cause.** `rate_song` created/updated the `Rating` and committed but never
generated a notification for the song's sharer. The application's notification pattern —
"when someone interacts with your shared song, notify you" — was implemented for the
add-to-playlist path but omitted from the rating path. It was a missing step, not a faulty
one.

**AI usage.** I used AI in the "compare two similar blocks" mode suggested in the brief:
I pasted `add_to_playlist` and `rate_song` and asked what the structural difference was. It
confirmed my own read — `add_to_playlist` has a trailing `create_notification` guarded by a
self-interaction check and `rate_song` has none. I then wrote the fix to mirror the existing
pattern rather than invent a new one, and verified behavior by HTTP, not by trusting the AI.

**Fix and side-effect check.** After the existing `db.session.commit()` in `rate_song`, I
added a notify step mirroring `add_to_playlist`: if `song.shared_by != user_id`, call
`create_notification(user_id=song.shared_by, notification_type="song_rated", body=f"{rater.username} rated your song '{song.title}' {score}/5.")`.
The `rater` and `song` objects were already loaded earlier in the function, so no extra
queries. The `!= user_id` guard reuses the same self-interaction pattern so a user rating
their own song isn't notified. Side effects verified by HTTP: (A) a non-sharer rating
creates exactly one `song_rated` notification for the sharer; (B) a sharer rating their
**own** song creates none (guard works) and the rating still returns `201`; (C) a re-rating
by a non-sharer notifies again via the update path. Ran the full suite — 13/13 pass, so the
added notification did not disturb search, streak, or playlist behavior.

---
*Milestone 1 checkpoint: `bugfix/mixtape` branch created ✓ · app runs at
`http://127.0.0.1:5000` ✓ · codebase map written ✓ · all five issues read ✓*

*Milestone 2 checkpoint: all three chosen bugs (#1, #4, #5) deliberately triggered and
their inputs/conditions documented ✓ · #3 dropped as not-reproducible (ORM auto-dedup) and
replaced with #1 ✓ · no code changed yet ✓*
