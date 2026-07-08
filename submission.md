# Codebase Map

## Overview

This app is built with Flask and follows a pretty standard 3 layer setup. Requests come into the **routes**, the routes call into **services** to do the actual work, and the services talk to the **models** to read and write the database. `app.py` is what wires all of this together when the app starts.

The big idea here is separation of concerns. Routes just handle HTTP stuff like reading the request and returning JSON. Services hold all the business logic. Models just describe the database tables. Routes don't talk to the database directly, except for one simple case in `users.py`. Other than that, they always go through a service.

## Main Files

**`app.py`**
This is the application factory. It creates the Flask app, sets up the database connection (the `db` object that every other file imports), and registers the four blueprints for `/songs`, `/playlists`, `/users`, and `/feed`. Basically this file is the entry point that connects everything else together.

**`models.py`**
This file has all the SQLAlchemy models: `User`, `Tag`, `Song`, `ListeningEvent`, `Rating`, `Playlist`, and `Notification`. There are also three association tables for many to many relationships called `friendships`, `song_tags`, and `playlist_entries`. The `playlist_entries` table is a little different from the others because it stores extra info like `position`, `added_by`, and `added_at`, not just the two foreign keys. Every model also has a `to_dict()` method, which is used to turn the model into JSON in the routes.

**`routes/`**
There are four blueprint files in here, one for each resource.

1. `songs.py` handles search, getting a song by id, rating a song, and logging a listen.
2. `playlists.py` handles creating a playlist, getting playlist info, and getting or adding songs.
3. `users.py` handles getting a user, getting their streak, and getting notifications.
4. `feed.py` handles friends listening now and the activity feed.

The routes follow the same basic pattern every time. They grab data from the request, check that the required fields are there, call a service function, and return the result as JSON. If something goes wrong, the services raise a `ValueError` and the route turns that into an error response, either a 400 or a 404.

**`services/`**
This is where the actual logic lives.

1. `search_service.py` searches songs by title or artist.
2. `streak_service.py` records a listening event and updates the user's streak based on the date.
3. `feed_service.py` gets friends' recent listening activity from the last 24 hours, and also a general activity feed.
4. `notification_service.py` creates notifications and handles rating a song and adding a song to a playlist.
5. `playlist_service.py` creates playlists and gets the songs in a playlist in order.

**`seed_data.py`**
This isn't part of the running app. It's just a script you run once to fill the database with sample users, songs, and playlists so there's something to test with.

**`tests/`**
Three test files that test the services directly: `test_streaks.py`, `test_search.py`, and `test_playlists.py`.

## How It All Connects

An HTTP request first hits a file in `routes/`, which checks the request and calls a service. That service, in `services/`, does the real logic and talks to the database. The database itself is defined in `models.py`. So the order is basically routes, then services, then models.

`app.py` sits above all of this since it's the file that registers the routes and owns the database connection that everything else imports.

## Data Flow Example: Rating a Song

I wanted to trace one feature end to end, so I picked rating a song since it touches a route, a service, and a model.

1. The client sends a `POST` request to `/songs/<song_id>/rate` with a JSON body like `{"user_id": ..., "score": ...}`.
2. This hits the `rate` function in `routes/songs.py`. It pulls `user_id` and `score` out of the request body. If either one is missing, it returns a 400 error right away. Otherwise it calls `rate_song(user_id, song_id, score)`, which actually lives in `notification_service.py`. That surprised me a little since I expected it to be in a songs service.
3. Inside `rate_song()`, the function checks that the score is between 1 and 5, looks up the `Song` and the `User` to make sure they both exist, and checks if this user already rated this song before. If a rating already exists, it updates the score. If not, it creates a brand new `Rating`. Either way, it saves the change to the database.
4. The route gets the `Rating` object back, turns it into a dictionary with `to_dict()`, and returns it as JSON with a 201 status.

So the full path is basically route validates the input, service does the logic and touches the database, then the route serializes and returns the result. This same pattern shows up in pretty much every endpoint in the app.

## Patterns I Noticed

Routes are thin and services are thick. The routes barely do anything besides checking inputs and calling a service. All the real decision making happens in the services folder.

