# Mixtape — Codebase Map (Project 5 Submission)

Mixtape is a small Flask + SQLAlchemy backend for a social music app: users share
songs, build collaborative playlists, rate songs, keep listening streaks, and see
what their friends are listening to. This document maps the codebase, traces one
feature end-to-end, and calls out the organizing patterns.

---

## AI usage

I used an AI assistant (Claude Code) throughout this project, mainly as a way to ask
questions about an unfamiliar codebase, help me hunt for the bugs, and pressure-test
my understanding. Being specific about how:

**What I asked it to explain / trace / summarize.**
- To explain the `services/` layer: what each module is responsible for and what
  each function does. This is the basis of the "Main files" section.
- To trace data flows end-to-end — e.g. how a listen event reaches a friend's feed
  (`record_listening_event` → `ListeningEvent` row → `get_friends_listening_now`),
  and how adding a song to a playlist triggers a notification. Tracing route →
  service → model out loud helped me learn the "thin route, logic in services"
  pattern.
- To help find and reproduce the five reported bugs, and to draft the root-cause
  writeups once I understood each one.

**What it helped me understand.** The biggest win was seeing *why* the app is
layered the way it is — every route delegates to one service function, services
raise `ValueError` and routes translate that to HTTP, and models expose `to_dict()`
as the serialization boundary. Once I saw that pattern, "trace the symptom back to
the service" became a repeatable strategy for all five issues.

**Where I had to verify things myself, or where the AI was incomplete / wrong.**
- **It tried to fix before reproducing.** On Issue #3 the assistant immediately
  edited the search query. I made it revert and reproduce the bug against seed data
  first — fixing before confirming the cause is guessing.
- **Its first explanation of Issue #3 was incomplete.** The claim was "the outerjoin
  produces duplicate rows," but when we actually called `search_songs("Anthem")` it
  returned **one** result, not duplicates — the opposite of what was predicted. I
  didn't accept the tidy explanation. We probed the raw SQL and found the join *does*
  fan out to 3 rows, but the legacy `query(Song).all()` API silently de-duplicates
  full entities, which masks it; the duplicates only surface on the newer
  `select().scalars().all()` path. The real story was more nuanced than the first
  answer, and I only trusted it after seeing the row counts myself.
- **AI summaries need checking against the real code.** An early codebase summary was
  drafted against a wrong mental model (wrong number of models, a table that doesn't
  exist, ratings stored in the wrong place). I cross-checked it against `models.py`
  and corrected it — a reminder that a confident-sounding summary can still be wrong.
- **A bug it did not predict.** While running side-effect checks, a test crashed and
  exposed a *separate* pre-existing bug in `add_to_playlist` (it inserts a
  `playlist_entries` row without the required `position`). Reading the code alone
  hadn't surfaced it; actually running the code did.

Bottom line: the AI was most useful for orientation, tracing, and drafting, but every
root-cause claim in this document is one I reproduced and verified by running the
code with controlled inputs — not one I took on the assistant's word.

---

## Main files and what each one does

### `app.py` — application factory + DB handle
Defines the shared `db = SQLAlchemy()` instance (imported everywhere else) and
`create_app()`, which configures the DB URI (`DATABASE_URL` env var, defaulting to
`sqlite:///mixtape.db`), registers the four route blueprints under URL prefixes
(`/songs`, `/playlists`, `/users`, `/feed`), and calls `db.create_all()`. Running
`python app.py` starts it in debug mode.

### `models.py` — the data model
Defines **7 models** and **3 association tables**:

