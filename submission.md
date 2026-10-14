# Mixtape Bug Hunt Submission

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
- `search_service.py`: `search_songs(query)` uses a SQLAlchemy 2.0 style `select(Song)` with an outer join to `song_tags` and a case-insensitive `ilike` match on title or artist, executed through `db.session.execute(...).scalars().all()`. `get_song(song_id)` fetches a single song.
- `notification_service.py`: `create_notification(user_id, notification_type, body)` is the generic helper that inserts and commits a `Notification`. `add_to_playlist(playlist_id, song_id, added_by_user_id)` validates the song, user, and playlist, appends the song to `playlist.songs`, and then calls `create_notification` for the song's original sharer when the adder is someone else. `rate_song(user_id, song_id, score)` validates the score range and the song and user, then updates an existing `Rating` or creates a new one and commits. `get_notifications` and `mark_as_read` handle reading and acknowledging notifications.
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
6. `db.session.commit()` persists the change and the `Rating` instance is returned.
7. Back in the route, `rating.to_dict()` is returned as JSON with status 201. Any `ValueError` from the service becomes a 400 with an `error` message.

### Data flow: adding a song to a playlist

1. A client sends `POST /playlists/<playlist_id>/songs` with `{"song_id": "...", "added_by": "..."}`.
2. `add_song()` in `routes/playlists.py` checks both fields and calls `add_to_playlist(playlist_id, song_id, added_by)` from `services/notification_service.py`.
3. `add_to_playlist` loads the `Song`, the adding `User`, and the `Playlist` (importing `Playlist` locally inside the function), raising `ValueError` for any that are missing.
4. If the song is not already in `playlist.songs`, it appends it through the relationship, which writes a row into `playlist_entries`, and commits.
5. If `song.shared_by` differs from the adder, it calls `create_notification` with type `song_added_to_playlist` and a message naming the adder, song, and playlist. That helper adds and commits the `Notification`.
6. The route returns `{"message": "Song added to playlist"}` with status 201.

Reading the playlist back follows `GET /playlists/<playlist_id>/songs` to `get_songs()` in `routes/playlists.py` to `get_playlist_songs()` in `playlist_service.py`, which returns serialized songs ordered by `playlist_entries.position`, and the route wraps them as `{"songs": [...], "count": n}`.

### seed_data.py

`seed()` builds an app with `create_app()`, then drops and recreates every table, so running it always resets the database. It creates five users (nova, darius, simone, kenji, aaliya) with preset streak values, five bidirectional friendships (nova is friends with darius, simone, and kenji; darius with simone; kenji with aaliya), and ten tags. It adds thirteen songs in three groups: three with no tags shared by nova, five with one tag shared by darius, and five with three tags each shared by simone. It then inserts listening events (a few from the last half hour and several spread across earlier hours and days), sets `last_listened_at` for nova, darius, and kenji, creates three playlists ("Late Night Vibes", "Friday Energy", "Study Mode") with seven songs each at positions 1 through 7, and adds one sample `song_added_to_playlist` notification for nova. Association rows are inserted directly with `table.insert().values(...)` rather than through ORM relationships.

### tests/

`tests/` contains pytest files for three of the services: `test_streaks.py`, `test_search.py`, and `test_playlists.py`. They are run with `pytest tests/`.

### Patterns in how the app is organized

- **Application factory plus blueprints.** Everything is assembled in `create_app`, and each resource area is a separate blueprint with its own URL prefix.
- **Thin routes, fat services.** Routes only parse input, validate presence of fields, and shape responses. All database work happens in `services/`, and the README directs debugging along the same path: start at the endpoint and trace into the service it calls.
- **Exceptions as the error contract.** Services raise `ValueError` for missing records or bad input, and routes translate that into 400 or 404 JSON errors. There are no custom exception classes.
- **Serialization on the model.** Each model owns a `to_dict()` method, and services usually return lists of these dicts rather than ORM objects.
- **Side effects grouped by service.** Notification-producing actions sit together in `notification_service.py`, and streak-affecting listens sit in `streak_service.py`, even though they are reached through song and playlist URLs.
- **Mixed query styles.** Most services use the legacy `db.session.query(...)` API and `db.session.get(...)` for primary key lookups, while `search_service.py` uses the SQLAlchemy 2.0 `select()` style.
- **String UUID keys and UTC timestamps** are used consistently across all tables.

### AI assistance

Claude Code was used to help read and summarize the codebase while writing this map.

---

## Root Cause Analysis

