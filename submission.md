# Mixtape Bug Hunt Submission

## AI Usage

I used AI assistance during Milestone 1 for codebase orientation only. The AI helped inspect the repository structure, summarize the responsibilities of the main Flask, SQLAlchemy, route, service, seed, and test files, and trace how requests move from routes into service functions. I verified the summaries by reading the actual files myself: `README.md`, `app.py`, `models.py`, the route modules, the service modules, the seed script, and the existing tests.

No bug fixes were made during this milestone. Any bug investigation and code changes will be done later, one issue at a time, with route-to-service tracing before each fix.

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

Bug fixes have not started yet. Each future bug fix will be documented here separately after reproducing the bug, tracing from route to service, making a targeted change, and committing that fix on its own.
