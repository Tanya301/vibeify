# SPEC — vibeify (v0.3)

## 1. Persona

Veteran Spotify Web API playlist engineer. Deep familiarity with OAuth 2.0 authorization flows, the Web API track/playlist endpoints, rate-limit behavior (429 + Retry-After), and the practical quirks of building user-facing music tooling against Spotify as the system of record — including the post-2024 endpoint restrictions that removed `/audio-features`, `/audio-analysis`, `/recommendations`, related-artists, and 30-second previews from new/non-extended client IDs.

## 2. Idea

vibeify helps a user create Spotify playlists from a free-text vibe/mood prompt, seeded and shaped by the music the user already listens to (tracks pulled from their existing playlists). The output is a new Spotify playlist in the user's own account, cohesive with the requested vibe but anchored in the user's taste.

## 3. Scope (v0.3)

### In scope
- Single-user CLI (or minimal local web app) targeting one authenticated Spotify account.
- Pull candidate tracks from the user's existing playlists (owned + followed).
- Accept a free-text vibe prompt; use an LLM to translate vibe → a structured vibe profile over **metadata-only signals** (artist, album, genre, track name, release year, explicit flag, duration) with audio-feature signals as an **optional, capability-gated** enhancement.
- Rank/select tracks from the user's library against the LLM-derived vibe profile.
- Create a new playlist in the user's Spotify account with the selected tracks.
- Persist a local run log (prompt, seed inputs, selected track URIs, resulting playlist ID) and cached LLM response for reproducibility.
- Ship a test suite that validates the auth, retry, scoring, dedup, ordering, publish, and retention paths without live Spotify calls.

### Out of scope (v0.1/0.2/0.3)
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

**Scopes requested (always, at first auth):**
- `playlist-read-private`, `playlist-read-collaborative` — read source playlists.
- `playlist-modify-private`, `playlist-modify-public` — create the output playlist.
- `user-library-read` — include Liked Songs as a source.

All four scope families are requested up-front on first consent. `--include-liked` at runtime is a behavioral toggle, not a scope toggle; we do not perform scope upgrades mid-session.

**Token storage.** Refresh token stored in the OS keychain where available; fallback files are OS-specific and follow platform conventions:
- macOS: Keychain (primary); fallback `~/Library/Application Support/vibeify/token.json` (0600).
- Linux: libsecret (primary); fallback `$XDG_CONFIG_HOME/vibeify/token.json` or `~/.config/vibeify/token.json` (0600).
- Windows: Credential Manager (primary); fallback `%APPDATA%\vibeify\token.json` (ACL: current user only).
Never committed; never logged.

**Fallback trigger (explicit).** The fallback file is used in — and only in — exactly these cases:
1. The platform keychain API returns an error on read or write, after one transparent retry.
2. The process detects a headless session where the keychain cannot prompt for unlock. Detection rule: no controlling TTY AND (on Linux) neither `DISPLAY` nor `WAYLAND_DISPLAY` is set, or (on macOS) the `security` framework reports no interactive session, or (on Windows) the process is running as a non-interactive service account.
3. The user passes `--no-keychain` explicitly.

In any other case — notably when the keychain is simply locked and can be prompted — vibeify does **not** silently fall back. If the keychain is unusable for a non-triggering reason, vibeify aborts with an actionable message. The on-disk 0600/ACL'd fallback is therefore opt-in (flag) or environment-forced (headless), never a silent degradation.

**Redirect URI:** Spotify requires exact-match registration including port, so we pin a small, registered set. Primary: `http://127.0.0.1:8898/callback`. Fallbacks (tried in order if 8898 is busy): `8899`, `8900`, `8901`. All four are registered in the Spotify app dashboard. If all four are in use, abort with an actionable error (§7). The loopback listener is bound only for the duration of the auth dance.

### 4.2 playlist_source — LLM-mediated vibe prompt over user's library (metadata-first)

**Decision (confirmed, revised for API reality):** Free-text vibe prompt → LLM → vibe profile (seed artists/genres + target metadata signals) → rank user's existing playlist tracks against that profile → select final tracklist. Audio-feature scoring is **optional and capability-gated**: vibeify probes at startup whether the authenticated app has access to `/audio-features`; if and only if the probe succeeds, feature-distance scoring is enabled as an additional score component.

