# SPEC — vibeify (v0.2)

## 1. Persona

Veteran Spotify Web API playlist engineer. Deep familiarity with OAuth 2.0 authorization flows, the Web API track/playlist endpoints, rate-limit behavior (429 + Retry-After), and the practical quirks of building user-facing music tooling against Spotify as the system of record — including the post-2024 endpoint restrictions that removed `/audio-features`, `/audio-analysis`, `/recommendations`, related-artists, and 30-second previews from new/non-extended client IDs.

## 2. Idea

vibeify helps a user create Spotify playlists from a free-text vibe/mood prompt, seeded and shaped by the music the user already listens to (tracks pulled from their existing playlists). The output is a new Spotify playlist in the user's own account, cohesive with the requested vibe but anchored in the user's taste.

## 3. Scope (v0.2)

### In scope
- Single-user CLI (or minimal local web app) targeting one authenticated Spotify account.
- Pull candidate tracks from the user's existing playlists (owned + followed).
- Accept a free-text vibe prompt; use an LLM to translate vibe → a structured vibe profile over **metadata-only signals** (artist, album, genre, track name, release year, explicit flag, duration) with audio-feature signals as an **optional, capability-gated** enhancement.
- Rank/select tracks from the user's library against the LLM-derived vibe profile.
- Create a new playlist in the user's Spotify account with the selected tracks.
- Persist a local run log (prompt, seed inputs, selected track URIs, resulting playlist ID) and cached LLM response for reproducibility.
- Ship a test suite that validates the auth, retry, scoring, dedup, ordering, and publish paths without live Spotify calls.

### Out of scope (v0.1/0.2)
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

All four scopes are requested up-front on first consent. `--include-liked` at runtime is a behavioral toggle, not a scope toggle; we do not perform scope upgrades mid-session.

**Token storage:** Refresh token stored in the OS keychain where available. Fallback files are OS-specific and follow platform conventions:
- macOS: Keychain (primary); fallback `~/Library/Application Support/vibeify/token.json` (0600).
- Linux: libsecret (primary); fallback `$XDG_CONFIG_HOME/vibeify/token.json` or `~/.config/vibeify/token.json` (0600).
- Windows: Credential Manager (primary); fallback `%APPDATA%\vibeify\token.json` (ACL: current user only).
Never committed; never logged.

**Redirect URI:** Spotify requires exact-match registration including port, so we pin a small, registered set. Primary: `http://127.0.0.1:8898/callback`. Fallbacks (tried in order if 8898 is busy): `8899`, `8900`, `8901`. All four are registered in the Spotify app dashboard. If all four are in use, abort with an actionable error (§7). The loopback listener is bound only for the duration of the auth dance.

### 4.2 playlist_source — LLM-mediated vibe prompt over user's library (metadata-first)

**Decision (confirmed, revised for API reality):** Free-text vibe prompt → LLM → vibe profile (seed artists/genres + target metadata signals) → rank user's existing playlist tracks against that profile → select final tracklist. Audio-feature scoring is **optional and capability-gated**: vibeify probes at startup whether the authenticated app has access to `/audio-features`; if and only if the probe succeeds, feature-distance scoring is enabled as an additional score component.

**Pipeline:**
1. Fetch the user's playlists via `GET /me/playlists` (paginated). If `--include-liked` is set, also fetch `GET /me/tracks`.
2. Collect track URIs from each source via `GET /playlists/{id}/tracks` (and `/me/tracks`). All track-reading endpoints are called with `market=from_token` so `is_playable` is populated. De-duplicate into a candidate pool.
3. **Capability probe:** one `GET /audio-features?ids=<single-known-track>` call. On 200 → enable `features_mode=true`. On 401 → refresh and retry once. On 403/404 → `features_mode=false`, log the capability decision in the run log, and proceed with metadata-only scoring. This probe result is cached per app+token for 24 h.
4. If `features_mode=true`, fetch audio features in batches of 100 via `GET /audio-features`. On 403/404 mid-run (Spotify revoking access), degrade to `features_mode=false` for the remainder of the run and record the degradation in the run log.
5. Build the LLM input: prompt + compact candidate-pool summary (see §4.2.1). Send to the LLM with `temperature=0` and any provider-specific seed. LLM returns structured JSON matching the VibeProfile schema in §6.
6. Validate the VibeProfile: JSON-schema check; resolve every LLM-emitted `spotify:artist:...` URI via batched `GET /artists?ids=` (50/req); drop unresolved URIs with a warning, do not abort unless >50% are unresolved.
7. Score each candidate track per §4.5.1. Apply exclusions.
8. Select the top N (default 30, configurable 10–100) subject to the dedup/ordering rules in §4.5.