| Model | Responsibility |
|-------|----------------|
| `User` | Account. Holds `listening_streak` + `last_listened_at` (streak state lives on the user, not a separate table). Self-referential many-to-many `friends`. |
| `Tag` | A named label for songs. |
| `Song` | A shared track. Carries `shared_by` (the sharer's user id) and `share_note` — sharing is an attribute of the song, not a separate event. |
| `ListeningEvent` | One row per "user listened to song at time". The **only** thing the feeds read from. |
| `Rating` | A user's 1–5 score for a song. Real table with a `UniqueConstraint(user_id, song_id)` — one rating per user per song. |
| `Playlist` | A named collection, with `is_collaborative` flag. |
| `Notification` | An in-app message for a recipient user, with `type`, `body`, and a `read` flag. |

Association tables: `friendships` (symmetric user↔user), `song_tags` (song↔tag), and
`playlist_entries` — the playlist↔song join table that **adds an explicit `position`
column** (plus `added_by`, `added_at`), so a playlist has an ordered sequence, not
just insertion order.

> Note vs. a common assumption: there is **no `PlaylistSong` model class** — playlist
> ordering is handled by the `playlist_entries` association table. And ratings **are**
> a first-class `Rating` model; they are not stored on `Song`.

### `routes/` — HTTP layer (Flask blueprints)
Each file is a blueprint of thin endpoints. They parse `request` input, call one
service function, and format the JSON response / error code. No business logic.

- **`songs.py`** (`/songs`) — `search`, get one song, `rate` (POST), `listen` (POST).
- **`playlists.py`** (`/playlists`) — create, get metadata, list songs, add song (POST).
- **`users.py`** (`/users`) — get profile, get streak, list notifications, mark-read (POST).
- **`feed.py`** (`/feed`) — "listening now" and "activity" feeds.

### `services/` — business logic layer
All real logic lives here; routes delegate to it.

- **`playlist_service.py`** — create playlists, fetch metadata, fetch ordered songs, list a user's playlists.
- **`search_service.py`** — case-insensitive title/artist search; fetch one song.
- **`feed_service.py`** — `get_friends_listening_now` (friends' events in the last 24h, deduped to newest-per-friend) and `get_activity_feed` (most recent N friend events, no time filter).
- **`streak_service.py`** — `record_listening_event` (writes a `ListeningEvent`, bumps the streak) and `update_listening_streak` (consecutive-day rules).
- **`notification_service.py`** — `create_notification`, `add_to_playlist` (adds a song *and* notifies the sharer), `rate_song`, `get_notifications`, `mark_as_read`.

### Supporting files
- **`seed_data.py`** — populates the DB with test users, songs, friendships, etc. **Songs are only created here** — there is no "share a song" endpoint; `shared_by` is set at seed time.
- **`tests/`** — `test_streaks.py`, `test_search.py`, `test_playlists.py`.
- **`requirements.txt`**, **`.gitignore`**.

---

## Data flow: adding a song to a playlist triggers a notification

This is the app's real notification-producing flow (songs themselves are seeded, not
shared through an endpoint, so "add to playlist" is where a user action creates a
notification).

```
POST /playlists/<playlist_id>/songs        routes/playlists.py → add_song()
  │   parses song_id, added_by from JSON body; validates both present
  ▼
notification_service.add_to_playlist(playlist_id, song_id, added_by_user_id)
  ├─ db.session.get(Song, song_id)        validate song exists
  ├─ db.session.get(User, added_by...)    validate adder exists
  ├─ db.session.get(Playlist, ...)        validate playlist exists
  ├─ if song not already in playlist:  playlist.songs.append(song); commit
  └─ if song.shared_by != added_by_user_id:      (don't notify yourself)
         create_notification(
             user_id=song.shared_by,               ← the original sharer
             notification_type="song_added_to_playlist",
             body="<adder> added your song '<title>' to '<playlist>'.")
           └─ new Notification(...); db.session.add; db.session.commit()
```

The sharer later reads it via `GET /users/<id>/notifications` →
`notification_service.get_notifications()`, and clears it via
`POST /users/notifications/<id>/read` → `mark_as_read()`.

**Contrast — rating a song:** `POST /songs/<id>/rate` → `rate_song()` writes/updates a
`Rating` but does **not** call `create_notification`. Per the tracker (issue #4), it
*should* notify the sharer just like `add_to_playlist` does — this is a known bug: the
notification step is simply missing from `rate_song`.

---

## Patterns in how the app is organized

1. **Strict route → service → model layering.** Every endpoint immediately delegates
   to one service function. Routes own input parsing and response/HTTP-status
   formatting; services own all business logic and DB work; models own persistence.
   You can predict where any logic lives from the URL alone.

2. **Services signal failure with `ValueError`; routes translate it to HTTP.** Service
   functions raise `ValueError(f"... not found")`; every route wraps the call in
   `try/except ValueError` and returns a 404 (or 400 on bad input). Errors never leak
   as 500s from expected conditions.

3. **`to_dict()` is the serialization boundary.** Models never get JSON-encoded
   directly — each defines a `to_dict()`, and services return lists/dicts of those.
   `User.to_dict()` deliberately omits `email`, so the API can't leak it.

4. **UUID string PKs everywhere**, via a shared `generate_uuid()` default — ids are
   portable and non-guessable, and every foreign key is a `String(36)`.

5. **Derived-on-read feeds.** There is no stored feed table. Feeds are computed at
   request time from `ListeningEvent` rows filtered by the viewer's `friends`. A
   "listen" is one write; feeds are pure reads.

6. **One-way service dependencies, with a lazy import to avoid a cycle.**
   `notification_service.add_to_playlist` imports `playlist_service` *inside the
   function* rather than at module top — a deliberate move to sidestep a circular
   import between the two service modules.

---

## Where the bugs live (per README issue tracker)

All five open issues sit in `services/`, matching pattern #1 above — routes are thin,
so any misbehavior traces back to the service:

| # | Symptom | Service |
|---|---------|---------|
| 1 | Streak keeps resetting | `streak_service.py` (spurious `today.weekday() != 6` Sunday condition) |
| 2 | "Listening now" shows yesterday's people | `feed_service.py` |
| 3 | Same song appears twice in search | `search_service.py` (`outerjoin` on `song_tags` multiplies rows) |
| 4 | Rating a song doesn't notify the sharer | `notification_service.py` (`rate_song` never calls `create_notification`) |
| 5 | Last song in a playlist never shows | `playlist_service.py` (`get_playlist_songs` returns `songs[:-1]`) |

---

## Bug fixes — root cause analysis

I fixed **Issues #3, #4, and #5**. Every bug was reproduced against the seed data
*before* any fix was written. Reproduction driver: an isolated scratch DB
(`DATABASE_URL` pointed at a temp file so `instance/mixtape.db` is never touched),
seeded via `seed_data.py`, then the relevant service function called directly with
controlled inputs and its output counted — faster and more precise than firing HTTP
requests. Issues #1 and #2 are left unfixed and out of scope.

### Issue #3 — The same song keeps showing up twice in search
1. **How I reproduced it:** Seeded the DB. `seed_data.py` deliberately gives
   *Crown Heights Anthem* (and the other `song_data_multi_tags` entries) **3 tags**,
   while other songs have 0 or 1. I called `search_songs("Anthem")` and counted
   rows, then counted the raw joined rows the query produces. The `outerjoin` on
   `song_tags` produced **3 raw DB rows** for the single matching song — one per tag
   (`raw joined row count: 3`). This maps exactly onto simone's "some once, others
   two or three times": row count == tag count.
2. **How I found the root cause:** Top-down from the route. `GET /songs/search`
   → [`routes/songs.py`](routes/songs.py) `search()` (just reads `q` and calls the
   service) → [`services/search_service.py`](services/search_service.py)
   `search_songs()`. Reading that function, the query is
   `query(Song).outerjoin(song_tags, ...).filter(...).all()`. The join immediately
   looked wrong: the `filter` only references `Song.title`/`Song.artist`, so the
   join contributes nothing to *which* songs match — it can only add rows. The
   moment of confidence was the probe result: the raw join returned exactly 3 rows
   for a 3-tag song and 1 row for a 1-tag song, so the duplicate count tracked tag
   count one-for-one — that is the join fanning out, not a coincidence.
3. **The root cause:** A song↔tag join is a one-to-many; joining `Song` to
   `song_tags` yields one row per (song, tag) pair. Since the query selects `Song`
   and never needed tag data for filtering (tags are loaded separately via the
   `Song.tags` relationship inside `Song.to_dict()`), the join served no purpose
   except to multiply each song's row by its tag count.
   *(Nuance: the legacy `query(Song).all()` API auto-uniquifies full entities, which
   can mask the fan-out at `.all()`; the modern `select(Song).scalars().all()` path
   returns all 3 duplicates. The join is the defect regardless of which read path is
   used, so removing it is the correct fix either way.)*
4. **The fix:** Removed the `.outerjoin(song_tags, ...)` line entirely (and the
   now-unused `Tag`/`song_tags` imports). The filter and ordering are unchanged.
5. **Side-effect check:** Confirmed `search_songs` still returns tag data via the
   relationship (`"Anthem"` → tags `['rap','hip-hop','boom bap']`), and that songs
   with 1 tag (*Block Party*) and 0 tags (*Midnight Drive*) each return exactly one
   row. Title-vs-artist matching is untouched (only the join was removed).

### Issue #4 — Notified when a friend adds my song to a playlist, but not when they rate it
1. **How I reproduced it:** As a non-owner (kenji), rated a song shared by simone:
   `rate_song(kenji.id, song.id, 5)`, then read `get_notifications(simone.id)`. The
   count stayed **0 → 0**. The `Rating` row *was* written (the score persists), but
   no `Notification` appeared — exactly aaliya's report.
2. **How I found the root cause:** Followed the working case against the broken one,
   both in the same file. `POST /songs/<id>/rate` →
   [`routes/songs.py`](routes/songs.py) `rate()` →
   [`services/notification_service.py`](services/notification_service.py)
   `rate_song()`. Reading `rate_song` top to bottom: validate score, load song and
   rater, create-or-update the `Rating`, `commit`, `return`. There is no notification
   call anywhere in it. I then read its sibling `add_to_playlist()` in the *same
   file* — the case aaliya said works — and it ends with a `create_notification(...)`
   call to `song.shared_by`. That contrast was the confirming moment: the notify step
   isn't broken, it was simply never written into `rate_song`.
3. **The root cause:** `rate_song` had no call to `create_notification` at all. The
   rating half of the feature was implemented; the notify half was omitted. Nothing
   was mis-conditioned — the code path to notify the sharer did not exist.
4. **The fix:** After the `commit`, added a `create_notification` call to
   `song.shared_by` with type `"song_rated"` and a body naming the rater, score, and
   song — guarded by `if song.shared_by != user_id:` so rating your own song doesn't
   notify yourself. This mirrors the exact guard and shape already used in
   `add_to_playlist`.
5. **Side-effect check:** Verified three paths — a new rating by a non-owner notifies
   (0→1); **re-rating** the same song (the `existing` update branch) also notifies
   (1→2), so updates aren't silently dropped; and a **self-rating** by the owner
   correctly produces no notification (no change). The rating write/return value is
   unchanged, so [`routes/songs.py`](routes/songs.py)'s `rating.to_dict()` response
   still works.
   *(Aside: while testing I found a separate, pre-existing bug in `add_to_playlist` —
   `playlist.songs.append(song)` inserts a `playlist_entries` row without the NOT
   NULL `position`/`added_by`, so adding a brand-new song crashes. It is unrelated to
   these three issues and I did not modify it.)*

### Issue #5 — The last song in a playlist never shows up
1. **How I reproduced it:** Called `get_playlist_songs` for *Friday Energy*, which
   has **7** rows in `playlist_entries`. It returned **6**. The missing one is always
   the highest `position`, i.e. the most recently added — matching darius's report
   that adding a new song "frees" the previous straggler and hides the newest.
2. **How I found the root cause:** `GET /playlists/<id>/songs` →
   [`routes/playlists.py`](routes/playlists.py) `get_songs()` →
   [`services/playlist_service.py`](services/playlist_service.py)
   `get_playlist_songs()`. The function queries songs joined to `playlist_entries`,
   orders by `playlist_entries.position` ascending, then returns
   `[song.to_dict() for song in songs[:-1]]`. The `[:-1]` slice was the immediate
   tell: it drops the final list element, and because the list is sorted by
   `position` ascending, the final element is always the most-recently-added song —
   which is precisely the reported symptom.
3. **The root cause:** The return statement sliced the ordered song list with
   `songs[:-1]`, discarding the last element. Combined with the ascending
   `position` sort, "last element" always equals "newest song," so the newest add
   was perpetually hidden.
4. **The fix:** Changed `songs[:-1]` to `songs` so every song in the ordered list is
   returned. One-token change; ordering and query are untouched.
5. **Side-effect check:** This is a boundary bug, so I checked both ends and the
   degenerate cases: a **3-song** playlist returns all 3 with the **first** and
   **last** song both present; a **single-song** playlist returns 1 (it returned 0
   before the fix — the off-by-one was worst here); an **empty** playlist returns 0
   (no crash from slicing an empty list). Ordering by `position` still holds.

### Test status after fixes
`pytest tests/` → **12 passed, 1 failed**. The single failure is
`test_streaks.py::test_streak_increments_on_sunday`, which targets the unfixed
**Issue #1** (the `weekday() != 6` Sunday condition in `streak_service.py`) — a
pre-existing failure, not a regression from these changes. `git diff --stat`
confirms only the three targeted service files were modified.
