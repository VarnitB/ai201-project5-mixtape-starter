# Mixtape Bug Hunt Submission

## AI Usage

I used AI assistance to navigate the codebase, trace routes into service functions, compare similar code paths, suggest small targeted fixes, and organize the RCA entries. It helped summarize the roles of `app.py`, `models.py`, `routes/`, `services/`, the seed script, and the existing tests.

I still reproduced the issues before fixing them, reviewed the code paths myself, checked the diffs, and ran the tests after each fix. For the notification bug, I also manually verified the route behavior with an in-memory app before and after the change. The AI was most useful for keeping the investigation structured; I verified the final behavior with pytest and manual checks.

## Codebase Map

### Main Files and Folders

- `README.md`: Explains the app structure, setup commands, route-to-service tracing strategy, test command, and the five known issue areas.
- `app.py`: Defines the Flask application factory `create_app()`, configures SQLAlchemy, registers the `songs`, `playlists`, `users`, and `feed` blueprints, and creates database tables inside the app context.
- `models.py`: Defines all SQLAlchemy models and association tables used by the app. It is the central data model for users, songs, tags, listening events, ratings, playlists, playlist entries, friendships, and notifications.
- `routes/`: Contains the Flask blueprints. Route files parse request data, call service-layer functions, and return JSON responses or error messages.
- `services/`: Contains the business logic. The README says the bugs live here, and the route files mostly delegate directly into these modules.
- `seed_data.py`: Rebuilds and populates the local database with test users, friendships, songs, tags, playlists, listening events, streak data, and sample notifications.
- `tests/`: Contains pytest coverage for service behavior around streaks, search, and playlist song retrieval.
- `requirements.txt`: Lists the Python dependencies needed to run the Flask app and tests.

### Route Structure

- `routes/songs.py` registers `/songs` endpoints:
  - `GET /songs/search` calls `search_service.search_songs()`.
  - `GET /songs/<song_id>` calls `search_service.get_song()`.
  - `POST /songs/<song_id>/rate` calls `notification_service.rate_song()`.
  - `POST /songs/<song_id>/listen` calls `streak_service.record_listening_event()`.
- `routes/playlists.py` registers `/playlists` endpoints:
  - `POST /playlists/` calls `playlist_service.create_playlist()`.
  - `GET /playlists/<playlist_id>` calls `playlist_service.get_playlist()`.
  - `GET /playlists/<playlist_id>/songs` calls `playlist_service.get_playlist_songs()`.
  - `POST /playlists/<playlist_id>/songs` calls `notification_service.add_to_playlist()`.
- `routes/users.py` registers `/users` endpoints:
  - `GET /users/<user_id>` reads a `User` directly from the database and returns `user.to_dict()`.
  - `GET /users/<user_id>/streak` calls `streak_service.get_streak()`.
  - `GET /users/<user_id>/notifications` calls `notification_service.get_notifications()`.
  - `POST /users/notifications/<notification_id>/read` calls `notification_service.mark_as_read()`.
- `routes/feed.py` registers `/feed` endpoints:
  - `GET /feed/<user_id>/listening-now` calls `feed_service.get_friends_listening_now()`.
  - `GET /feed/<user_id>/activity` calls `feed_service.get_activity_feed()`.

### Main Models and Data Stored

- `User`: Stores account identity fields such as username and email, plus listening streak state with `listening_streak` and `last_listened_at`. It also has relationships to shared songs, ratings, listening events, notifications, playlists, and friends.
- `Song`: Stores a shared song's title, artist, optional album, genre, sharing user, shared timestamp, and optional share note. Songs can have ratings, listening events, and tags.
- `Tag`: Stores reusable tag names, connected to songs through the `song_tags` association table.
- `ListeningEvent`: Stores a user listening to a song at a specific time. This powers listening history, streak updates, and feed features.
- `Rating`: Stores a user's 1-5 score for a song. A unique constraint prevents the same user from creating multiple rating rows for the same song.
- `Playlist`: Stores playlist metadata: name, creator, creation time, and whether it is collaborative.
- `Notification`: Stores messages sent to users, with a notification type, body, creation timestamp, and read/unread state.
- `friendships`: A many-to-many association table connecting users to friends.
- `song_tags`: A many-to-many association table connecting songs to tags.
- `playlist_entries`: A many-to-many association table connecting playlists to songs. It also stores `position`, `added_by`, and `added_at`, so playlist membership includes ordering and add metadata instead of only a plain relationship.

### Service Layer Responsibilities

- `services/search_service.py`: Looks up songs by query and fetches a single song by ID.
- `services/streak_service.py`: Records listening events and updates or retrieves a user's listening streak.
- `services/feed_service.py`: Builds the "Friends Listening Now" feed and the broader friend activity feed from listening events.
- `services/playlist_service.py`: Creates playlists, fetches playlist metadata, fetches playlists for a user, and retrieves ordered songs in a playlist.
- `services/notification_service.py`: Creates notifications, adds songs to playlists while notifying the original sharer, saves or updates ratings, retrieves notifications, and marks notifications as read.