`ValueError` is used for everything. Whether something wasn't found or the input was invalid, services raise a `ValueError`, and the route turns it into either a 400 or a 404. There isn't a separate error type for "not found" versus "bad input."

Association tables aren't all simple. Most many to many tables, like `song_tags`, are just two foreign keys. But `playlist_entries` also stores ordering info and who added the song. Because of that, `playlist_service.py` has to query that table directly instead of just using the normal relationship.

Services sometimes call other services. For example, `notification_service.py` imports from `playlist_service.py`. So the layers aren't perfectly isolated since services can depend on each other, not just on models.

Naming doesn't always match exactly. `rate_song()` is in `notification_service.py`, not in a ratings or songs service. I had to actually open the file to find it instead of guessing from the name.

# Milestone 2: Bug Reproduction

I seeded the database with `python seed_data.py` and ran the app with `FLASK_APP=app:create_app flask run`. The seed script generates new random UUIDs every run, so I had to grab the actual user and song IDs for this run (a quick `GET /songs/search`, plus checking the seeded users) before making the requests below.

## Issue #1: My listening streak keeps resetting

I ran `pytest tests/test_streaks.py -v` instead of testing this by hand. One of the tests, `test_streak_increments_on_sunday`, has a user listen on a Saturday and then the next day, a Sunday, using hardcoded dates so the result doesn't depend on what day it actually is.

Two days listened in a row, Saturday then Sunday, should bump the streak from 1 to 2 like any other consecutive pair. Instead the test fails on `assert 1 == 2` — the streak stays at 1 after the Sunday listen. Sunday is resetting the streak instead of continuing it.

## Issue #4: I got notified when a friend added my song to a playlist but not when they rated it

"Crown Heights Anthem" belongs to simone in the seed data, so I had nova rate it:

```
curl -X POST "http://127.0.0.1:5000/songs/<crown_heights_anthem_id>/rate" \
  -H "Content-Type: application/json" \
  -d '{"user_id":"<nova_id>","score":5}'
```

then checked simone's notifications:

```
curl "http://127.0.0.1:5000/users/<simone_id>/notifications"
```

As a sanity check I also looked at nova's notifications, which already has a seeded `song_added_to_playlist` notification from darius adding "Midnight Drive" — so that notification path works fine.

The rating itself succeeds (201, saved to the DB), but `GET /users/<simone_id>/notifications` comes back with `"count": 0`. simone should be notified that nova rated her song, same as nova was notified about the playlist add. Rating a song just doesn't create a notification at all.

## Issue #5: The last song in a playlist never shows up

nova's "Late Night Vibes" playlist has 7 songs seeded in order: Midnight Drive, Still Waters, First Light, Block Party, Late Night Session, Golden Hour, Free Throws. I requested its songs:

```
curl "http://127.0.0.1:5000/playlists/<late_night_vibes_id>/songs"
```

All 7 should come back in position order, ending with "Free Throws." Instead the response has `"count": 6` and stops at "Golden Hour" — "Free Throws," the last song by position, is missing entirely.

# Milestone 3: Bug Fixes

## Issue #1: My listening streak keeps resetting

**How I reproduced it:** Ran `pytest tests/test_streaks.py -v`. `test_streak_increments_on_sunday` failed — a Saturday listen then a Sunday listen should make the streak go from 1 to 2, but it stayed at 1.

**How I found the root cause:** Opened `services/streak_service.py` and read `update_listening_streak()`, since that's what the failing test calls. Found this line: `elif days_since_last == 1 and today.weekday() != 6:`.

**The root cause:** `weekday() == 6` means Sunday, so this line was blocking the streak from incrementing specifically on Sundays. There's no reason a streak should care what day of the week it is, just whether the listen was 1 day after the last one.

**My fix and side-effect check:** It was unnecessary to check for Sunday, so I removed that condition, leaving `elif days_since_last == 1:`. Reran the tests and all 5 pass now, including the Sunday one. I also checked the other tests (same-day listens, skipped days) still pass, so the rest of the streak logic wasn't affected.
