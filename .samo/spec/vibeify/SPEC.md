# SPEC — vibeify (v0.1)

## 1. Persona

Veteran Spotify Web API playlist engineer. Deep familiarity with OAuth 2.0 authorization flows, the Web API track/recommendations/playlist endpoints, rate-limit behavior (429 + Retry-After), and the practical quirks of building user-facing music tooling against Spotify as the system of record.

## 2. Idea

vibeify helps a user create Spotify playlists from a free-text vibe/mood prompt, seeded and shaped by the music the user already listens to (tracks pulled from their existing playlists). The output is a new Spotify playlist in the user's own account, cohesive with the requested vibe but anchored in the user's taste.

## 3. Scope (v0.1)

### In scope
- Single-user CLI (or minimal local web app) targeting one authenticated Spotify account.
- Pull candidate tracks from the user's existing playlists (owned + followed).
- Accept a free-text vibe prompt; use an LLM to translate vibe → seed tracks/artists/genres and target audio-feature ranges.
- Rank/select tracks from the user's library against the LLM-derived vibe profile.
- Create a new playlist in the user's Spotify account with the selected tracks.
- Persist a local run log (prompt, seed inputs, selected track URIs, resulting playlist ID) for reproducibility.

### Out of scope (v0.1)
- Multi-user SaaS, sharing, collaborative playlists.
- Non-Spotify sources (Apple Music, YouTube, local files).
- Auto-refresh / scheduled playlist regeneration.
- Mobile UI, notifications, social features.
- Training or fine-tuning a custom recommendation model.

## 4. Interview Decisions

### 4.1 auth_flow — Authorization Code with PKCE

**Decision:** Use Spotify's Authorization Code flow **with PKCE** (no client secret shipped).

**Why:**
- vibeify is a public client (CLI / local app) that cannot safely hold a client secret.
- PKCE is Spotify's officially recommended flow for public clients and supports user-level scopes required for playlist writes.
- Returns a refresh token, so the user only authenticates interactively once.

**Scopes requested:**
- `playlist-read-private`, `playlist-read-collaborative` — read source playlists.
- `playlist-modify-private`, `playlist-modify-public` — create the output playlist.
- `user-library-read` — optional, to include Liked Songs as a source.

**Token storage:** Refresh token stored in the OS keychain where available (macOS Keychain / Windows Credential Manager / libsecret). Fallback: `~/.config/vibeify/token.json` with `0600` permissions. Never committed; never logged.

**Redirect URI:** `http://127.0.0.1:<ephemeral-port>/callback` with a local loopback listener bound only for the duration of the auth dance.

### 4.2 playlist_source — LLM-mediated vibe prompt over user's library

**Decision (confirmed):** Free-text vibe prompt → LLM → vibe profile (seed artists/tracks/genres + target audio-feature ranges) → rank user's existing playlist tracks against that profile → select final tracklist.

**Pipeline:**
1. Fetch the user's playlists via `GET /me/playlists` (paginated).
2. Collect track URIs from each playlist via `GET /playlists/{id}/tracks`. De-duplicate into a candidate pool.
3. Fetch audio features in batches of 100 via `GET /audio-features`.
4. Send vibe prompt + compact summary of the candidate pool (artist/genre distribution, feature ranges) to the LLM. LLM returns structured JSON: `{ target_features: {...}, seed_artists: [...], seed_genres: [...], mood_keywords: [...], exclusions: [...] }`.
5. Score each candidate track: weighted distance across target audio features + artist/genre match + mood-keyword match on track/album metadata. Apply exclusions.
6. Select the top N (default 30, configurable 10–100) subject to the dedup/ordering rules in §4.5.

**LLM:** Pluggable. Default `claude-opus-4-7`. Prompt + response schema versioned in-repo so output is reproducible.

### 4.3 persistence — Spotify is the system of record; local run log for reproducibility

**Decision:** The generated playlist is created **directly in the user's Spotify account** (via `POST /users/{user_id}/playlists` + `POST /playlists/{id}/tracks`). Spotify is the source of truth for the playlist itself.

Locally, vibeify keeps a minimal append-only run log at `~/.local/share/vibeify/runs/<timestamp>.json` containing:
- Prompt text, LLM model ID, prompt/response hashes.
- Source playlist IDs sampled.
- Selected track URIs (ordered) and per-track scores.
- Resulting playlist ID + URL.

An SQLite cache at `~/.local/share/vibeify/cache.db` stores audio features and track metadata keyed by track URI with a 30-day TTL to avoid re-fetching on every run.

**Why:** Creating the playlist in Spotify directly matches user expectation ("I asked for a Spotify playlist, I want it in Spotify") and avoids building a parallel UI. The local log enables re-runs, debugging, and A/B comparison across prompts without polluting the user's account.

### 4.4 rate_limit_strategy — Respect Retry-After, bounded exponential backoff, client-side throttling

**Decision:**
- All Spotify API calls go through a single HTTP client wrapper.
- On **429**: read `Retry-After` (seconds), sleep for that duration plus 0–500 ms jitter, then retry. Retry the same request up to 5 times; after that, surface the error.
- On **5xx**: exponential backoff — 1s, 2s, 4s, 8s, 16s with jitter; max 5 retries.
- On **401**: refresh the access token once and retry; if still 401, re-prompt the user to re-authenticate.
- Client-side token bucket: default ceiling of 10 requests/second per authenticated user, tunable via config. Keeps us well under Spotify's undocumented per-app ceilings and reduces 429 incidence.
- Batch endpoints used wherever possible: `/audio-features` (100/req), `/tracks` (50/req), playlist track adds (100/req).
- All retry logic is idempotent-aware: POST/PUT/DELETE requests are retried only on 429 and 5xx, never on 4xx other than 429.