### End-to-End Data Flow: Viewing Playlist Songs

1. A client requests `GET /playlists/<playlist_id>/songs`.
2. Flask routes the request to `get_songs()` in `routes/playlists.py`.
3. The route does not query the database directly. It calls `playlist_service.get_playlist_songs(playlist_id)`.
4. `get_playlist_songs()` first loads the `Playlist` with `db.session.get(Playlist, playlist_id)`.
5. If the playlist does not exist, the service raises `ValueError`, and the route returns a `404` JSON error.
6. If the playlist exists, the service queries `Song`, joins through the `playlist_entries` association table, filters entries for the requested playlist, and orders by `playlist_entries.position`.
7. The service converts each `Song` model to a dictionary using `song.to_dict()`.
8. The route wraps the returned list as `{"songs": songs, "count": len(songs)}` and sends it as JSON.

This flow shows the app's main organization pattern: route files handle HTTP concerns, while service files own the database query and business rules.

### Organization Patterns Noticed

- The app uses a Flask app factory pattern in `app.py`, which lets tests create isolated apps with an in-memory SQLite database.
- Most endpoints are thin route handlers. They parse request data, validate required fields, delegate to one service function, and format the response.
- Service functions generally raise `ValueError` when requested data is missing or invalid. Routes catch those exceptions and convert them into HTTP error responses.
- Models define `to_dict()` helpers so routes and services can return JSON-friendly dictionaries.
- Many-to-many relationships are represented with SQLAlchemy association tables. Some association tables are simple joins, while `playlist_entries` also stores ordering and metadata.
- Existing tests focus directly on service-layer behavior, matching the README's guidance that the known bugs live in `services/`.

## Root Cause Analysis Entries

Issues #5, #1, and #4 are fixed below.

### Issue #5: The last song in a playlist never shows up

- How I reproduced it: Before fixing, I ran `pytest tests/`. `test_playlist_returns_all_songs` expected 5 songs but got 4, and `test_playlist_returns_songs_in_order` only showed `Track 1` through `Track 4`.
- How I found the root cause: I traced `GET /playlists/<playlist_id>/songs` in `routes/playlists.py` to `playlist_service.get_playlist_songs()`. The SQL query joined `playlist_entries`, filtered by playlist, and ordered by position correctly. The problem was the final return line.
- The root cause: `get_playlist_songs()` returned `[song.to_dict() for song in songs[:-1]]`. The `[:-1]` slice always drops the final item from the list, so the last playlist song never reaches the route response.
- My fix and side-effect check: I changed the return line to use `songs` instead of `songs[:-1]`. Then I ran `pytest tests/test_playlists.py`, and all 3 playlist tests passed, including the empty playlist case.

### Issue #1: My listening streak keeps resetting

- How I reproduced it: Before fixing, I ran `pytest tests/test_streaks.py`. The Sunday test failed: Saturday 2024-06-15 started the streak at 1, then Sunday 2024-06-16 should have made it 2, but it stayed at 1.
- How I found the root cause: I traced `POST /songs/<song_id>/listen` in `routes/songs.py` to `record_listening_event()`, then into `update_listening_streak()` in `services/streak_service.py`. The date difference was correct, so the suspicious part was the weekday check.
- The root cause: The code only incremented on consecutive days when `days_since_last == 1 and today.weekday() != 6`. In Python, `weekday()` returns `6` for Sunday, so the code treated every Sunday listen as not eligible for a consecutive-day increment.
- My fix and side-effect check: I removed the Sunday exclusion and let any `days_since_last == 1` increment the streak. Then I ran `pytest tests/test_streaks.py`, and all 5 streak tests passed, including same-day listening and skipped-day reset.

### Issue #4: Rating a song does not create a notification

- How I reproduced it: I created an in-memory app with a sharer, a rater, and a song owned by the sharer. Before the fix, `rate_song(rater.id, song.id, 5)` saved the rating, but `get_notifications(sharer.id)` returned `0` notifications.
- How I found the root cause: I traced `POST /songs/<song_id>/rate` in `routes/songs.py` to `notification_service.rate_song()`. Then I compared it to the working playlist path, where `add_to_playlist()` calls `create_notification()` after a user adds someone else's song.
- The root cause: `rate_song()` created or updated the `Rating` row and committed it, but never created a notification. The shared helper `create_notification()` existed, and `add_to_playlist()` already showed the expected pattern.
- My fix and side-effect check: I added a `song_rated` notification after the rating commit when the rater is not the song's original sharer. I verified through `POST /songs/<song_id>/rate` and `GET /users/<sharer_id>/notifications`: the rating returned `201`, and the sharer received one `song_rated` notification. I also ran `pytest tests/`, and all 13 tests passed.

git log --oneline screenshot: log_ss.png