**Pipeline:**
1. Fetch the user's playlists via `GET /me/playlists` (paginated). If `--include-liked` is set, also fetch `GET /me/tracks`.
2. Collect track URIs from each source via `GET /playlists/{id}/tracks` (and `/me/tracks`). All track-reading endpoints are called with `market=from_token` so `is_playable` is populated. De-duplicate into a candidate pool.
3. **Capability probe.** One `GET /audio-features/{id}` call against a pinned canary URI baked into the binary — the current canary is `spotify:track:4iV5W9uYEdYUVa79Axb7Rh` (a long-lived, globally-available Spotify-catalog track; the constant is versioned in-repo under `vibe/canary.py` and can be rotated without a spec change). The probe classifies responses as follows: `200` → `features_mode=true`. `401` → refresh access token once and retry the probe; still `401` → abort with an auth error (not a capability failure). `403` → `features_mode=false` (capability-denied). `404` → `features_mode=false` (treated conservatively as capability-denied; a 404 on a known-live canary means either the canary was removed from the global catalog, in which case the canary needs rotating, or the app's token genuinely cannot resolve features-endpoint track IDs). `429`/`5xx` → follow the §4.4 retry policy; if retries are exhausted the run aborts before any LLM spend. The probe result and classification are recorded in the run log and cached per `(app_client_id, user_id)` for 24 h.
4. If `features_mode=true`, fetch audio features in batches of 100 via `GET /audio-features`. On `403`/`404` mid-run (Spotify revoking access between probe and fetch), **discard any partially-fetched audio features for this run**, set `features_mode=false` for the remainder of the pipeline, record the degradation event and the count of discarded feature rows in the run log, and continue to steps 5+ with metadata-only scoring. Partially-fetched features are never retained or mixed with metadata-only scoring within a single run — that would make score magnitudes incomparable across the candidate pool.
5. Build the LLM input: prompt + compact candidate-pool summary (see §4.2.1). Send to the LLM with `temperature=0` and any provider-specific seed. LLM returns structured JSON matching the VibeProfile schema in §6.
6. Validate the VibeProfile: JSON-schema check; resolve every LLM-emitted `spotify:artist:...` URI via batched `GET /artists?ids=` (50/req). Unresolved-URI handling (see §4.2.2) is applied independently to `seed_artists` and to `exclusions.artist_ids`.
7. Score each candidate track per §4.5.1. Apply exclusions.
8. Select the top N (default 30, configurable 10–100) subject to the dedup/ordering rules in §4.5.

**4.2.1 LLM context-window strategy.** The candidate-pool summary sent to the LLM is capped at 8 000 tokens and composed of: top 50 artists by track count; genre histogram bucketed to the top 30 genres; release-year histogram in 5-year bins; track-count, explicit-ratio, mean-duration, and (if `features_mode=true`) per-feature min/p25/p50/p75/max. Individual track titles are not sent. When the full summary exceeds 8 000 tokens, artist/genre lists are truncated first; the truncation counts are recorded in the run log.

**4.2.2 Unresolved-URI policy (concrete).** For each of the two URI sets `seed_artists` and `exclusions.artist_ids`, after the batched `/artists` lookup:
- Let `total` = number of URIs the LLM emitted in that set, `unresolved` = number that did not resolve (404 or missing from response).
- If `total < 3`, drop the unresolved URIs with a warning and continue — small sets are too noisy to treat as a quorum.
- Else, **abort the run** if `unresolved > 0.5 * total` (strict inequality; at exactly 50% the run continues with a warning).
- The two sets are evaluated independently: a seed_artists abort does not depend on exclusions.artist_ids resolution and vice versa.
- Abort behavior writes a run log entry with the unresolved URIs and the numerator/denominator, then exits non-zero without creating a playlist.

**LLM:** Pluggable. Default `claude-opus-4-7`. Sampling is pinned: `temperature=0`, `top_p=1`, provider seed recorded when supported (with an explicit note in the run log for providers that do not honor seeds — determinism in §4.5 is conditional on seed honoring, and providers whose metadata does not assert seed support record `seed_honored=false` in the run log). System prompt and response schema are versioned in-repo; the `prompt_hash` is the SHA-256 of `system_prompt || user_prompt || schema_version`. LLM responses are cached locally keyed by `prompt_hash` so repeat runs short-circuit the LLM call.

### 4.3 persistence — Spotify is the system of record; local run log + LLM cache for reproducibility

**Decision:** The generated playlist is created **directly in the user's Spotify account** (via `POST /users/{user_id}/playlists` + `POST /playlists/{id}/tracks`). Spotify is the source of truth for the playlist itself.

Locally, vibeify keeps:
- An append-only run log at `<data_dir>/runs/<timestamp>.json` containing prompt text, LLM model ID, prompt/response hashes, seed, sampling params, capability-probe result and classification, source playlist IDs, truncation stats, candidate count, selected track URIs (ordered) with per-track score breakdowns, resulting playlist ID + URL.
- An LLM response cache at `<data_dir>/llm-cache/<prompt_hash>.json` (content-addressed, never expired; evicted by LRU when the cache exceeds 500 MB).
- A SQLite cache at `<data_dir>/cache.db` storing track metadata and (if applicable) audio features keyed by track URI. Audio features and core track metadata are treated as **immutable**; they are cached indefinitely and evicted only by LRU when the DB exceeds 200 MB. Playlist membership and `is_playable` are cached with a 24 h TTL because they are user/market-dependent.
- `<data_dir>` resolution: Linux `$XDG_DATA_HOME/vibeify` or `~/.local/share/vibeify`; macOS `~/Library/Application Support/vibeify`; Windows `%LOCALAPPDATA%\vibeify`.

**Cache eviction — concrete ordering.**
- `cache.db`: when DB size exceeds 200 MB, eviction runs in two passes. **Pass 1:** delete all TTL-bearing entries (playlist membership, `is_playable`) whose TTL has expired. **Pass 2:** if still over cap, evict by LRU across all remaining entries (including unexpired TTL entries and immutable entries), oldest access-time first. LRU recency is updated on **both reads and writes** — any hit bumps the entry's `last_accessed_at`.
- `llm-cache/`: when directory size exceeds 500 MB, evict by LRU, oldest access-time first. Recency is updated on both reads and writes; a cache hit refreshes the mtime.
- Eviction runs at the end of a vibeify invocation (after the run log is written) so it never interferes with the running pipeline, and runs synchronously to avoid partial-state surprises.

**Run-log retention:** a `vibeify runs prune --keep N` command (default keeps the last 200 runs by filename timestamp, newest first) and an auto-prune that runs at startup when >1 000 run logs are present. Auto-prune reduces the count to 200 (the same default). The in-flight run's log file is written at the end of the invocation, so auto-prune cannot delete it.

**Why:** Creating the playlist in Spotify directly matches user expectation and avoids building a parallel UI. The local log + LLM cache enables re-runs, debugging, and A/B comparison across prompts. Treating audio features as immutable reflects Spotify's data model and avoids gratuitous re-fetching.

### 4.4 rate_limit_strategy — Respect Retry-After, bounded exponential backoff, client-side throttling, testable wrapper

**Decision:**
- All Spotify API calls go through a single HTTP client wrapper. The wrapper accepts an **injectable clock and transport** so every policy below is unit-testable with a fake clock and a scripted transport (see §9).
- On **429**: read `Retry-After`. Per RFC 7231 this may be either `delta-seconds` (integer) or an HTTP-date. Parse both forms; on HTTP-date, compute `max(0, date − now)`. On unparseable or missing header, use 1 s. Sleep that duration plus 0–500 ms uniform jitter; retry up to 5 times; after that, surface the error.
- On **5xx**: exponential backoff — 1, 2, 4, 8, 16 s plus 0–500 ms jitter; max 5 retries (subject to per-method carve-outs below).
- On **401**: refresh the access token once and retry; if still 401, re-prompt the user to re-authenticate.
- On **403/404** on capability-gated endpoints (`/audio-features`, `/recommendations`): never retry; flip the relevant capability flag (§4.2 steps 3/4).
- Client-side token bucket: default capacity **10 tokens**, refill **10 tokens/sec** (continuous, i.e., one token per 100 ms). This permits a cold-start burst of 10 requests then throttles to steady-state 10 req/s. Both values are tunable via config.
- Batch endpoints used wherever possible: `/audio-features` (100/req), `/tracks` (50/req), `/artists` (50/req), playlist track adds (100/req).

**Idempotency rules (per method/endpoint):**
- GET requests retry on 429 and 5xx per the schedule above.
- PUT/DELETE retry on 429 and 5xx (Spotify's playlist mutation endpoints are naturally idempotent at the batch granularity used here).
- **`POST /users/{user_id}/playlists` (playlist create) — retries on 429 only.** On 5xx the POST is **not** retried (there is no Spotify-provided idempotency key; a blind retry would create a duplicate empty playlist). Instead, reconciliation runs as below.
- **`POST /playlists/{id}/tracks` (track add) — retries on 429 only.** On 5xx the POST is **not** blindly retried, because without a `position` parameter this endpoint appends, and a 5xx-after-server-applied followed by a retry would duplicate tracks. Recovery runs as below.
- Never retry 4xx other than 429.

**5xx recovery — playlist create.**
1. Immediately before `POST /users/{user_id}/playlists`, fetch the first page of `GET /me/playlists` and record the set `P_before` of playlist IDs (up to the first 50; playlist create surfaces in the most-recent page). Store `P_before` in-memory for this run.
2. Issue the POST.
3. On 5xx (no body, or a body without a usable playlist ID): fetch `GET /me/playlists` (first page) again and compute `P_after \ P_before`. Within that set, select playlists whose `name` and `description` match the values the client intended to send.
   - Exactly one match → **reuse** that playlist ID for the track-add phase; log the reconciliation.
   - Zero matches → **abort cleanly** without creating a duplicate. Run log records the failure.
   - More than one match → **abort cleanly** and surface an error asking the user to rename/delete the conflicting playlists; we never pick blindly.
4. Reconciliation is bounded: it runs at most once per create attempt and is not itself retried on 5xx (the GET is retried per the GET rules, but the reconciliation decision is made once against the GET result).

**5xx recovery — track add.**
1. Before the first track-add batch, record `length_before = 0` (new playlist). Before each subsequent batch, record `length_before = length_after_previous_batch` (the client tracks expected length).
2. Issue `POST /playlists/{id}/tracks` for a 100-URI window. Each successful response returns a `snapshot_id` and the client increments `length_after` by the batch size.
3. On 5xx, do **not** blindly retry. Instead: `GET /playlists/{id}?fields=tracks.total` to read the server-side length.
   - If `server_total == length_before + batch_size` → the 5xx happened after the server applied the batch; record success, advance `length_before`, proceed to next batch.
   - If `server_total == length_before` → the batch was not applied; retry the exact same 100-URI window (subject to the §4.4 retry-count ceiling of 5).
   - If `server_total` is any other value → **abort cleanly**, leaving the partial playlist intact with its final state recorded in the run log.
4. This recovery path uses the playlist's monotonic length as the client-side disambiguator; it does not rely on `snapshot_id` semantics beyond the success path.

**Why:** Honoring `Retry-After` is required by Spotify's guidance and avoids app-level bans. Reconciliation via `P_before`/`P_after` set difference with a name+description match is deterministic and does not depend on `snapshot_id` encoding creation time (which it does not). Track-add reconciliation via server-side length is deterministic and survives the endpoint's append-without-position behavior. Making the clock and transport injectable is what lets §9 test any of this.

### 4.5 track_dedup_scope — URI-level dedup, canonical preference, stable ordering

**Decision:**
- **Primary dedup key:** Spotify track URI.
- **Secondary dedup key:** `(normalized_title, primary_artist_id)` to catch re-releases/remasters/live versions. `normalized_title` is computed by this exact sequence:
  1. Unicode NFKD normalization, then drop combining marks (category `Mn`).
  2. Lowercase.
  3. Strip any trailing parenthetical/bracketed suffix matching `\s*[\(\[\{][^\)\]\}]*[\)\]\}]\s*$` (applied repeatedly until no more trailing brackets; e.g., `"Dreams (Live at Wembley) (Remastered)"` → `"Dreams"`).
  4. Strip any trailing qualifier of the form `\s+-\s+.+$` (e.g., `" - 2004 Remaster"`, `" - Single Version"`).
  5. Replace all characters outside `[a-z0-9\s]` with a single space.
  6. Collapse runs of whitespace to a single space and trim.
- In a dedup collision, prefer the track with the highest **source-playlist count** (appears in the most of the user's source playlists); ties broken by most-recent `added_at` across those playlists, then by track URI lexicographic order. (Renamed from 'most-played' in v0.1 — the proxy is source-playlist coverage, not play count, which is not exposed by the Web API.)
- **Exclusions:** Drop local tracks (`is_local: true`), unplayable tracks (`is_playable: false` — all track reads use `market=from_token`, so this field is populated), and tracks matching `exclusions.artist_ids` or `exclusions.genres` from the VibeProfile.
- **Ordering:** See §4.5.2.
- **Length:** Default 30 tracks; `--length N` overrides (clamped 10–100). If the candidate pool after filtering is smaller than requested, emit a warning and use the full filtered pool.
- **Explicit content:** Include by default; `--clean` flag excludes tracks marked `explicit: true`.

**Determinism.** Output is deterministic for a given `(prompt_hash, library_snapshot, model, seed)` quadruple **provided the LLM provider honors the seed at temperature=0**. Providers that do not are flagged in the run log (`seed_honored=false`) and the determinism claim is weakened to best-effort for those providers.

**`library_snapshot` is defined as the tuple of:**
1. The set of source playlist IDs used this run (owned + followed + Liked Songs if `--include-liked`), sorted.
2. For each source playlist, its Spotify `snapshot_id` at fetch time.
3. The `market` value used on track reads (always `from_token`, but recorded for audit).
4. The capability-probe classification (`features_mode` and the probe's HTTP status).
5. The SHA-256 of the serialized candidate-pool summary that was sent to the LLM (§4.2.1).
6. The set of `/artists` IDs resolved (the URIs the LLM emitted, successfully looked up), sorted.

The run log stores `library_snapshot` as an embedded object; §9 test 6 freezes this tuple to reproduce output.

**4.5.1 Scoring function (form specified).** Each surviving candidate track `t` receives a score in `[0, 1]`:

`score(t) = w_f · feature_score(t) + w_a · artist_score(t) + w_g · genre_score(t) + w_m · mood_score(t) + w_y · year_score(t)`

with default weights `w_f=0.35, w_a=0.25, w_g=0.15, w_m=0.15, w_y=0.10` when `features_mode=true`; when `features_mode=false`, `w_f=0` and the remaining weights are renormalized to sum to 1 (so `w_a=0.385, w_g=0.231, w_m=0.231, w_y=0.154` after division by `0.65`). All weights are config-tunable; the exact tuned values are a build-phase concern.

Component definitions:
- `feature_score(t)`: mean over the features listed in `target_features` of `range_fit(v, [lo, hi])`, where `range_fit` returns `1.0` if `lo ≤ v ≤ hi`, else `max(0, 1 − |v − nearest_bound| / scale_f)` with per-feature `scale_f` (energy/valence/danceability/acousticness/instrumentalness: `0.3`; tempo_bpm: `30`). Tempo is a float throughout; any integer bounds in the VibeProfile are coerced to float at validation time.
- `artist_score(t)`: `1.0` if any of `t`'s artist IDs ∈ `seed_artists`, else `0.5` if any of `t`'s artist IDs share ≥1 genre with any seed-artist (lookup via cached `/artists`), else `0.0`.
- `genre_score(t)`: Jaccard overlap of `t`'s artist-genres (union across all of `t`'s artists) with `seed_genres`. Defined as `|A ∩ B| / |A ∪ B|` with `0.0` when both sets are empty.
- `mood_score(t)`: token-level match of `mood_keywords` against the lowercased concatenation `t.name + ' ' + t.album.name + ' ' + genres_text`, where `genres_text` is the space-separated join of all of `t`'s artist-genres with each genre's internal hyphens replaced by spaces (so `indie-folk` tokenizes to `{indie, folk}`, not `{indie-folk}`). Tokenization of both the keywords and the concatenated target is a split on `[^a-z0-9]+` after lowercasing, producing a set of tokens. Score is `|matched_keyword_tokens| / |keyword_tokens|`. **If `mood_keywords` is empty or `keyword_tokens` is empty after tokenization, `mood_score(t) = 0.0` for all `t` and the component contributes nothing** (the weight `w_m` is kept in the sum; it is not renormalized out, because empty mood_keywords is a signal of a vibe that does not lean on mood text).
- `year_score(t)`: `1.0` unless the VibeProfile contains a `year_range` field, in which case `range_fit` over `t.album.release_year` with `scale_y = 10`.

**4.5.2 Segmented ordering (opener / core / closer).** The final N tracks are split into three segments by count — `opener = ceil(0.2·N)`, `closer = ceil(0.2·N)`, `core = N − opener − closer`. Each segment has a target energy level derived from the VibeProfile's `target_features.energy` midpoint `m`:
- opener target: `max(0, m − 0.10)` — ease into the vibe.
- core target: `m` — hold at the vibe's declared center.
- closer target: `max(0, m − 0.20)` — taper off.

These offsets are intentional: vibeify playlists are "vibe" playlists (often low-key moods), and the intended arc is ease-in / hold / wind-down. The asymmetric 0.10/0.20 offsets were chosen — not calibrated against user data — to give a clearly softer closer than opener without inverting the arc. They are config-tunable.

Within each segment, tracks are ordered ascending by `|track_energy − segment_target|`. Assignment of tracks to segments is done greedily from the top-scored pool: each track is assigned to the segment whose target it best fits, subject to the segment's capacity; ties broken by overall `score(t)` descending, then by track URI lexicographic order.

**Metadata-only fallback (when `features_mode=false`).** Since per-track energy is unavailable, segments and intra-segment order use artist- and duration-derived signals exclusively:
- Define `artist_distinctiveness(a)` as `1 / sqrt(candidate_count(a))`, where `candidate_count(a)` is the number of candidate tracks by artist `a` in the pool. Artists who appear many times are less distinctive; one-off artists are more distinctive. For multi-artist tracks, use the max over the track's artists.
- Define `artist_rotation_key(t) = (artist_id_of_first_artist, within_artist_score_rank(t))`, where `within_artist_score_rank(t)` is `t`'s rank among that artist's candidates ordered by `score(t)` descending. Sort by this key to round-robin artists. Lexicographic order on tuples gives a stable round-robin (artist A's rank-1 track, then artist B's rank-1 track, ..., then artist A's rank-2 track, etc., sorted by artist_id).
- **Opener segment:** tracks ordered ascending by `-artist_distinctiveness` (most distinctive first), then by `artist_rotation_key`. Take the top `opener` count.
- **Closer segment:** tracks ordered descending by `t.duration_ms` (longest first), then ascending by `artist_rotation_key`. Take the top `closer` count from the remaining pool.
- **Core segment:** the remaining tracks, ordered ascending by `artist_rotation_key` (rotating artists through the middle).
- A single track cannot be in more than one segment; assignment proceeds opener → closer → core, in that order, and each assignment consumes the track from the shared pool.
- Intra-segment order is the same order used to pick segment members (no re-sort), so the defined keys fully determine final order. This is deterministic given `library_snapshot` and `VibeProfile`.
- This fallback is documented in the run log under `segmentation_mode = "metadata_fallback"`.

**Why:** URI dedup is cheap and correct for the common case; the precisely specified `normalized_title` catches the 'same song, different release' problem reproducibly. A fully specified score function and fallback segmentation algorithm mean two implementers produce the same playlist in both `features_mode=true` and `features_mode=false`.

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
      │    Client     │           │ (vibe → profile) │          │ (SQLite + runs + │
      │ (clock/tx     │           │ (cache, pinned   │          │  llm-cache)      │
      │  injectable)  │           │  sampling)       │          │                  │
      └───────────────┘           └──────────────────┘          └──────────────────┘
```

**Modules:**
- `auth` — PKCE flow, token storage, refresh, redirect-port negotiation, fallback-trigger detection.
- `spotify` — typed client wrapper over Web API with the retry/throttle policy from §4.4; accepts injectable clock and transport for tests; owns the capability probe and its canary URI constant.
- `library` — fetch + cache user playlists, tracks, and (optionally) audio features; emits `library_snapshot`.
- `vibe` — LLM prompt/response schema, pinned sampling, response cache, URI validation, unresolved-URI policy.
- `rank` — scoring (§4.5.1), dedup (including `normalized_title`), segmented ordering (§4.5.2 and its metadata fallback).
- `publish` — creates the Spotify playlist (with reconciliation on 5xx, §4.4) and writes the run log.
- `cli` — user-facing entry point; hosts `vibeify runs prune`.

## 6. Data Shapes

**VibeProfile** (LLM output, validated):
```json
{
  "target_features": {
    "energy": [0.3, 0.6],
    "valence": [0.2, 0.5],
    "tempo_bpm": [70.0, 100.0],
    "danceability": [0.0, 0.5],
    "acousticness": [0.3, 1.0],
    "instrumentalness": [0.0, 0.4]
  },
  "year_range": [1990, 2025],
  "seed_artists": ["spotify:artist:..."],
  "seed_genres": ["indie-folk", "ambient"],
  "mood_keywords": ["rainy", "melancholic", "late-night"],
  "exclusions": { "artist_ids": [], "genres": ["metal"] }
}
```

Notes:
- `tempo_bpm` values are floats; integers are coerced to float at the validation boundary.
- `exclusions` is always the object form `{ artist_ids: [], genres: [] }` — this is the single canonical shape referenced everywhere (§4.2, §4.5, §6).
- `target_features` is optional when `features_mode=false`; if present under that mode it is ignored with a logged warning.
- `year_range` is optional.
- `mood_keywords` is optional; an empty array causes `mood_score` to be zero for all tracks (see §4.5.1).

**RunLog entry** (local, append-only):
```json
{
  "run_id": "2026-04-20T19-02-11Z-ab12",
  "prompt": "rainy sunday morning, coffee, no lyrics",
  "llm": {
    "model": "claude-opus-4-7",
    "temperature": 0,
    "top_p": 1,
    "seed": 42,
    "seed_honored": true,
    "prompt_hash": "...",
    "response_hash": "...",
    "schema_version": "1"
  },
  "capability": {
    "features_mode": true,
    "probed_at": "2026-04-20T19-01-50Z",
    "probe_status": 200,
    "probe_canary_uri": "spotify:track:4iV5W9uYEdYUVa79Axb7Rh",
    "mid_run_degradation": false
  },
  "library_snapshot": {
    "playlist_ids": ["..."],
    "playlist_snapshot_ids": {"...": "..."},
    "market": "from_token",
    "candidate_summary_hash": "...",
    "resolved_artist_ids": ["..."]
  },
  "summary_truncation": { "artists_dropped": 0, "genres_dropped": 0 },
  "sources": { "playlist_ids": ["..."], "included_liked": false, "candidate_count": 1843 },
  "unresolved_uris": {
    "seed_artists": { "total": 6, "unresolved": 1, "aborted": false },
    "exclusions_artist_ids": { "total": 0, "unresolved": 0, "aborted": false }
  },
  "segmentation_mode": "energy",
  "selected": [{ "uri": "spotify:track:...", "score": 0.87, "components": { "f": 0.9, "a": 1.0, "g": 0.4, "m": 0.5, "y": 1.0 }, "segment": "core" }],
  "result": { "playlist_id": "...", "url": "https://open.spotify.com/playlist/..." }
}
```

## 7. Failure Modes & Handling

| Failure | Handling |
| --- | --- |
| User declines OAuth consent | Abort with clear message; no partial state. |
| Keychain unusable and no fallback trigger met | Abort with actionable message; user can retry after unlocking keychain or pass `--no-keychain`. |
| All registered redirect ports (8898–8901) in use | Abort with actionable error listing the ports and suggesting the user close whichever local server holds them. |
| No source playlists / empty library | Refuse to generate; prompt user to add source playlists or enable Liked Songs. |
| Capability probe returns 401 after refresh | Abort as an auth failure (not a capability failure). |
| Capability probe returns 403/404 | Log `features_mode=false`, continue with metadata-only scoring. |
| Capability probe exhausts retries on 429/5xx | Abort the run before any LLM spend. |
| `/audio-features` returns 403/404 mid-run | Discard partially-fetched features; degrade to `features_mode=false` for the rest of the run; record degradation. |
| LLM returns invalid/unparseable JSON | Retry once with stricter schema reminder; on second failure, abort with the raw response logged locally. |
| LLM emits URIs that don't resolve | Apply §4.2.2 policy independently to `seed_artists` and `exclusions.artist_ids`: warn if set has <3 URIs; abort that run if unresolved > 50% of a set with ≥3 URIs. |
| Candidate pool < requested length | Emit warning, publish what we have. |
| Playlist create returns 5xx | Do not retry the POST; run reconciliation (§4.4); on zero or multiple matches, abort cleanly without creating a duplicate. |
| Playlist track-add batch fails with 5xx | Run the length-based reconciliation (§4.4); retry only if the batch was not applied. On any other result, abort cleanly leaving the partial playlist recorded in the run log. |
| Refresh token revoked | Delete stored token, re-run auth flow. |

## 8. Non-Goals / Explicit Deferrals

- No web frontend in v0.3.
- No server-side component; fully local.
- No cross-account features.
- No automatic regeneration / scheduling.
- No analytics or telemetry beyond the local run log.
- No embedding-based mood matching; keyword/token matching only.

## 9. Testing Strategy

Tests are a spec-level requirement, not a build-phase afterthought. The following layers MUST exist before v0.3 ships:

1. **HTTP fixture / VCR layer for Spotify.** All Spotify calls go through the wrapper in `spotify/`. Tests use a scripted transport that replays recorded cassettes and a fake clock. Coverage includes: paginated `/me/playlists`, paginated `/playlists/{id}/tracks` with `market=from_token`, batched `/audio-features`, batched `/artists`, `POST /users/{id}/playlists`, batched `POST /playlists/{id}/tracks`, 401-refresh path, 429 with both `delta-seconds` and HTTP-date `Retry-After`, 5xx on GET, 5xx on playlist-create (asserts no blind retry + reconciliation via `P_before`/`P_after` set diff + name/description match with zero-match, one-match, and multi-match variants), 5xx on track-add (asserts length-based reconciliation with applied, not-applied, and mismatch variants). Cassettes are scrubbed of tokens before commit.

2. **Retry / rate-limit policy tests.** With a fake clock and fake transport: assert sleep duration equals `Retry-After` + [0, 500 ms]; assert max-attempts boundary (5); assert 4xx-non-429 is not retried; assert `POST /users/{id}/playlists` is not retried on 5xx and reconciliation executes; assert `POST /playlists/{id}/tracks` runs length-based reconciliation on 5xx and retries only when the batch was not applied; assert client-side token bucket behavior with both dimensions: cold-start 10-request burst is admitted without delay, and 11th request blocks for ~100 ms (capacity=10, refill=10/s); assert `features_mode` flip on 403/404 at probe and mid-run.

3. **VibeProfile validation tests.** Golden-file tests for: well-formed JSON accepted; malformed JSON triggers the single retry defined in §7; integer `tempo_bpm` is coerced to float; wrong `exclusions` shape (array form) is rejected at validation; empty `mood_keywords` validates and produces `mood_score=0`; partial unresolved URIs with a set of 2 warn and proceed; partial unresolved with a set of 6 and 2 unresolved warns and proceeds; 4-of-6 unresolved aborts with a run-log entry; scope independence — aborting on `seed_artists` does not depend on `exclusions.artist_ids` and vice versa.

4. **Scorer & ordering tests.** Deterministic unit tests against a fixed candidate fixture: assert `feature_score`, `artist_score`, `genre_score`, `mood_score` (including the hyphenated-genre tokenization path, e.g., `indie-folk` producing `{indie, folk}`), `year_score` values for known inputs; assert segment assignment (opener/core/closer counts and per-segment targets 0.10/0.00/0.20 below midpoint in `features_mode=true`); assert tie-break order (source-playlist count → added_at → URI lex); assert `features_mode=false` re-normalizes weights and uses the metadata-fallback segmentation; assert the fallback's `artist_distinctiveness` and `artist_rotation_key` produce the specified round-robin on a fixture with 3 artists at varying candidate counts.

5. **Dedup tests.** URI dedup, `(normalized_title, primary_artist_id)` fallback (with specific fixtures for parenthetical stripping, trailing-dash qualifier stripping, NFKD normalization — e.g., `"Cafe\u0301"` matches `"café"` — and punctuation collapse), local-track and unplayable-track exclusion, `--clean` behavior, exclusions object shape.

6. **Run-log replay test (scoring/ordering determinism).** Given a stored run log + a frozen `library_snapshot` + a cached LLM response, re-running the pipeline produces the exact same selected URIs in the exact same order. This test guards the scoring/ordering half of the §4.5 determinism claim. It explicitly does not test LLM seed honoring (it bypasses the LLM via the cache).

7. **LLM adapter seed-honoring test.** Separate from test 6: against a fake LLM provider whose responses depend on the seed parameter, assert that (a) the adapter passes `seed` through on every call, (b) `temperature=0` and `top_p=1` are pinned, (c) provider metadata indicating seed support maps to `seed_honored=true` in the run log, (d) provider metadata indicating no seed support maps to `seed_honored=false`, and (e) a run against a non-seed-honoring provider still completes successfully with the best-effort determinism claim documented in the run log. This test is what surfaces seed-plumbing regressions in the LLM adapter.

8. **Capability-regression contract test.** A test that runs the startup probe against fixtures covering all probe outcomes (200, 401→refresh→200, 401→refresh→401, 403, 404, 429-then-200, 500-then-200, retries exhausted) and asserts the correct `features_mode` and run behavior for each. The `features_mode=false` branch uses a candidate-library fixture with **declared minimums**: at least 60 playable tracks (`is_playable=true`) across at least 8 distinct primary artists spanning at least 5 genres, with at least 20 tracks whose title or album contains at least one of the fixture's mood keywords. The test asserts the run produces a playlist of exactly `--length` tracks (defaulting to 30) under `features_mode=false`, not merely non-empty. Fixture declarations are in-repo under `tests/fixtures/capability_regression/README.md`.

9. **Cache eviction tests.** `cache.db` eviction: assert expired-TTL pass runs first; assert LRU pass only runs if still over cap; assert read access bumps recency. `llm-cache` eviction: assert read bumps recency; assert LRU eviction fires only when directory size exceeds 500 MB.

10. **Run-log retention tests.** Assert `vibeify runs prune --keep N` keeps the newest-by-timestamp N run logs and deletes the rest; assert off-by-one — `--keep 0` deletes all, `--keep 1` keeps exactly one; assert auto-prune fires at startup when `>1 000` logs are present and reduces to 200; assert the in-flight run's log is written after the auto-prune pass and is therefore never deleted by it; assert auto-prune does not fire at ≤1 000 logs.

11. **Fallback-trigger tests.** With a mocked keychain API and environment: assert the on-disk fallback is used when the keychain errors after one retry; when the environment is headless (no TTY + no display vars); when `--no-keychain` is passed. Assert that fallback is NOT used when the keychain is simply locked but promptable — the run aborts with an actionable message instead.

## 10. Open Questions (resolved or deferred)

All interview items are resolved above. Remaining implementation-level questions (exact scoring weights beyond defaults, exact LLM prompt wording, exact cache-size thresholds, rotation of the capability-probe canary URI if it is ever removed from Spotify's catalog) are left to the build phase and will be tunable via config or the `vibe/canary.py` constant. If Spotify later restores broad `/audio-features` access or introduces an idempotency-key header for playlist create, or adds a `position` parameter that makes track-add genuinely idempotent without length-based reconciliation, the capability probe and the POST recovery paths in §4.4 can be relaxed accordingly.