**Why:** Honoring `Retry-After` is required by Spotify's ToS-adjacent guidance and avoids app-level bans. Combining it with client-side throttling + batching keeps typical runs well below any limit even with large libraries.

### 4.5 track_dedup_scope — URI-level dedup, canonical preference, stable ordering

**Decision:**
- **Dedup key:** Spotify track URI. A secondary dedup pass uses `(normalized_title, primary_artist_id)` to catch re-releases/remasters where the URI differs but the recording is effectively the same — in a collision, prefer the track already present in the user's most-played source playlists (proxy: track appears in the most playlists).
- **Exclusions:** Drop local tracks (`is_local: true`), unplayable tracks (`is_playable: false` in user's market), and tracks explicitly excluded by the LLM profile.
- **Ordering:** Final playlist is ordered to produce a listenable arc — sort by LLM-derived target-feature distance within 3 "segments" (opener / core / closer) rather than pure score descending. Deterministic for a given (prompt, library snapshot, model) triple: ties broken by track URI lexicographic order.
- **Length:** Default 30 tracks; `--length N` overrides (clamped 10–100). If the candidate pool after filtering is smaller than requested, emit a warning and use the full filtered pool.
- **Explicit content:** Include by default; `--clean` flag excludes tracks marked `explicit: true`.

**Why:** URI dedup is cheap and correct for the common case; the title/artist fallback catches the "same song, different release" problem users notice. Segmented ordering produces a playlist that flows rather than front-loading the best-match tracks.

## 5. Architecture

```
  ┌──────────────┐   vibe prompt    ┌───────────────┐
  │   CLI / UI   │─────────────────▶│  Orchestrator │
  └──────────────┘                  └───────┬───────┘
                                            │
              ┌─────────────────────────────┼─────────────────────────────┐
              ▼                             ▼                             ▼
      ┌───────────────┐           ┌──────────────────┐          ┌──────────────────┐
      │  Spotify API  │           │   LLM Adapter    │          │  Local Storage   │
      │    Client     │           │ (vibe → profile) │          │ (SQLite + runs)  │
      └───────────────┘           └──────────────────┘          └──────────────────┘
```

**Modules:**
- `auth` — PKCE flow, token storage, refresh.
- `spotify` — typed client wrapper over Web API with the retry/throttle policy from §4.4.
- `library` — fetch + cache user playlists, tracks, audio features.
- `vibe` — LLM prompt/response schema; translates free text → structured vibe profile.
- `rank` — scoring, dedup, ordering (§4.5).
- `publish` — creates the Spotify playlist and writes the run log.
- `cli` — user-facing entry point.

## 6. Data Shapes

**VibeProfile** (LLM output, validated):
```json
{
  "target_features": {
    "energy": [0.3, 0.6],
    "valence": [0.2, 0.5],
    "tempo_bpm": [70, 100],
    "danceability": [0.0, 0.5],
    "acousticness": [0.3, 1.0],
    "instrumentalness": [0.0, 0.4]
  },
  "seed_artists": ["spotify:artist:..."],
  "seed_genres": ["indie-folk", "ambient"],
  "mood_keywords": ["rainy", "melancholic", "late-night"],
  "exclusions": { "artist_ids": [], "genres": ["metal"] }
}
```

**RunLog entry** (local, append-only):
```json
{
  "run_id": "2026-04-20T19-02-11Z-ab12",
  "prompt": "rainy sunday morning, coffee, no lyrics",
  "llm": { "model": "claude-opus-4-7", "prompt_hash": "...", "response_hash": "..." },
  "sources": { "playlist_ids": ["..."], "candidate_count": 1843 },
  "selected": [{ "uri": "spotify:track:...", "score": 0.87 }],
  "result": { "playlist_id": "...", "url": "https://open.spotify.com/playlist/..." }
}
```

## 7. Failure Modes & Handling

| Failure | Handling |
| --- | --- |
| User declines OAuth consent | Abort with clear message; no partial state. |
| No source playlists / empty library | Refuse to generate; prompt user to add source playlists or enable Liked Songs. |
| LLM returns invalid/unparseable JSON | Retry once with stricter schema reminder; on second failure, abort with the raw response logged locally. |
| Candidate pool < requested length | Emit warning, publish what we have. |
| Playlist create fails mid-run (partial adds) | Playlist is created first (empty), tracks added in idempotent 100-track batches; on failure, retry from last successful batch. Run log records final state. |
| Refresh token revoked | Delete stored token, re-run auth flow. |

## 8. Non-Goals / Explicit Deferrals

- No web frontend in v0.1.
- No server-side component; fully local.
- No cross-account features.
- No automatic regeneration / scheduling.
- No analytics or telemetry beyond the local run log.

## 9. Open Questions (resolved or deferred)

All five interview items are resolved above. Remaining implementation-level questions (e.g., exact scoring weights, LLM prompt wording) are left to the build phase and will be tunable via config.