**4.2.1 LLM context-window strategy.** The candidate-pool summary sent to the LLM is capped at 8 000 tokens and composed of: top 50 artists by track count; genre histogram bucketed to the top 30 genres; release-year histogram in 5-year bins; track-count, explicit-ratio, mean-duration, and (if `features_mode=true`) per-feature min/p25/p50/p75/max. Individual track titles are not sent. When the full summary exceeds 8 000 tokens, artist/genre lists are truncated first; the truncation counts are recorded in the run log.

**LLM:** Pluggable. Default `claude-opus-4-7`. Sampling is pinned: `temperature=0`, `top_p=1`, provider seed recorded when supported (with an explicit note in the run log for providers that do not honor seeds — determinism in §4.5 is conditional on seed honoring). System prompt and response schema are versioned in-repo; the `prompt_hash` is the SHA-256 of `system_prompt || user_prompt || schema_version`. LLM responses are cached locally keyed by `prompt_hash` so repeat runs short-circuit the LLM call.

### 4.3 persistence — Spotify is the system of record; local run log + LLM cache for reproducibility

**Decision:** The generated playlist is created **directly in the user's Spotify account** (via `POST /users/{user_id}/playlists` + `POST /playlists/{id}/tracks`). Spotify is the source of truth for the playlist itself.

Locally, vibeify keeps:
- An append-only run log at `<data_dir>/runs/<timestamp>.json` containing prompt text, LLM model ID, prompt/response hashes, seed, sampling params, capability-probe result, source playlist IDs, truncation stats, candidate count, selected track URIs (ordered) with per-track score breakdowns, resulting playlist ID + URL.
- An LLM response cache at `<data_dir>/llm-cache/<prompt_hash>.json` (content-addressed, never expired; evicted by LRU when the cache exceeds 500 MB).
- A SQLite cache at `<data_dir>/cache.db` storing track metadata and (if applicable) audio features keyed by track URI. Audio features and core track metadata are treated as **immutable**; they are cached indefinitely and evicted only by LRU when the DB exceeds 200 MB. Playlist membership and `is_playable` are cached with a 24 h TTL because they are user/market-dependent.
- `<data_dir>` resolution: Linux `$XDG_DATA_HOME/vibeify` or `~/.local/share/vibeify`; macOS `~/Library/Application Support/vibeify`; Windows `%LOCALAPPDATA%\vibeify`.

**Run-log retention:** a `vibeify runs prune --keep N` command (default keeps the last 200 runs) and an auto-prune that runs at startup when >1 000 run logs are present.

**Why:** Creating the playlist in Spotify directly matches user expectation and avoids building a parallel UI. The local log + LLM cache enables re-runs, debugging, and A/B comparison across prompts. Treating audio features as immutable reflects Spotify's data model and avoids gratuitous re-fetching.

### 4.4 rate_limit_strategy — Respect Retry-After, bounded exponential backoff, client-side throttling, testable wrapper

