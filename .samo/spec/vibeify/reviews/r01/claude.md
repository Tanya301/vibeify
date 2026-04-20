# Reviewer B — Claude

## summary

The spec is well-organized and the auth/rate-limit/dedup decisions are individually sensible, but it has several blocking issues a veteran Spotify engineer should have caught: the entire scoring core depends on audio-features and seed-recommendations endpoints that Spotify has restricted for new clients, with no fallback. Determinism is asserted without pinning LLM sampling, the redirect URI scheme is incompatible with Spotify's exact-match registration, playlist-create retry on 5xx will produce duplicate empty playlists, and the exclusions schema is contradictory between §4.2 and §6. The scoring function's *form* (not just weights) and the segment-ordering algorithm are underspecified enough that two implementers would build different products. Most critically, the spec contains zero testing strategy: no fixture/VCR layer for Spotify, no harness for the retry policy, no golden tests for VibeProfile validation, no replay test for the reproducibility claim. Resolve the API-deprecation question first (it may invalidate §4.2 and §6 entirely), then tighten determinism, idempotency, and testability before v0.2.

## contradiction

- (major) Spotify API deprecations not acknowledged. The pipeline (§4.2 step 3, step 4 'seed_artists/seed_genres', §6 VibeProfile) leans heavily on `/audio-features` and seed-based recommendation primitives. As of late 2024, Spotify removed access to `/audio-features`, `/audio-analysis`, `/recommendations`, related-artists, and 30-second previews for new/non-extended Web API client IDs. A veteran Spotify engineer would not write a v0.1 spec whose entire scoring core (`target_features`, weighted feature distance) depends on an endpoint new apps cannot call. The spec must either declare the app is grandfathered (with evidence) or define a fallback scoring path that does not require audio-features.
- (major) `VibeProfile.exclusions` shape is inconsistent. §4.2 step 4 lists `exclusions: [...]` (flat array) while §6 specifies `exclusions: { artist_ids: [], genres: [] }` (object). §4.5 also references LLM-derived 'exclusions' without disambiguating. Pick one shape and apply it everywhere, or the validator/scorer/LLM-prompt schema will disagree.
- (major) Determinism claim conflicts with LLM usage. §4.5 asserts output is 'Deterministic for a given (prompt, library snapshot, model) triple,' and §4.3 says the run log enables 'reproducibility,' but the spec never pins LLM temperature, top_p, seed, or system-prompt hash, nor does it cache the LLM response keyed by prompt_hash to short-circuit re-runs. With default sampling, the same triple yields different VibeProfiles. Either specify temperature=0 + a recorded seed (and note providers that don't honor seeds), or weaken the determinism claim.

## weak-testing

- (major) No testing strategy whatsoever. The spec defines an OAuth dance, a paginated fetch pipeline, a 429/5xx retry policy, an LLM JSON contract, a scoring/ranking algorithm, and idempotent batched writes — and says nothing about how any of it is verified. At minimum the spec should require: (a) a recorded-fixture/VCR layer for Spotify HTTP so retry, pagination, and 401-refresh paths are tested without live calls; (b) golden-file tests for VibeProfile JSON-schema validation including the malformed-LLM retry path in §7; (c) deterministic unit tests for the scorer and dedup/segment-ordering rules in §4.5 against a fixed candidate fixture; (d) a contract test that fails loudly when a Spotify endpoint the pipeline depends on returns 403/404 (catches the audio-features deprecation regression). Without this, none of the §4.4/§4.5/§7 guarantees are checkable.
- (major) §4.4's retry/idempotency rules have no acceptance criteria. 'Retry the same request up to 5 times,' '0–500 ms jitter,' '10 req/s token bucket,' and 'POST/PUT/DELETE retried only on 429 and 5xx' are all asserted but the spec does not say how they are validated (e.g., a fake clock + fake transport that injects 429 with `Retry-After`, asserts sleep duration within jitter window, asserts max-attempts boundary, asserts 4xx-non-429 is not retried). Add a testability requirement: the HTTP wrapper must accept an injectable clock and transport so these policies are unit-testable.
- (minor) No reproducibility test for the run-log claim. §4.3 says the log enables 'A/B comparison across prompts' and §4.5 claims determinism. There is no proposed harness for replaying a stored run-log against a frozen library snapshot and asserting the same selected URIs come out. Without a replay test, the reproducibility claim is undemonstrated.

## ambiguity

- (major) Playlist-create idempotency under 5xx retry is undefined. §4.4 permits retrying POST on 5xx; §7 says 'Playlist is created first (empty), tracks added in idempotent 100-track batches; on failure, retry from last successful batch.' But `POST /users/{user_id}/playlists` has no idempotency key in the Spotify API — a 5xx response after the server created the playlist, retried, will produce two empty playlists in the user's account on every transient failure. Specify either (a) do not retry the create call on 5xx, only the track-add calls, or (b) on retry, first list recent playlists and reuse a name+description match created within the last N seconds.
- (major) Redirect URI scheme is incompatible with Spotify's exact-match requirement. §4.1 specifies `http://127.0.0.1:<ephemeral-port>/callback`. Spotify requires every redirect URI to be registered verbatim in the app dashboard, including port. The spec must either (a) pin a fixed port (and document fallback behavior if it's busy), or (b) register a small set of known ports and pick from them, or (c) register a single port and document the contention failure mode in §7. As written, the auth flow will fail for any port not pre-registered.
- (major) Scoring function is underspecified. §4.2 step 5 says 'weighted distance across target audio features + artist/genre match + mood-keyword match on track/album metadata.' §6 expresses `target_features` as `[min, max]` ranges, but 'distance' is undefined for ranges (distance to midpoint? zero inside the range? penalty proportional to overshoot?). 'mood-keyword match on track/album metadata' is also ambiguous — track name, album name, artist genres, or all three, and with what matching (substring, token, embedding)? §9 defers 'exact scoring weights' but the *form* of the function is also missing and is not just a tuning knob.
- (minor) Secondary dedup tiebreak proxy is misaligned with its name. §4.5 says 'prefer the track already present in the user's most-played source playlists (proxy: track appears in the most playlists).' 'Most-played' and 'appears in the most playlists' are different signals — a track in five rarely-opened playlists is not 'most-played.' Either rename the criterion to 'most-saved-across-playlists' or specify a different proxy (e.g., recently-added, owned-vs-followed weighting).
- (major) Segmented ordering (opener/core/closer) is named but not defined. §4.5 says the final list is sorted 'within 3 segments' by 'LLM-derived target-feature distance' but never says how segment boundaries are computed (equal thirds by count? by energy/valence trajectory? does the LLM emit a per-segment target?), nor which feature defines the arc. Without this, the ordering is neither deterministic nor reviewable.
- (major) `is_playable` requires a market parameter that is not specified. §4.5 drops tracks with `is_playable: false` 'in user's market,' but the spec never says which Spotify endpoints are called with `market=from_token` (or the user's country code from `GET /me`). Without this, `is_playable` is absent from many responses and the filter silently no-ops or drops everything.
- (minor) LLM context-window strategy for large libraries is missing. §4.2 step 4 sends a 'compact summary of the candidate pool' to the LLM, but a user with thousands of unique tracks/artists has no defined truncation, sampling, or summarization rule. Specify the maximum payload (token budget), the summarization shape (e.g., top-K artists by count, genre histogram bucketed to N), and what is dropped when over budget.
- (minor) LLM-emitted Spotify URIs are not validated. The VibeProfile contains `seed_artists: ["spotify:artist:..."]` and exclusion artist_ids, both of which the LLM can hallucinate. §7 covers 'invalid/unparseable JSON' but not 'valid JSON with non-existent URIs.' Add a validation pass (resolve URIs via `GET /artists?ids=...` in a batch) and define behavior on partial validity.
- (minor) `Retry-After` parsing rule omits HTTP-date form. §4.4 says 'read `Retry-After` (seconds).' Per RFC 7231, `Retry-After` may be either delta-seconds or an HTTP-date. Spotify almost always returns seconds, but the wrapper should specify behavior for the date form (parse and compute delta, fall back to default if malformed) so it doesn't crash on an edge response.
- (minor) `user-library-read` scope is described as 'optional' (§4.1) without specifying when it's requested. Is it always in the consent screen, or only if `--include-liked` is passed at first auth? If toggled later, does the app re-trigger the auth flow to upgrade scopes? Specify the scope-upgrade path or commit to always-requested.
- (minor) Cache TTL for immutable data is unjustified. §4.3 sets a 30-day TTL on audio features and track metadata. Audio features for a given track URI are immutable; track metadata changes only on rare relink events. A 30-day eviction means large recurring re-fetch cost for no correctness gain. Either drop the TTL (cache forever, evict by LRU/size) or justify the 30-day window (e.g., to pick up market-relink changes).
- (minor) Run log retention/rotation is unspecified. §4.3 writes one JSON file per run to `~/.local/share/vibeify/runs/<timestamp>.json` with no rotation, size cap, or pruning command. After hundreds of runs this is awkward to navigate. Add a retention policy or a `vibeify runs prune` command.
- (minor) Token-storage path inconsistency. §4.1 fallback path is `~/.config/vibeify/token.json` (XDG_CONFIG_HOME), while §4.3 uses `~/.local/share/vibeify/` (XDG_DATA_HOME) for runs and cache. The split is defensible (config vs. data) but not justified, and on Windows/macOS the fallback paths are not specified at all (only the keychain backend is named). Specify per-OS fallback paths.
- (minor) `tempo_bpm` typing. §6 shows `tempo_bpm: [70, 100]` as integers, but Spotify's `tempo` is a float (e.g., 119.876). Specify rounding/coercion at the validation boundary so the scorer doesn't reject feature values silently.

