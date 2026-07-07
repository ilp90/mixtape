# Mixtape Bug Hunt — Submission

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
*Milestone 1 checkpoint: `bugfix/mixtape` branch created ✓ · app runs at
`http://127.0.0.1:5000` ✓ · codebase map written ✓ · all five issues read ✓*