All five issues were reproduced starting from a fresh `python seed_data.py`, with the app running through `FLASK_APP=app:create_app flask run` at `http://127.0.0.1:5000`. Each reproduction was run twice from a fresh seed and gave the same result both times. The time-dependent issues (1 and 2) were reproduced with a throwaway script that runs inside the app context, sends requests through the app's own routes with Flask's test client, and holds "now" at a fixed instant by substituting a frozen `datetime` inside the service module for the duration of each request.

### Issue 1: Listening streak resets on Sunday

**How I reproduced it**

I used simone, who starts the seed with a streak of 0 and no previous listen. With the clock held at 18:00 UTC on each day from Monday 2026-10-05 through Tuesday 2026-10-13, I sent one `POST /songs/<Golden Hour id>/listen` with `{"user_id": "<simone id>"}` per day and then called `GET /users/<simone id>/streak`. Every listen returned 201. The streak climbed 1, 2, 3, 4, 5, 6 from Monday through Saturday as expected. On Sunday the response was `{"streak": 1}` instead of 7, even though she had listened the day before. The count then started over from Sunday, giving 2 on Monday and 3 on Tuesday where 8 and 9 were expected. After the script finished, the running server returned the same `{"streak": 3}` for simone, so the stored value matched. The triggering condition is a listen on a Sunday that directly follows a listen on Saturday. Listens on consecutive days anywhere from Monday through Saturday extend the streak normally.

**Root cause**

**Fix**

### Issue 2: Friends Listening Now shows people from yesterday

**How I reproduced it**

The seeded listening events are stamped from the real clock, so to get a clean picture I created a separate viewer and five friends in the throwaway script, with symmetric friendship rows and exactly one listening event per friend. With "now" held at Wednesday 2026-10-14 08:30 UTC, the friends had last listened 5 minutes ago, 3 hours ago, at 21:00 the previous evening (11.5 hours ago), at 09:30 the previous morning (23 hours ago), and at 07:30 the previous morning (25 hours ago). `GET /feed/<viewer id>/listening-now` returned a count of 4. It included the friend whose last listen was 21:00 the night before and even the one from 09:30 the previous morning, and it left out only the 25 hour old listen. The expected result for a "listening now" feed is just the friends who listened in the last few minutes. The triggering condition is any friend whose most recent listen falls anywhere inside the previous 24 hours, so a friend who stopped listening last night still shows up the next morning.

**Root cause**

**Fix**

### Issue 3: The same song shows up more than once in search

**How I reproduced it**

`GET /songs/search?q=Anthem` returned a count of 3, with Crown Heights Anthem by Borough Kings listed three times. To find the pattern, I compared several searches with the number of tags each matching song has in the seed. `q=Hours` returned After Hours three times (3 tags). `q=Block` returned Block Party once (1 tag). `q=Elara` returned Still Waters once (0 tags) and Soft Landing once (1 tag). The triggering condition is how many tags a song has. A song shows up once for each of its tags, so songs with three tags appear three times, and songs with one tag or no tags appear once. The expected result is one entry per matching song.

**Root cause**

**Fix**

### Issue 4: Rating a friend's song does not notify them

**How I reproduced it**

nova and darius are friends, and nova shared Midnight Drive. Before rating, `GET /users/<nova id>/notifications` returned a count of 1: the seeded `song_added_to_playlist` notification "darius added your song 'Midnight Drive' to the playlist 'Late Night Vibes'.", which shows the expected notification format. I then sent `POST /songs/<Midnight Drive id>/rate` with `{"user_id": "<darius id>", "score": 5}`. It returned 201 with the saved rating, and the database held one rating row for darius on that song. Calling `GET /users/<nova id>/notifications` again still returned a count of 1, with only the original playlist notification and nothing about the rating. The expected result is a second notification telling nova that darius rated her song. The triggering condition is any rating by one user of a song shared by a different user.

**Root cause**

**Fix**

### Issue 5: The most recently added song in a playlist never shows up

**How I reproduced it**

The seeded playlist Late Night Vibes, created by nova, has seven entries in `playlist_entries` at positions 1 through 7, with Free Throws at position 7. `GET /playlists/<Late Night Vibes id>/songs` returned a count of 6, ending at Golden Hour (position 6), and Free Throws was missing. To watch the effect of an addition, I tried `POST /playlists/<Late Night Vibes id>/songs` with `{"song_id": "<Soft Landing id>", "added_by": "<darius id>"}`, but in this build that request returned HTTP 500 and no row was written. So the throwaway script added Soft Landing as position 8 directly in `playlist_entries`, the same way the seed script adds entries. After that, the table held 8 entries and the endpoint returned a count of 7. Free Throws now appeared as the last song, and Soft Landing, the newest addition, was the one missing. The triggering condition is simply having any songs in the playlist. Whatever song holds the highest position is always left out of the response, so each new addition hides itself and brings back the song added before it.

**Root cause**

**Fix**