## suggested-next-version

0.2

<!-- samospec:critique v1 -->
{
  "findings": [
    {
      "category": "contradiction",
      "text": "Spotify API deprecations not acknowledged. The pipeline (§4.2 step 3, step 4 'seed_artists/seed_genres', §6 VibeProfile) leans heavily on `/audio-features` and seed-based recommendation primitives. As of late 2024, Spotify removed access to `/audio-features`, `/audio-analysis`, `/recommendations`, related-artists, and 30-second previews for new/non-extended Web API client IDs. A veteran Spotify engineer would not write a v0.1 spec whose entire scoring core (`target_features`, weighted feature distance) depends on an endpoint new apps cannot call. The spec must either declare the app is grandfathered (with evidence) or define a fallback scoring path that does not require audio-features.",
      "severity": "major"
    },
    {
      "category": "contradiction",
      "text": "`VibeProfile.exclusions` shape is inconsistent. §4.2 step 4 lists `exclusions: [...]` (flat array) while §6 specifies `exclusions: { artist_ids: [], genres: [] }` (object). §4.5 also references LLM-derived 'exclusions' without disambiguating. Pick one shape and apply it everywhere, or the validator/scorer/LLM-prompt schema will disagree.",
      "severity": "major"
    },
    {
      "category": "contradiction",
      "text": "Determinism claim conflicts with LLM usage. §4.5 asserts output is 'Deterministic for a given (prompt, library snapshot, model) triple,' and §4.3 says the run log enables 'reproducibility,' but the spec never pins LLM temperature, top_p, seed, or system-prompt hash, nor does it cache the LLM response keyed by prompt_hash to short-circuit re-runs. With default sampling, the same triple yields different VibeProfiles. Either specify temperature=0 + a recorded seed (and note providers that don't honor seeds), or weaken the determinism claim.",
      "severity": "major"
    },
    {
      "category": "weak-testing",
      "text": "No testing strategy whatsoever. The spec defines an OAuth dance, a paginated fetch pipeline, a 429/5xx retry policy, an LLM JSON contract, a scoring/ranking algorithm, and idempotent batched writes — and says nothing about how any of it is verified. At minimum the spec should require: (a) a recorded-fixture/VCR layer for Spotify HTTP so retry, pagination, and 401-refresh paths are tested without live calls; (b) golden-file tests for VibeProfile JSON-schema validation including the malformed-LLM retry path in §7; (c) deterministic unit tests for the scorer and dedup/segment-ordering rules in §4.5 against a fixed candidate fixture; (d) a contract test that fails loudly when a Spotify endpoint the pipeline depends on returns 403/404 (catches the audio-features deprecation regression). Without this, none of the §4.4/§4.5/§7 guarantees are checkable.",
      "severity": "major"
    },
    {
      "category": "weak-testing",
      "text": "§4.4's retry/idempotency rules have no acceptance criteria. 'Retry the same request up to 5 times,' '0–500 ms jitter,' '10 req/s token bucket,' and 'POST/PUT/DELETE retried only on 429 and 5xx' are all asserted but the spec does not say how they are validated (e.g., a fake clock + fake transport that injects 429 with `Retry-After`, asserts sleep duration within jitter window, asserts max-attempts boundary, asserts 4xx-non-429 is not retried). Add a testability requirement: the HTTP wrapper must accept an injectable clock and transport so these policies are unit-testable.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "Playlist-create idempotency under 5xx retry is undefined. §4.4 permits retrying POST on 5xx; §7 says 'Playlist is created first (empty), tracks added in idempotent 100-track batches; on failure, retry from last successful batch.' But `POST /users/{user_id}/playlists` has no idempotency key in the Spotify API — a 5xx response after the server created the playlist, retried, will produce two empty playlists in the user's account on every transient failure. Specify either (a) do not retry the create call on 5xx, only the track-add calls, or (b) on retry, first list recent playlists and reuse a name+description match created within the last N seconds.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "Redirect URI scheme is incompatible with Spotify's exact-match requirement. §4.1 specifies `http://127.0.0.1:<ephemeral-port>/callback`. Spotify requires every redirect URI to be registered verbatim in the app dashboard, including port. The spec must either (a) pin a fixed port (and document fallback behavior if it's busy), or (b) register a small set of known ports and pick from them, or (c) register a single port and document the contention failure mode in §7. As written, the auth flow will fail for any port not pre-registered.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "Scoring function is underspecified. §4.2 step 5 says 'weighted distance across target audio features + artist/genre match + mood-keyword match on track/album metadata.' §6 expresses `target_features` as `[min, max]` ranges, but 'distance' is undefined for ranges (distance to midpoint? zero inside the range? penalty proportional to overshoot?). 'mood-keyword match on track/album metadata' is also ambiguous — track name, album name, artist genres, or all three, and with what matching (substring, token, embedding)? §9 defers 'exact scoring weights' but the *form* of the function is also missing and is not just a tuning knob.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "Secondary dedup tiebreak proxy is misaligned with its name. §4.5 says 'prefer the track already present in the user's most-played source playlists (proxy: track appears in the most playlists).' 'Most-played' and 'appears in the most playlists' are different signals — a track in five rarely-opened playlists is not 'most-played.' Either rename the criterion to 'most-saved-across-playlists' or specify a different proxy (e.g., recently-added, owned-vs-followed weighting).",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "Segmented ordering (opener/core/closer) is named but not defined. §4.5 says the final list is sorted 'within 3 segments' by 'LLM-derived target-feature distance' but never says how segment boundaries are computed (equal thirds by count? by energy/valence trajectory? does the LLM emit a per-segment target?), nor which feature defines the arc. Without this, the ordering is neither deterministic nor reviewable.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "`is_playable` requires a market parameter that is not specified. §4.5 drops tracks with `is_playable: false` 'in user's market,' but the spec never says which Spotify endpoints are called with `market=from_token` (or the user's country code from `GET /me`). Without this, `is_playable` is absent from many responses and the filter silently no-ops or drops everything.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "LLM context-window strategy for large libraries is missing. §4.2 step 4 sends a 'compact summary of the candidate pool' to the LLM, but a user with thousands of unique tracks/artists has no defined truncation, sampling, or summarization rule. Specify the maximum payload (token budget), the summarization shape (e.g., top-K artists by count, genre histogram bucketed to N), and what is dropped when over budget.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "LLM-emitted Spotify URIs are not validated. The VibeProfile contains `seed_artists: [\"spotify:artist:...\"]` and exclusion artist_ids, both of which the LLM can hallucinate. §7 covers 'invalid/unparseable JSON' but not 'valid JSON with non-existent URIs.' Add a validation pass (resolve URIs via `GET /artists?ids=...` in a batch) and define behavior on partial validity.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "`Retry-After` parsing rule omits HTTP-date form. §4.4 says 'read `Retry-After` (seconds).' Per RFC 7231, `Retry-After` may be either delta-seconds or an HTTP-date. Spotify almost always returns seconds, but the wrapper should specify behavior for the date form (parse and compute delta, fall back to default if malformed) so it doesn't crash on an edge response.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "`user-library-read` scope is described as 'optional' (§4.1) without specifying when it's requested. Is it always in the consent screen, or only if `--include-liked` is passed at first auth? If toggled later, does the app re-trigger the auth flow to upgrade scopes? Specify the scope-upgrade path or commit to always-requested.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "Cache TTL for immutable data is unjustified. §4.3 sets a 30-day TTL on audio features and track metadata. Audio features for a given track URI are immutable; track metadata changes only on rare relink events. A 30-day eviction means large recurring re-fetch cost for no correctness gain. Either drop the TTL (cache forever, evict by LRU/size) or justify the 30-day window (e.g., to pick up market-relink changes).",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "Run log retention/rotation is unspecified. §4.3 writes one JSON file per run to `~/.local/share/vibeify/runs/<timestamp>.json` with no rotation, size cap, or pruning command. After hundreds of runs this is awkward to navigate. Add a retention policy or a `vibeify runs prune` command.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "Token-storage path inconsistency. §4.1 fallback path is `~/.config/vibeify/token.json` (XDG_CONFIG_HOME), while §4.3 uses `~/.local/share/vibeify/` (XDG_DATA_HOME) for runs and cache. The split is defensible (config vs. data) but not justified, and on Windows/macOS the fallback paths are not specified at all (only the keychain backend is named). Specify per-OS fallback paths.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "`tempo_bpm` typing. §6 shows `tempo_bpm: [70, 100]` as integers, but Spotify's `tempo` is a float (e.g., 119.876). Specify rounding/coercion at the validation boundary so the scorer doesn't reject feature values silently.",
      "severity": "minor"
    },
    {
      "category": "weak-testing",
      "text": "No reproducibility test for the run-log claim. §4.3 says the log enables 'A/B comparison across prompts' and §4.5 claims determinism. There is no proposed harness for replaying a stored run-log against a frozen library snapshot and asserting the same selected URIs come out. Without a replay test, the reproducibility claim is undemonstrated.",
      "severity": "minor"
    }
  ],
  "summary": "The spec is well-organized and the auth/rate-limit/dedup decisions are individually sensible, but it has several blocking issues a veteran Spotify engineer should have caught: the entire scoring core depends on audio-features and seed-recommendations endpoints that Spotify has restricted for new clients, with no fallback. Determinism is asserted without pinning LLM sampling, the redirect URI scheme is incompatible with Spotify's exact-match registration, playlist-create retry on 5xx will produce duplicate empty playlists, and the exclusions schema is contradictory between §4.2 and §6. The scoring function's *form* (not just weights) and the segment-ordering algorithm are underspecified enough that two implementers would build different products. Most critically, the spec contains zero testing strategy: no fixture/VCR layer for Spotify, no harness for the retry policy, no golden tests for VibeProfile validation, no replay test for the reproducibility claim. Resolve the API-deprecation question first (it may invalidate §4.2 and §6 entirely), then tighten determinism, idempotency, and testability before v0.2.",
  "suggested_next_version": "0.2",
  "usage": null,
  "effort_used": "max"
}
<!-- samospec:critique end -->