**Decision:**
- All Spotify API calls go through a single HTTP client wrapper. The wrapper accepts an **injectable clock and transport** so every policy below is unit-testable with a fake clock and a scripted transport (see §9).
- On **429**: read `Retry-After`. Per RFC 7231 this may be either `delta-seconds` (integer) or an HTTP-date. Parse both forms; on HTTP-date, compute `max(0, date − now)`. On unparseable or missing header, use 1 s. Sleep that duration plus 0–500 ms uniform jitter; retry up to 5 times; after that, surface the error.
- On **5xx**: exponential backoff — 1, 2, 4, 8, 16 s plus 0–500 ms jitter; max 5 retries.
- On **401**: refresh the access token once and retry; if still 401, re-prompt the user to re-authenticate.
- On **403/404** on capability-gated endpoints (`/audio-features`, `/recommendations`): never retry; flip the relevant capability flag (§4.2 step 3/4).
- Client-side token bucket: default ceiling of 10 requests/second per authenticated user, tunable via config.
- Batch endpoints used wherever possible: `/audio-features` (100/req), `/tracks` (50/req), `/artists` (50/req), playlist track adds (100/req).
- Idempotency rules:
  - GET requests retry on 429 and 5xx.
  - PUT/DELETE retry on 429 and 5xx (Spotify's playlist mutation endpoints are naturally idempotent at the batch granularity used here).
  - **POST retries on 429 only.** POSTs are **not** retried on 5xx by default, because `POST /users/{user_id}/playlists` has no idempotency key and a 5xx-then-retry would produce duplicate empty playlists. For the specific case of playlist creation, on 5xx we instead perform a single reconciliation: `GET /me/playlists` (first page), look for a playlist whose name+description matches ours and whose `snapshot_id` implies it was created within the last 60 s; if found, reuse it, otherwise fail the run cleanly.
  - Playlist track-add POSTs (`POST /playlists/{id}/tracks`) are retried on 5xx because batch adds are idempotent when the client replays the same 100-URI window from the last successful offset.
- Never retry 4xx other than 429.

**Why:** Honoring `Retry-After` is required by Spotify's guidance and avoids app-level bans. The POST-on-5xx carve-out prevents duplicate playlists, which is the realistic failure mode for the create call. Making the clock and transport injectable is what lets §9 test any of this.

### 4.5 track_dedup_scope — URI-level dedup, canonical preference, stable ordering

**Decision:**
- **Dedup key:** Spotify track URI. A secondary pass uses `(normalized_title, primary_artist_id)` to catch re-releases/remasters. In a collision, prefer the track with the highest **source-playlist count** (i.e., appears in the most of the user's source playlists); ties broken by most-recent `added_at` across those playlists, then by track URI lexicographic order. (Renamed from 'most-played' in v0.1 — the proxy is source-playlist coverage, not play count, which is not exposed by the Web API.)
- **Exclusions:** Drop local tracks (`is_local: true`), unplayable tracks (`is_playable: false` — all track reads use `market=from_token`, so this field is populated), and tracks matching `exclusions.artist_ids` or `exclusions.genres` from the VibeProfile.
- **Ordering:** See §4.5.2.
- **Length:** Default 30 tracks; `--length N` overrides (clamped 10–100). If the candidate pool after filtering is smaller than requested, emit a warning and use the full filtered pool.
- **Explicit content:** Include by default; `--clean` flag excludes tracks marked `explicit: true`.
- **Determinism:** Output is deterministic for a given `(prompt_hash, library snapshot, model, seed)` quadruple **provided the LLM provider honors the seed at temperature=0**. Providers that do not are flagged in the run log and the determinism claim is weakened to best-effort for those providers.

**4.5.1 Scoring function (form specified).** Each surviving candidate track `t` receives a score in `[0, 1]`:

`score(t) = w_f · feature_score(t) + w_a · artist_score(t) + w_g · genre_score(t) + w_m · mood_score(t) + w_y · year_score(t)`

with default weights `w_f=0.35, w_a=0.25, w_g=0.15, w_m=0.15, w_y=0.10` when `features_mode=true`; when `features_mode=false`, `w_f=0` and the remaining weights are renormalized to sum to 1. All weights are config-tunable; the exact tuned values are a build-phase concern.

Component definitions:
- `feature_score(t)`: mean over the features listed in `target_features` of `range_fit(v, [lo, hi])`, where `range_fit` returns `1.0` if `lo ≤ v ≤ hi`, else `max(0, 1 − |v − nearest_bound| / scale_f)` with per-feature `scale_f` (energy/valence/danceability/acousticness/instrumentalness: `0.3`; tempo_bpm: `30`). Tempo is a float throughout; any integer bounds in the VibeProfile are coerced to float at validation time.
- `artist_score(t)`: `1.0` if any of `t`'s artist IDs ∈ `seed_artists`, else `0.5` if any of `t`'s artist IDs share ≥1 genre with any seed-artist (lookup via cached `/artists`), else `0.0`.
- `genre_score(t)`: Jaccard overlap of `t`'s artist-genres (union across all of `t`'s artists) with `seed_genres`.
- `mood_score(t)`: token-level match of `mood_keywords` against the lowercased concatenation of `t.name + ' ' + t.album.name + ' ' + joined-artist-genres`. Scoring is `matches / len(mood_keywords)`, clamped to `[0, 1]`. Tokenization is simple whitespace+punctuation split; no embeddings in v0.2.
- `year_score(t)`: `1.0` unless the VibeProfile contains a `year_range` field, in which case `range_fit` over `t.album.release_year` with `scale_y = 10`.

**4.5.2 Segmented ordering (opener / core / closer).** The final N tracks are split into three segments by count — `opener = ceil(0.2·N)`, `closer = ceil(0.2·N)`, `core = N − opener − closer`. Each segment has a target energy level derived from the VibeProfile's `target_features.energy` midpoint `m`: opener target `max(0, m − 0.15)`, core target `m`, closer target `max(0, m − 0.25)`. (If `features_mode=false`, segments are assigned by artist rotation instead: opener = most-distinctive artists for the vibe; core = mixed; closer = longest tracks. The fallback is documented in the run log.) Within each segment, tracks are ordered ascending by `|track_energy − segment_target|` (or in fallback: ascending by artist-rotation key). Assignment of tracks to segments is done greedily from the top-scored pool: each track is assigned to the segment whose target it best fits, subject to the segment's capacity; ties broken by overall score descending, then by track URI lexicographic order.

**Why:** URI dedup is cheap and correct for the common case; the title/artist fallback catches the 'same song, different release' problem. A precisely specified score function and segmentation algorithm mean two implementers produce the same playlist.

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
- `auth` — PKCE flow, token storage, refresh, redirect-port negotiation.
- `spotify` — typed client wrapper over Web API with the retry/throttle policy from §4.4; accepts injectable clock and transport for tests; owns the capability probe.
- `library` — fetch + cache user playlists, tracks, and (optionally) audio features.
- `vibe` — LLM prompt/response schema, pinned sampling, response cache, URI validation.
- `rank` — scoring (§4.5.1), dedup, segmented ordering (§4.5.2).
- `publish` — creates the Spotify playlist (with reconciliation on 5xx, §4.4) and writes the run log.
- `cli` — user-facing entry point.

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
  "capability": { "features_mode": true, "probed_at": "2026-04-20T19-01-50Z" },
  "summary_truncation": { "artists_dropped": 0, "genres_dropped": 0 },
  "sources": { "playlist_ids": ["..."], "included_liked": false, "candidate_count": 1843 },
  "selected": [{ "uri": "spotify:track:...", "score": 0.87, "components": { "f": 0.9, "a": 1.0, "g": 0.4, "m": 0.5, "y": 1.0 }, "segment": "core" }],
  "result": { "playlist_id": "...", "url": "https://open.spotify.com/playlist/..." }
}
```

## 7. Failure Modes & Handling

| Failure | Handling |
| --- | --- |
| User declines OAuth consent | Abort with clear message; no partial state. |
| All registered redirect ports (8898–8901) in use | Abort with actionable error listing the ports and suggesting the user close whichever local server holds them. |
| No source playlists / empty library | Refuse to generate; prompt user to add source playlists or enable Liked Songs. |
| `/audio-features` returns 403/404 at probe | Log `features_mode=false`, continue with metadata-only scoring. |
| `/audio-features` returns 403/404 mid-run | Degrade to `features_mode=false` for the rest of the run; record degradation; re-score already-fetched tracks under the new weights. |
| LLM returns invalid/unparseable JSON | Retry once with stricter schema reminder; on second failure, abort with the raw response logged locally. |
| LLM emits seed_artist URIs that don't resolve | Drop individual unresolved URIs with a warning; abort only if >50% fail to resolve. |
| Candidate pool < requested length | Emit warning, publish what we have. |
| Playlist create returns 5xx | Do not retry the POST; run reconciliation (§4.4) against `/me/playlists`; if no match, abort cleanly without creating a duplicate. |
| Playlist track-add batch fails | Retry the specific 100-URI batch per §4.4; on exhaustion, stop and leave the partial playlist with its final state recorded in the run log. |
| Refresh token revoked | Delete stored token, re-run auth flow. |

## 8. Non-Goals / Explicit Deferrals

- No web frontend in v0.2.
- No server-side component; fully local.
- No cross-account features.
- No automatic regeneration / scheduling.
- No analytics or telemetry beyond the local run log.
- No embedding-based mood matching; keyword/token matching only.

## 9. Testing Strategy

Tests are a spec-level requirement, not a build-phase afterthought. The following layers MUST exist before v0.2 ships:

1. **HTTP fixture / VCR layer for Spotify.** All Spotify calls go through the wrapper in `spotify/`. Tests use a scripted transport that replays recorded cassettes and a fake clock. Coverage includes: paginated `/me/playlists`, paginated `/playlists/{id}/tracks` with `market=from_token`, batched `/audio-features`, batched `/artists`, `POST /users/{id}/playlists`, batched `POST /playlists/{id}/tracks`, 401-refresh path, 429 with both `delta-seconds` and HTTP-date `Retry-After`, 5xx on GET, 5xx on playlist-create (asserts no retry + reconciliation), 5xx on track-add (asserts retry). Cassettes are scrubbed of tokens before commit.

2. **Retry / rate-limit policy tests.** With a fake clock and fake transport: assert sleep duration equals `Retry-After` + [0, 500 ms]; assert max-attempts boundary (5); assert 4xx-non-429 is not retried; assert POST is not retried on 5xx for `/users/{id}/playlists`; assert POST is retried on 5xx for `/playlists/{id}/tracks`; assert client-side token bucket rate-limits to 10 req/s; assert `features_mode` flip on 403/404.

3. **VibeProfile validation tests.** Golden-file tests for: well-formed JSON accepted; malformed JSON triggers the single retry defined in §7; integer `tempo_bpm` is coerced to float; wrong `exclusions` shape (array form) is rejected at validation; partial unresolved `seed_artists` warn; >50% unresolved aborts.

4. **Scorer & ordering tests.** Deterministic unit tests against a fixed candidate fixture: assert `feature_score`, `artist_score`, `genre_score`, `mood_score`, `year_score` values for known inputs; assert segment assignment (opener/core/closer counts and per-segment targets); assert tie-break order (source-playlist count → added_at → URI lex); assert `features_mode=false` re-normalizes weights and uses the artist-rotation fallback for segments.

5. **Dedup tests.** URI dedup, `(normalized_title, primary_artist_id)` fallback, local-track and unplayable-track exclusion, `--clean` behavior, exclusions object shape.

6. **Run-log replay test.** Given a stored run log + a frozen library snapshot + a cached LLM response, re-running the pipeline produces the exact same selected URIs in the exact same order. This test guards the §4.5 determinism claim.

7. **Capability-regression contract test.** A test that runs the startup probe against a fixture where `/audio-features` returns 403/404 and asserts the run completes with `features_mode=false` and a non-empty playlist. This catches future Spotify endpoint restrictions.

## 10. Open Questions (resolved or deferred)

All interview items are resolved above. Remaining implementation-level questions (exact scoring weights beyond defaults, exact LLM prompt wording, exact cache-size thresholds) are left to the build phase and will be tunable via config. If Spotify later restores broad `/audio-features` access or introduces an idempotency-key header for playlist create, the capability probe and the POST-retry carve-out in §4.4 can be relaxed accordingly.
