# Mixtape Bug Hunt Submission

## AI Usage

I used Claude Code, Anthropic's coding agent, in my terminal for every milestone of this project. I wrote the instructions for each milestone and set the rules it had to follow, and Claude Code did most of the hands-on work inside those rules: reading files, running the app and scripts, writing the fixes, and drafting this document. Where the root cause entries below say "I", that includes work Claude Code carried out at my direction. Each entry ends with a short note on what it did for that bug.

### What I directed and decided

- I set the order of work and the ground rules for each milestone. The rules were: no app code changes while reproducing, prove a hypothesis by running code before fixing anything, make the smallest fix, one bug per commit, stop after each bug and wait for me, and never push without asking.
- I moved from one bug to the next only after reading the report for the previous one.
- For Issue #2, the code alone did not say how long "listening now" should last. Claude Code laid out two options with the evidence for each, a 30 minute rolling window or a "since midnight" cutoff, and I chose the 30 minute window.
- I chose to stop after four fixes and move to final review, so Issue #5 is reproduced but not fixed.
- [KWESI: add any other decisions you made or edits you made yourself, for example changes you made to the write-ups or to Claude Code's proposed fixes.]

### Codebase navigation

Claude Code read the README, `app.py`, `models.py`, every file in `routes/` and `services/`, and `seed_data.py`. It then drafted the codebase map: what each file is responsible for, the models and association tables with their notable columns, which route file calls which service file, and two full data flows traced through real function names. Before that, it audited my setup. It checked that `origin` pointed to my fork, that the virtual environment had the required packages, that the database was seeded, and that the `bugfix/mixtape` branch existed. It also started the app with `flask run` and confirmed five endpoints returned 200.

### Debugging

For each bug, Claude Code traced the request from the route into the service and the model, named a suspect, and then tested that suspect by running code before changing anything. It kept all of these throwaway scripts in a `scratch/` folder listed in `.git/info/exclude`, so none of them were committed. Some specific things it did:

- `freezegun` was not installed. To control time for Issues #1 and #2, it swapped in a frozen `datetime` inside the service module instead of adding a new dependency.
- For Issue #1, it called `update_listening_streak` for all seven consecutive weekday pairs to show that only Saturday to Sunday reset the streak.
- For Issue #2, it tested a competing timezone explanation and ruled it out by showing the cutoff fell exactly 24 hours back.
- For Issue #3, it printed the SQL the query generated and the raw joined rows.
- For Issue #4, it counted rows in the whole notification table to tell "never created" apart from "created but not found".

It also explained Python and SQLAlchemy behavior along the way:
- `weekday()` returns 6 for Sunday.
- SQLite hands back datetimes without a timezone.
- The legacy `query()` API collapses repeated rows for a single model, while `select().scalars()` does not.

### Where things had to be checked, and where the AI was wrong or incomplete

- **Adding a song to a playlist.** The first draft of the map described this as writing a row into `playlist_entries`. On a closer read, Claude Code suspected that insert would fail, because the required `position` and `added_by` columns are never set. It raised this with me in chat and kept it out of the map. The suspicion was only confirmed in Milestone 2, when `POST /playlists/<id>/songs` returned HTTP 500 with `NOT NULL constraint failed: playlist_entries.position`. During final review the map wording was changed so it no longer claimed the row gets written.
- **Song count.** The docstring in `seed_data.py` says it creates 25 songs. Running the seed showed 13, and the map uses the real number.
- **The Issue #2 window.** My own milestone instructions mentioned that "today's early-morning listens still show", which pointed toward a calendar day cutoff. The comments in `seed_data.py` pointed toward about 30 minutes. Claude Code asked me rather than picking one.
- **The Issue #3 explanation.** The point about the legacy query collapsing duplicate rows was first written from memory. It was only checked during final review, by running the same join both ways (one row versus three).
- **Claims about what I had reviewed.** Claude Code drafted sentences such as "I reviewed the summary against the source files" and "I reviewed the evidence and the diff before committing". It removed them because it had no way of knowing whether I had done those things.
- **A bundled commit.** Claude Code first committed the Milestone 2 reproduction notes for all five issues together with the Issue #1 fix, because they were still uncommitted when that fix went in. It caught this during final review. With my approval, it rewrote the branch history so the notes have their own `docs:` commit. It checked that every file ended up identical to before the rewrite.
- **Re-rating in Issue #4.** Claude Code chose to send a new notification every time a user re-rates a song, and flagged that choice to me. [KWESI: say whether you agree with notifying on every re-rate or would have preferred only the first rating.]
- **Unused import in Issue #3.** The `song_tags` import in `search_service.py` is now unused. It was left in on purpose to keep that commit limited to the query change.

### What I verified myself

[KWESI: add what you verified yourself here. For example: whether you read each diff before saying "next", whether you re-ran any reproduction scripts, tests or endpoints yourself, whether you checked the codebase map against the source files, and anything you corrected.]

## Codebase Map

Mixtape is a small Flask JSON API for a social music app. Users share songs, rate them, log listens, build collaborative playlists, and see what their friends are playing. The code is split into three layers: an app factory that wires everything together, a models file that defines the database schema, and a pair of folders (`routes/` and `services/`) that separate HTTP handling from business logic. There are no templates or front end; every endpoint returns JSON.

### app.py

`app.py` creates the shared `db = SQLAlchemy()` object at module level and defines `create_app(config=None)`, the application factory. The factory reads `DATABASE_URL` (defaulting to `sqlite:///mixtape.db`, which Flask-SQLAlchemy places in the `instance/` folder) and `SECRET_KEY` from the environment, applies any override dictionary passed in (which is how tests can swap in their own config), and calls `db.init_app(app)`. It then imports and registers four blueprints with URL prefixes: `songs_bp` at `/songs`, `playlists_bp` at `/playlists`, `users_bp` at `/users`, and `feed_bp` at `/feed`. Finally it runs `db.create_all()` inside an app context so tables exist on startup. The blueprint imports happen inside the factory so that `models.py` and the services, which all import `db` from `app`, can load without a circular import at module import time. The app is launched with `FLASK_APP=app:create_app flask run`.

### models.py

`models.py` defines every table. All primary keys are 36 character UUID strings produced by `generate_uuid()`, and timestamps default to `datetime.now(timezone.utc)`.

There are three association tables built with `db.Table`:

- `friendships` links `user_id` to `friend_id`, both pointing at `user.id`. It is a self-referential many-to-many, and the seed script inserts a row in each direction so that friendship is symmetric.
- `song_tags` links `song_id` to `tag_id`.
- `playlist_entries` links `playlist_id` to `song_id` and also carries extra columns: `position` (integer, required, used for ordering), `added_by` (a user id, required), and `added_at`.

There are six model classes:

- `User` has `username`, `email`, `listening_streak` (integer), `last_listened_at`, and `created_at`. Its relationships are `shared_songs`, `ratings`, `listening_events`, `notifications`, `playlists`, and `friends`. The `friends` relationship goes through the `friendships` table with explicit `primaryjoin` and `secondaryjoin` conditions and uses `lazy="dynamic"`, so it returns a query object rather than a list.
- `Tag` has a unique `name`.
- `Song` has `title`, `artist`, `album`, `genre`, `shared_by` (foreign key to the user who shared it), `shared_at`, and `share_note`. It relates to `ratings`, `listening_events`, and `tags` (through `song_tags`, loaded with `lazy="subquery"`). Its `to_dict()` flattens tags into a list of tag names.
- `ListeningEvent` records one listen with `user_id`, `song_id`, and `listened_at`.
- `Rating` stores a `score` (1 to 5) with `user_id`, `song_id`, and `rated_at`. A `UniqueConstraint` named `unique_user_song_rating` allows only one rating per user per song.
- `Playlist` has `name`, `created_by`, `created_at`, and `is_collaborative`, plus a `songs` relationship through `playlist_entries`.
- `Notification` has `user_id` (the recipient), `notification_type` (a short string such as `song_added_to_playlist`), `body`, `created_at`, and a `read` flag.

Each model that is returned over the API has a `to_dict()` method, and that is the only serialization mechanism in the app.

### routes/

Each file in `routes/` defines one Flask `Blueprint`. Route functions are thin: they pull values from the URL, query string, or JSON body, check that required fields are present (returning 400 if not), call one service function, and turn the result into a JSON response. Services signal "not found" or invalid input by raising `ValueError`, and every route wraps its service call in `try/except ValueError`, mapping it to 404 for lookups or 400 for writes.

- `routes/songs.py` (`/songs`): `GET /search?q=` calls `search_songs`, `GET /<song_id>` calls `get_song`, `POST /<song_id>/rate` calls `rate_song`, and `POST /<song_id>/listen` calls `record_listening_event`.
- `routes/playlists.py` (`/playlists`): `POST /` calls `create_playlist`, `GET /<playlist_id>` calls `get_playlist`, `GET /<playlist_id>/songs` calls `get_playlist_songs`, and `POST /<playlist_id>/songs` calls `add_to_playlist`.
- `routes/users.py` (`/users`): `GET /<user_id>` is the one route that queries the database directly with `db.session.get(User, ...)` instead of going through a service. `GET /<user_id>/streak` calls `get_streak`, `GET /<user_id>/notifications` calls `get_notifications` (with an optional `unread_only=true` flag), and `POST /notifications/<notification_id>/read` calls `mark_as_read`.
- `routes/feed.py` (`/feed`): `GET /<user_id>/listening-now` calls `get_friends_listening_now`, and `GET /<user_id>/activity` calls `get_activity_feed`.

### services/

The services hold all of the query and write logic. Each one imports `db` from `app` and the models it needs, and returns either a model instance or plain dicts built with `to_dict()`.

- `streak_service.py`: `record_listening_event(user_id, song_id)` creates a `ListeningEvent` stamped with the current UTC time, calls `update_listening_streak(user, now)` to adjust `listening_streak` and `last_listened_at` by comparing calendar dates, and commits. `get_streak(user_id)` returns the stored streak.
- `feed_service.py`: `get_friends_listening_now(user_id)` collects the user's friend ids, queries `ListeningEvent` rows from those friends newer than a cutoff built from the module constant `RECENT_THRESHOLD`, orders them newest first, and keeps only the first event per friend. `get_activity_feed(user_id, limit=20)` returns the latest events from friends with no time cutoff. Both build result dicts with `friend`, `song`, and `listened_at` keys.
- `search_service.py`: `search_songs(query)` uses a SQLAlchemy 2.0 style `select(Song)` with a case-insensitive `ilike` match on title or artist, executed through `db.session.execute(...).scalars().all()`. Each result's tags come from the `Song.tags` relationship when it is serialized. `get_song(song_id)` fetches a single song.
- `notification_service.py`: `create_notification(user_id, notification_type, body)` is the generic helper that inserts and commits a `Notification`. `add_to_playlist(playlist_id, song_id, added_by_user_id)` validates the song, user, and playlist, appends the song to `playlist.songs`, and then calls `create_notification` for the song's original sharer when the adder is someone else. `rate_song(user_id, song_id, score)` validates the score range and the song and user, then updates an existing `Rating` or creates a new one, commits, and calls `create_notification` with type `song_rated` for the song's sharer when the rater is someone else. `get_notifications` and `mark_as_read` handle reading and acknowledging notifications.
- `playlist_service.py`: `create_playlist` inserts a new playlist after checking the creator exists. `get_playlist_songs(playlist_id)` joins `Song` to `playlist_entries`, filters by playlist, orders by `position` ascending, and serializes the songs. `get_playlist` returns playlist metadata, and `get_user_playlists` lists playlists by creator (it is defined in the service but not currently wired to a route).

### How routes map to services

| Route file | Service files it calls |
|---|---|
| `routes/songs.py` | `search_service.py`, `notification_service.py` (rating), `streak_service.py` (listening) |
| `routes/playlists.py` | `playlist_service.py`, `notification_service.py` (adding a song) |
| `routes/users.py` | `streak_service.py`, `notification_service.py` |
| `routes/feed.py` | `feed_service.py` |

The mapping is not one to one. Some actions are grouped by their side effect rather than by the resource in the URL: rating a song and adding a song to a playlist both live in `notification_service.py` because those are the interactions that can produce a notification for the song's sharer. Likewise, logging a listen lives in `streak_service.py` because it drives the streak.

### Data flow: rating a song

1. A client sends `POST /songs/<song_id>/rate` with a JSON body like `{"user_id": "...", "score": 4}`.
2. Flask dispatches to `rate()` in `routes/songs.py`. It reads `user_id` and `score` from `request.get_json()` and returns 400 if either is missing.
3. `rate()` calls `rate_song(user_id, song_id, int(score))` from `services/notification_service.py`.
4. `rate_song` raises `ValueError` if the score is outside 1 to 5, then loads the `Song` and the rating `User` with `db.session.get`, raising `ValueError` if either is missing.
5. It queries `Rating` with `filter_by(user_id=..., song_id=...)`. If a rating exists, it overwrites `score`; otherwise it creates a new `Rating` and adds it to the session. This upsert pattern lines up with the `unique_user_song_rating` constraint on the model.
6. `db.session.commit()` persists the change.
7. If `song.shared_by` differs from the rater, it calls `create_notification` with type `song_rated` and a message naming the rater, the song and the score. The `Rating` instance is then returned.
8. Back in the route, `rating.to_dict()` is returned as JSON with status 201. Any `ValueError` from the service becomes a 400 with an `error` message.

### Data flow: adding a song to a playlist

1. A client sends `POST /playlists/<playlist_id>/songs` with `{"song_id": "...", "added_by": "..."}`.
2. `add_song()` in `routes/playlists.py` checks both fields and calls `add_to_playlist(playlist_id, song_id, added_by)` from `services/notification_service.py`.
3. `add_to_playlist` loads the `Song`, the adding `User`, and the `Playlist` (importing `Playlist` locally inside the function), raising `ValueError` for any that are missing.
4. If the song is not already in `playlist.songs`, it appends it through that relationship, which is backed by the `playlist_entries` table, and commits.
5. If `song.shared_by` differs from the adder, it calls `create_notification` with type `song_added_to_playlist` and a message naming the adder, song, and playlist. That helper adds and commits the `Notification`.
6. The route returns `{"message": "Song added to playlist"}` with status 201.

Reading the playlist back follows `GET /playlists/<playlist_id>/songs` to `get_songs()` in `routes/playlists.py` to `get_playlist_songs()` in `playlist_service.py`, which returns serialized songs ordered by `playlist_entries.position`, and the route wraps them as `{"songs": [...], "count": n}`.

### seed_data.py

`seed()` builds an app with `create_app()`, then drops and recreates every table, so running it always resets the database. It creates five users (nova, darius, simone, kenji, aaliya) with preset streak values, five bidirectional friendships (nova is friends with darius, simone, and kenji; darius with simone; kenji with aaliya), and ten tags. It adds thirteen songs in three groups: three with no tags shared by nova, five with one tag shared by darius, and five with three tags each shared by simone. It then inserts listening events (a few from the last half hour and several spread across earlier hours and days), sets `last_listened_at` for nova, darius, and kenji, creates three playlists ("Late Night Vibes", "Friday Energy", "Study Mode") with seven songs each at positions 1 through 7, and adds one sample `song_added_to_playlist` notification for nova. Association rows are inserted directly with `table.insert().values(...)` rather than through ORM relationships.

### tests/

`tests/` contains pytest files for three of the services: `test_streaks.py`, `test_search.py`, and `test_playlists.py`. They are run with `pytest tests/`.

### Patterns in how the app is organized

- **Application factory plus blueprints.** Everything is assembled in `create_app`, and each resource area is a separate blueprint with its own URL prefix.
- **Thin routes, fat services.** Routes only parse input, validate presence of fields, and shape responses. All database work happens in `services/`, and the README suggests reading the code along the same path: start at the endpoint and trace into the service it calls.
- **Exceptions as the error contract.** Services raise `ValueError` for missing records or bad input, and routes translate that into 400 or 404 JSON errors. There are no custom exception classes.
- **Serialization on the model.** Each model owns a `to_dict()` method, and services usually return lists of these dicts rather than ORM objects.
- **Side effects grouped by service.** Notification-producing actions sit together in `notification_service.py`, and streak-affecting listens sit in `streak_service.py`, even though they are reached through song and playlist URLs.
- **Mixed query styles.** Most services use the legacy `db.session.query(...)` API and `db.session.get(...)` for primary key lookups, while `search_service.py` uses the SQLAlchemy 2.0 `select()` style.
- **String UUID keys and UTC timestamps** are used consistently across all tables.

---

## Root Cause Analysis

All five issues were reproduced starting from a fresh `python seed_data.py`, with the app running through `FLASK_APP=app:create_app flask run` at `http://127.0.0.1:5000`. Each reproduction was run twice from a fresh seed and gave the same result both times. The time-dependent issues (1 and 2) were reproduced with a throwaway script that runs inside the app context, sends requests through the app's own routes with Flask's test client, and holds "now" at a fixed instant by substituting a frozen `datetime` inside the service module for the duration of each request.

### Issue 1: Listening streak resets on Sunday

**Issue #1: My listening streak keeps resetting (status: fixed)**

**How I reproduced it**

I used simone, who starts the seed with a streak of 0 and no previous listen. With the clock held at 18:00 UTC on each day from Monday 2026-10-05 through Tuesday 2026-10-13, I sent one `POST /songs/<Golden Hour id>/listen` with `{"user_id": "<simone id>"}` per day and then called `GET /users/<simone id>/streak`. Every listen returned 201. The streak climbed 1, 2, 3, 4, 5, 6 from Monday through Saturday as expected. On Sunday the response was `{"streak": 1}` instead of 7, even though she had listened the day before. The count then started over from Sunday, giving 2 on Monday and 3 on Tuesday where 8 and 9 were expected. After the script finished, the running server returned the same `{"streak": 3}` for simone, so the stored value matched. The triggering condition is a listen on a Sunday that directly follows a listen on Saturday. Listens on consecutive days anywhere from Monday through Saturday extend the streak normally.

**How I found the root cause**

I started at the two endpoints involved. `GET /users/<id>/streak` is handled by `streak()` in `routes/users.py`, which calls `get_streak()` in `services/streak_service.py`. That function only loads the `User` and returns the stored `listening_streak` column, so the read side was just reporting a value that had already been saved wrong. The value is written on the other path: `POST /songs/<id>/listen` goes to `listen()` in `routes/songs.py`, which calls `record_listening_event()`. That function creates a `ListeningEvent` stamped with the current UTC time, passes the user and that time to `update_listening_streak(user, now)`, and commits. I checked `models.py` to confirm that `User.listening_streak` is a plain integer and `last_listened_at` is a `DateTime` with no timezone, which SQLite stores as naive UTC. Inside `update_listening_streak`, the branch that increments the streak required `days_since_last == 1 and today.weekday() != 6`, and in Python `weekday()` returns 6 for Sunday. That made it my main suspect, and the existing `test_streak_increments_on_sunday` test also failed with `assert 1 == 2`. I became confident it was the cause, and not just an odd looking line, when I called `update_listening_streak` directly from a scratch script for all seven consecutive day pairs with a starting streak of 5. Six pairs produced 6 and only Saturday to Sunday produced 1. The same script showed that same day listens, skipped days, and a first ever listen were already handled correctly, so nothing else in the function was contributing.

**The root cause**

`update_listening_streak()` in `services/streak_service.py` decides between "increment" and "reset" by comparing calendar dates. It should increment whenever the previous listen was exactly one calendar day earlier. Instead the increment branch also required `today.weekday() != 6`, so whenever "today" was a Sunday the condition was false even with a one day gap, and the call fell through to the `else` branch that sets the streak to 1. Because `record_listening_event()` runs this on every listen and `get_streak()` simply returns the saved column, a user who listened every day saw their streak drop to 1 the first time they listened on a Sunday, and the count then started over from there.

**My fix**

I removed the `and today.weekday() != 6` clause, so the branch is now `elif days_since_last == 1:`. A one day gap now increments the streak on every day of the week, while the existing `days_since_last == 0` branch (no change) and the `else` branch (reset for gaps of two or more days) are untouched. This was a one line change in one file. No other code reads the weekday or depends on that clause.

**My side-effect check**

- **Original reproduction:** I re-ran it from a fresh seed. Simone's streak now goes 1 through 7 from Monday to Sunday and continues to 8 and 9 on Monday and Tuesday.
- **Both sides of the boundary, calling the service directly:**
  - Every consecutive day pair, Saturday to Sunday included, now increments from 5 to 6.
  - Saturday 23:59 to Sunday 00:01 increments.
  - A naive Saturday timestamp, the way SQLite returns it, followed by a Sunday listen increments.
  - Streaks that should reset still reset: Friday to Sunday with Saturday skipped, Saturday to Monday with Sunday skipped, and Sunday to Tuesday all go to 1.
  - A second listen on the same Sunday leaves the streak unchanged.
  - A first ever listen on a Sunday starts the streak at 1.
- **Tests:** all five tests in `tests/test_streaks.py` pass now, including `test_streak_increments_on_sunday`, which failed before. The only remaining failures in the suite belong to the search and playlist issues, which I have not touched.
- **Live app, real clock:**
  - nova had already listened today, and `POST /songs/<id>/listen` left her streak at 7.
  - darius last listened yesterday and went from 3 to 4.
  - aaliya has no `last_listened_at` and was set to 1.
  - `GET /users/<id>` returned 200, and `GET /feed/<id>/activity` returned 200 with the new listening events.
  - An unknown user still gets 400 from the listen endpoint and 404 from the streak endpoint.

**How AI helped**

Claude Code traced the call chain from both routes into `streak_service.py`, wrote the scratch script that called `update_listening_streak` for every weekday pair to confirm the Sunday condition was the only cause, and ran the before and after checks.

### Issue 2: Friends Listening Now shows people from yesterday

**Issue #2: Friends Listening Now shows people from yesterday (status: fixed)**

**How I reproduced it**

The seeded listening events are stamped from the real clock, so to get a clean picture I created a separate viewer and five friends in the throwaway script, with symmetric friendship rows and exactly one listening event per friend. With "now" held at Wednesday 2026-10-14 08:30 UTC, the friends had last listened 5 minutes ago, 3 hours ago, at 21:00 the previous evening (11.5 hours ago), at 09:30 the previous morning (23 hours ago), and at 07:30 the previous morning (25 hours ago). `GET /feed/<viewer id>/listening-now` returned a count of 4. It included the friend whose last listen was 21:00 the night before and even the one from 09:30 the previous morning, and it left out only the 25 hour old listen. The expected result for a "listening now" feed is just the friends who listened in the last few minutes. The triggering condition is any friend whose most recent listen falls anywhere inside the previous 24 hours, so a friend who stopped listening last night still shows up the next morning.

**How I found the root cause**

`GET /feed/<user_id>/listening-now` is handled by `listening_now()` in `routes/feed.py`. That route only calls `get_friends_listening_now(user_id)` in `services/feed_service.py` and wraps the returned list with a count, so the filtering had to happen in the service. In the service I followed each step:
- It loads the `User`.
- It computes `cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD`.
- It collects friend ids through the `User.friends` relationship, which goes through the `friendships` association table in `models.py`.
- It queries `ListeningEvent` rows from those friends with `listened_at >= cutoff`, newest first.
- It keeps the first event per friend.

`RECENT_THRESHOLD` is a module level constant set to `timedelta(hours=24)`, which was my main suspect. A second possibility was a timezone mismatch, because the cutoff carries a UTC timezone while `ListeningEvent.listened_at` comes back from SQLite without one.

To tell the two apart, I called `get_friends_listening_now` directly from a scratch script against an in-memory database, with the clock frozen at Wednesday 08:30 UTC and one friend per listen age. Friends who listened 1 minute, 31 minutes, 2 hours, 11.5 hours and 23 hours 58 minutes earlier were all returned, and only the friend at 24 hours 1 minute was left out. The script also printed the stored text (`'2026-10-14 08:29:00.000000'`) and the cutoff (`2026-10-13 08:30 UTC`), and the boundary fell exactly 24 hours back. That ruled out the timezone idea, since a mismatch would have shifted the boundary. The moment I was confident was when I set only `RECENT_THRESHOLD` to 30 minutes in memory, with no file edits, and reran the same script. Only the friends from the last 29 minutes came back. That showed the constant alone decided who appeared.

**The root cause**

`get_friends_listening_now()` in `services/feed_service.py` builds its cutoff from the module constant `RECENT_THRESHOLD`, which was set to `timedelta(hours=24)`. The function is meant to show friends who are listening now, and the seed script describes this as listens from about the past 30 minutes, with events two hours old or more excluded. Instead, any friend whose latest `ListeningEvent` fell anywhere in the previous 24 hours passed the `listened_at >= cutoff` filter. A friend who last played a song at 21:00 was still within the window at 08:30 the next morning, so they appeared in "Friends Listening Now" as if they were currently listening.

**My fix**

I changed `RECENT_THRESHOLD` from `timedelta(hours=24)` to `timedelta(minutes=30)`. This narrows the window to match what "listening now" means. I chose 30 minutes over a "since midnight" cutoff because a calendar day boundary would still show someone who listened at 00:30 when it is 23:00. It would also hide someone who started a song at 23:55 when it is 00:05. The query, the per friend deduplication, and `get_activity_feed()`, which deliberately has no time filter, are unchanged. It was a one line change in one file.

**My side-effect check**

- **Original reproduction:** I re-ran it from a fresh seed. At a frozen 08:30, only the friend from 5 minutes earlier is shown. The friends from 3 hours ago, from 21:00 the night before, from 09:30 the previous morning and from 25 hours ago are all hidden.
- **Exact boundary:** a listen 29 minutes old is shown and one 31 minutes old is hidden.
- **Around midnight, with the clock at 00:10:**
  - A friend who listened at 00:05 today is shown, so early morning listens still appear.
  - A friend who listened at 23:55 the night before is shown, because that is only 15 minutes ago and they are still effectively listening.
  - A friend whose last listen was 23:30 the night before is hidden.
- **Other feed rules:**
  - A friend with two recent listens and one five hours old appears once, with the newest song.
  - A user who is not a friend is never shown.
  - A user with no friends gets an empty list.
  - An unknown user still raises `ValueError`, which the route turns into a 404.
- **Activity feed:** `get_activity_feed` still returned all six friend events, including the five hour old one.
- **Live app, real clock:**
  - nova's listening-now showed darius, simone and kenji, all from 10 to 20 minutes earlier.
  - darius's listening-now showed simone but no longer showed nova, whose latest listen was two hours old.
  - darius's activity feed still listed that two hour old listen.
  - `/feed/nope/listening-now` returned 404.
- **Tests:** the full suite is unchanged, with 10 passed and 3 failed. All 3 failures are the still open search and playlist issues.

**How AI helped**

Claude Code traced the route into `feed_service.py`, wrote the frozen clock scratch script that ruled out a timezone problem and showed the 24 hour constant alone decided the result, and ran the boundary, midnight and live endpoint checks. When choosing the new window, it presented the 30 minute rolling window and the "since midnight" option along with the evidence from the seed script, and I picked the 30 minute window.

### Issue 3: The same song shows up more than once in search

**Issue #3: The same song keeps showing up twice in search (status: fixed)**

**How I reproduced it**

`GET /songs/search?q=Anthem` returned a count of 3, with Crown Heights Anthem by Borough Kings listed three times. To find the pattern, I compared several searches with the number of tags each matching song has in the seed. `q=Hours` returned After Hours three times (3 tags). `q=Block` returned Block Party once (1 tag). `q=Elara` returned Still Waters once (0 tags) and Soft Landing once (1 tag). The triggering condition is how many tags a song has. A song shows up once for each of its tags, so songs with three tags appear three times, and songs with one tag or no tags appear once. The expected result is one entry per matching song.

**How I found the root cause**

`GET /songs/search?q=` is handled by `search()` in `routes/songs.py`. It checks that `q` is present, calls `search_songs(query)` in `services/search_service.py`, and returns the list with its length as `count`. Since the route never adds or copies results, the extra entries had to come from the service. `search_songs` builds `select(Song)`, adds `.outerjoin(song_tags, Song.id == song_tags.c.song_id)`, filters with a case-insensitive `ilike` on `Song.title` or `Song.artist`, and runs it with `db.session.execute(...).scalars().all()`. In `models.py`, `song_tags` is the many-to-many association table between songs and tags, and `Song.tags` is a relationship through that same table, which `Song.to_dict()` uses to build the `tags` list. That made the join my suspect, since the filter never touches a tag column and the tags in the output come from the relationship anyway. The git log also showed an earlier commit that moved this query from the legacy `db.session.query(...)` style to the 2.0 `select()` style. The legacy query quietly collapses repeated rows for the same entity, but `select(...).scalars()` does not, so any row multiplication would now reach the user. During final review I confirmed this by running the same join both ways against the seeded data: the legacy `db.session.query(Song)` version returned Crown Heights Anthem once and the `select(Song)` version returned it three times. To confirm, I ran a scratch script against the seeded database:
- It printed the emitted SQL, which was `FROM song LEFT OUTER JOIN song_tags ... WHERE lower(song.title) LIKE ... OR lower(song.artist) LIKE ...`.
- It showed the raw joined rows for "Anthem": three rows for Crown Heights Anthem, one per tag id.
- It ran the same filter without the join and got one row that still carried all three tags through the relationship.

I was confident this was the cause when I searched every one of the thirteen seeded songs by its own title. Each came back exactly max(tag count, 1) times: three times for each three-tag song and once for every song with zero or one tag. That matched the reported symptom exactly.

**The root cause**

`search_songs()` in `services/search_service.py` should return each song whose title or artist contains the query once. Instead, it LEFT OUTER JOINed the `song_tags` association table, which turns one song into one result row per tag it has (and one row with nulls if it has none). The join was not needed for anything: the filter only checks `Song.title` and `Song.artist`, and each result's tags are loaded separately through the `Song.tags` relationship in `to_dict()`. Because the 2.0 style `db.session.execute(select(Song)...).scalars().all()` returns every row without collapsing repeats of the same `Song`, a song with three tags appeared three times in the response and its `count` was inflated, while songs with zero or one tag appeared once. That is why the duplicates looked conditional.

**My fix**

I removed the `.outerjoin(song_tags, Song.id == song_tags.c.song_id)` line from the query in `search_songs()`. The query now selects from `song` alone with the same title and artist filter, so the database returns one row per matching song. The tags in each result still come from the `Song.tags` relationship, so the response shape is unchanged. I chose removing the join over adding `.distinct()` or `.unique()` because the join was the source of the extra rows and served no purpose. Deduplicating afterwards would have left the unneeded join in place and only hidden its effect. It was a one line change in one file. The `song_tags` name is still imported at the top of the file. I left that import as it is to keep the diff to the query itself.

**My side-effect check**

- **Original reproduction:** I re-ran it from a fresh seed on the live app. `q=Anthem` now returns Crown Heights Anthem once (count 1) and `q=Hours` returns After Hours once. `q=Elara` (two songs with zero and one tag) and `q=Block` are unchanged.
- **Every seeded song, searched by its own title:** each now comes back exactly once, with its complete tag list. For example, Crown Heights Anthem still shows `['rap', 'hip-hop', 'boom bap']` and the untagged songs show `[]`.
- **Matching rules:**
  - A broad query for "a" returns all 13 matching songs with no duplicates.
  - Matching is still case-insensitive (`ANTHEM`, `elara`) and still matches on artist alone (`Borough`).
  - A query that matches nothing returns an empty list.
  - A query for a tag name such as "rap" returns nothing, which is the same as before the fix, since search only covers titles and artists.
- **Related endpoints:**
  - `GET /songs/search` with no `q` still returns 400.
  - `GET /songs/<id>` still returns 200, and an unknown id still returns 404.
- **Tests:** all five tests in `tests/test_search.py` pass now, including `test_search_no_duplicates_multi_tag_song`, which failed before. The full suite is 11 passed and 2 failed, and both failures are the still open playlist issue.

**How AI helped**

Claude Code traced the search route into `search_songs`, noticed from the git history that the query had recently moved to the 2.0 `select()` style, wrote the scratch script that printed the emitted SQL and the raw joined rows, and compared every seeded song's tag count against how many times it was returned. It also ran the live and test suite checks after the fix.

### Issue 4: Rating a friend's song does not notify them

**Issue #4: I got notified when a friend added my song to a playlist but not when they rated it (status: fixed)**

**How I reproduced it**

nova and darius are friends, and nova shared Midnight Drive. Before rating, `GET /users/<nova id>/notifications` returned a count of 1: the seeded `song_added_to_playlist` notification "darius added your song 'Midnight Drive' to the playlist 'Late Night Vibes'.", which shows the expected notification format. I then sent `POST /songs/<Midnight Drive id>/rate` with `{"user_id": "<darius id>", "score": 5}`. It returned 201 with the saved rating, and the database held one rating row for darius on that song. Calling `GET /users/<nova id>/notifications` again still returned a count of 1, with only the original playlist notification and nothing about the rating. The expected result is a second notification telling nova that darius rated her song. The triggering condition is any rating by one user of a song shared by a different user.

**How I found the root cause**

`POST /songs/<song_id>/rate` is handled by `rate()` in `routes/songs.py`. It checks for `user_id` and `score`, calls `rate_song(user_id, song_id, int(score))` in `services/notification_service.py`, and returns the rating. The other side is `GET /users/<user_id>/notifications`, handled by `notifications()` in `routes/users.py`, which calls `get_notifications()` in the same service file. That function selects `Notification` rows by `user_id`, newest first, with an optional unread filter. In `models.py`, a `Notification` is just a row with a recipient `user_id`, a `notification_type`, a `body` and a `read` flag. So the symptom had two possible explanations. Either the rating never wrote a notification, or it wrote one that the read side did not find (wrong recipient, or filtered out).

Reading `rate_song`, I saw that it validates the score, loads the song and the rater, updates or inserts a `Rating`, commits, and returns, without any mention of notifications. Searching the project for `create_notification` showed exactly one call site, inside `add_to_playlist` in the same file. That function notifies `song.shared_by` with type `song_added_to_playlist` when the adder is someone else. The docstring on `create_notification` even lists `'song_rated'` as an expected type, but nothing ever created one.

I was confident when a scratch script against an in-memory database ruled out the second explanation and confirmed the first. Calling `rate_song` directly saved the rating, and updated it on a second call, while the entire `notification` table stayed at zero rows. That meant nothing was written for any user, not just the wrong one. Separately, creating a `song_rated` notification by hand and reading it back with `get_notifications`, with and without `unread_only`, returned it correctly, so the read path was fine.

**The root cause**

`rate_song()` in `services/notification_service.py` should save the rating and then notify the user who shared the song, the same way `add_to_playlist()` in that file notifies the sharer when someone adds their song to a playlist. Instead it only saved the `Rating` and returned. It never called `create_notification()`, so no `song_rated` row was ever written. The route still returned 201 with the saved rating, which made the request look successful, but when the sharer checked `GET /users/<id>/notifications` there was nothing about the rating. Only the playlist notifications, which do go through `create_notification()`, showed up.

**My fix**

In `rate_song()`, after the rating is committed, I added the same check that `add_to_playlist()` uses. If `song.shared_by` is not the rater, it calls `create_notification()` for `song.shared_by` with type `song_rated` and a body like "darius rated your song 'Midnight Drive' 5 out of 5." The notification is created only after the rating commit succeeds and after all the existing validation, so a rejected rating never produces one. People rating their own songs are skipped, which matches the playlist behavior. A re-rate also sends a notification with the new score, since the sharer would want to know the rating changed. The change is in one function in one file, and nothing else needed to change because `create_notification()` and `get_notifications()` already work.

**My side-effect check**

- **Original reproduction:** I re-ran it from a fresh seed on the live app. After darius rated nova's song Midnight Drive, nova's notifications went from 1 to 2. The new entry was `song_rated` with "darius rated your song 'Midnight Drive' 5 out of 5." above the seeded playlist notification. The rating was saved as one row, as before.
- **Calling the service directly:**
  - The rater gets no notification, only the sharer.
  - Rating your own song creates none.
  - A re-rate from 4 to 2 keeps a single `Rating` row and adds a notification with the new score.
  - Invalid scores (0 and 6), an unknown song and an unknown user still raise `ValueError` and create no notification.
- **Read side:**
  - New notifications start unread.
  - On the live app, `?unread_only=true` returned 2 before and 1 after `POST /users/notifications/<id>/read`, which returned 200.
- **Error responses:** a score of 9 still returns 400 "Score must be between 1 and 5", and a missing `user_id` still returns 400.
- **Playlist notification:** I did not change `add_to_playlist`. I could not exercise it live, because `POST /playlists/<id>/songs` fails before the notification step, as noted under Issue 5.
- **Tests:** the full suite is 11 passed and 2 failed, and both failures are the still open playlist issue.

**How AI helped**

Claude Code traced both the rate and the notifications endpoints into `notification_service.py`, searched for every call to `create_notification`, and wrote the scratch script that separated "never created" from "created but not found" before any code changed. It then wrote the fix using the same pattern `add_to_playlist` already used and ran the live and edge case checks.

### Issue 5: The most recently added song in a playlist never shows up

**Issue #5: The last song in a playlist never shows up (status: reproduced, not fixed)**

I fixed four of the five issues for this submission. This one was reproduced during Milestone 2 but not fixed, so it has a reproduction write-up only and no root cause analysis, fix or side-effect check.

**How I reproduced it**

The seeded playlist Late Night Vibes, created by nova, has seven entries in `playlist_entries` at positions 1 through 7, with Free Throws at position 7. `GET /playlists/<Late Night Vibes id>/songs` returned a count of 6, ending at Golden Hour (position 6), and Free Throws was missing. To watch the effect of an addition, I tried `POST /playlists/<Late Night Vibes id>/songs` with `{"song_id": "<Soft Landing id>", "added_by": "<darius id>"}`, but in this build that request returned HTTP 500 and no row was written. So the throwaway script added Soft Landing as position 8 directly in `playlist_entries`, the same way the seed script adds entries. After that, the table held 8 entries and the endpoint returned a count of 7. Free Throws now appeared as the last song, and Soft Landing, the newest addition, was the one missing. The triggering condition is simply having any songs in the playlist. Whatever song holds the highest position is always left out of the response, so each new addition hides itself and brings back the song added before it.
