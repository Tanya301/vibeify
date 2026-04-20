# Reviewer B — Claude

## summary

Spec is materially stronger than v0.1 — the scoring/segmentation algebra is now precise enough to reproduce across implementations in features_mode=true, and the retry/idempotency policy is thoughtful. Remaining issues cluster in two places. (1) The 5xx recovery story for Spotify POSTs is built on two claims that don't match the Web API: snapshot_id does not encode creation time (breaks the playlist-create reconciliation), and POST /playlists/{id}/tracks is not idempotent without a position parameter (breaks the track-add retry carve-out). (2) The features_mode=false path — which is the expected common case on post-2024 client IDs — is underspecified in exactly the places that drive output identity: the segment-ordering fallback, the normalized-title dedup, the capability-probe target, and the 50%-unresolved threshold. Testing is otherwise thorough, but §9 test 6 is weaker than advertised relative to the determinism claim it guards (it bypasses the LLM via cache and therefore cannot detect seed-honoring regressions), and run-log retention behavior is untested.

## contradiction

- (major) §4.4 specifies that on a 5xx during playlist creation, reconciliation should find a playlist 'whose snapshot_id implies it was created within the last 60 s'. But Spotify's GET /me/playlists response contains neither a creation timestamp nor a time-interpretable snapshot_id — snapshot_id is an opaque version token, not a timestamp. The specified reconciliation mechanism cannot work as written; two implementers following the spec would either invent incompatible heuristics or silently skip reconciliation. A spec-level alternative (e.g., compare name+description match + absence-before-POST check) is required.
- (major) §4.4 claims 'Playlist track-add POSTs (POST /playlists/{id}/tracks) are retried on 5xx because batch adds are idempotent when the client replays the same 100-URI window from the last successful offset.' Spotify's POST /playlists/{id}/tracks without a 'position' parameter always appends, so a 5xx that actually reached the server followed by a retry duplicates tracks. The spec neither mandates using the position parameter nor describes how the client disambiguates 'server received and applied' from 'server never received' on a 5xx. The idempotency claim contradicts the endpoint's documented append behavior and directly undermines §9 test 1's assertion that 5xx-on-track-add is safely retried.

## ambiguity

- (major) §4.5.2 features_mode=false segmented-ordering fallback is specified as 'opener = most-distinctive artists for the vibe; core = mixed; closer = longest tracks'. None of 'most-distinctive', 'mixed', or 'longest tracks' is defined as an algorithm, and the 'artist-rotation key' referenced in 'ascending by artist-rotation key' is never defined anywhere in the spec. Two implementers would produce completely different orderings in the metadata-only mode, violating the §4.5 determinism guarantee in what is explicitly the common case (post-2024 clients without /audio-features access).
- (major) §4.5 secondary dedup pass keys on '(normalized_title, primary_artist_id)' but 'normalized_title' is never defined. The whole point of the fallback is catching re-releases/remasters (e.g., 'Dreams', 'Dreams - 2004 Remaster', 'Dreams (Live at Wembley)'). Without specifying case handling, punctuation stripping, parenthetical suffix removal, whitespace collapsing, or unicode normalization, two implementations will dedup different sets of tracks. §9 test 5 asserts '(normalized_title, primary_artist_id) fallback' works but cannot be written precisely against an undefined normalization function.
- (major) §4.2 step 3 describes the capability probe as 'one GET /audio-features?ids=<single-known-track> call' without specifying which track. Is it a hardcoded, well-known URI baked into the binary? The first URI from the user's candidate pool (which may itself be region-locked and return 404 for reasons unrelated to capability)? A fixture URI? This directly affects correctness of the probe (distinguishing capability-403 from market-404) and the reproducibility of §9 test 7, which has no stable probe target to script.
- (major) §4.2 step 6 says 'drop unresolved URIs with a warning, do not abort unless >50% are unresolved'. Underspecified on three axes: (1) does the threshold apply only to seed_artists, or also to exclusions.artist_ids (also artist URIs emitted by the LLM)? (2) at exactly 50% behavior is undefined — strict '>' suggests proceed, but many readers read '>50%' colloquially as ≥. (3) If seed_artists has 1 URI and it fails to resolve, one failure is 100% and the run aborts even though this is a single-failure edge case. The rule needs a concrete denominator, a floor (abort only if unresolved ≥ k AND ratio > 50%), and explicit scope.
- (major) §4.5.1 defines mood_score as 'matches / len(mood_keywords), clamped to [0, 1]' but does not define behavior when mood_keywords is empty; the VibeProfile schema in §6 does not mark it required and an empty array is structurally valid, yielding division by zero. Additionally, the tokenization target 'joined-artist-genres' does not specify a join separator, which matters because genres are themselves hyphenated (e.g., 'indie-folk') and the tokenization rule is 'simple whitespace+punctuation split' — implementations that join with ' ' vs '-' vs ',' produce different token sets and therefore different mood_score values.
- (minor) §4.5.2 segment energy targets read 'opener = max(0, m−0.15), core = m, closer = max(0, m−0.25)'. This gives closer < opener < core, which is an atypical arc (usual openers ramp up and closers taper; here every segment is at or below core). The spec does not justify the asymmetric 0.15/0.25 offsets, so implementers and test authors cannot tell whether these are intentional values or transcription errors. Because §9 test 4 asserts 'per-segment targets', the numbers are test-visible and should be explained or corrected.
- (minor) §4.3 specifies two retention policies for cache.db without resolving their interaction: audio features + core metadata are LRU-evicted when the DB exceeds 200 MB; playlist membership and is_playable use a 24h TTL. When the DB exceeds the cap, does eviction preferentially drop expired TTL entries first, or is it pure LRU across all entry types? Symmetrically for the 500 MB LLM cache: does a cache hit bump recency (read-time LRU) or is recency write-time only? These choices affect steady-state behavior and cannot be derived from the spec.
- (minor) §4.4 sets a client-side token bucket 'ceiling of 10 requests/second' but never specifies burst capacity (bucket size) or refill cadence. §9 test 2 asserts 'token bucket rate-limits to 10 req/s' — this assertion is not well-defined without a burst spec: a 10-RPS bucket with capacity 10 permits a 10-request burst at t=0, whereas capacity 1 permits only one request per 100 ms. The test will pass or fail depending on which interpretation the implementer picks, and paginated-fetch timing will differ materially in practice.
- (minor) §4.5's determinism clause invokes a 'library snapshot' as part of the deterministic input tuple, but the term is never defined. Does it include: the set of followed/owned playlist IDs at t0, each playlist's snapshot_id, the market used, the capability-probe result, cached vs live /artists lookups? Without a concrete definition, §9 test 6 cannot establish the preconditions it must freeze to reproduce identical output, and the determinism guarantee becomes unfalsifiable.
- (minor) §4.1 lists OS-specific fallback token files 'where [the keychain is] available' but never specifies the fallback trigger. Is the file used when the keychain API errors, when libsecret is missing, when the user is in a non-interactive session (SSH, cron) where keychain unlock cannot prompt, or only when the user opts in with a flag? The security posture of 0600 files on disk is materially different from a keychain-only policy; leaving the trigger undefined means two implementations can make opposite security choices while both claiming spec compliance.
- (minor) §4.2 step 4 mid-run degradation says 're-score already-fetched tracks under the new weights', but §4.2's own pipeline performs scoring (step 7) strictly AFTER feature fetch (step 4). If degradation occurs mid-fetch, no tracks have been scored yet, so 're-score' is ill-defined. It also leaves unclear whether partially-fetched audio features are (a) discarded wholesale and the run proceeds on metadata only, or (b) retained and used for the tracks that happen to have them while others are scored metadata-only — the two choices produce different rankings for the same run and the same capability event.

## weak-testing

- (major) §9 test 6 (run-log replay) claims to 'guard the §4.5 determinism claim', but that claim is explicitly 'conditional on seed honoring'. As written, the test replays 'a cached LLM response' — i.e., it bypasses the LLM entirely — so it validates deterministic scoring/ordering given an identical VibeProfile, not deterministic end-to-end output. The spec describes no test that detects seed-honoring regressions in the LLM adapter, nor any test for the weakened best-effort determinism on non-seed-honoring providers. The determinism promise is therefore materially under-tested relative to how it is advertised in §4.5 and §6 (run-log field seed_honored).
- (minor) §9 does not specify any test for the run-log retention behavior defined in §4.3: the 'vibeify runs prune --keep N' command (default 200) and the startup auto-prune at >1000 run logs. Both are stateful, filesystem-touching behaviors easy to regress silently (off-by-one on --keep, auto-prune firing at the wrong threshold, auto-prune deleting the in-flight run's log). Every other mutating code path is covered; this one is not.
- (minor) §9 test 7 (capability-regression) asserts 'the run completes with features_mode=false and a non-empty playlist' but does not specify properties of the candidate-library fixture. If the fixture contains only unplayable-in-market tracks (filtered by §4.5 exclusions), no tracks matching any mood keyword, or fewer than the default 30 tracks, the 'non-empty playlist' assertion is either vacuous (trivially satisfied by any track) or brittle (fails for reasons unrelated to capability handling). A concrete fixture spec with declared minimum sizes, playability, and genre coverage is needed.

## suggested-next-version

v0.3

<!-- samospec:critique v1 -->
{
  "findings": [
    {
      "category": "contradiction",
      "text": "§4.4 specifies that on a 5xx during playlist creation, reconciliation should find a playlist 'whose snapshot_id implies it was created within the last 60 s'. But Spotify's GET /me/playlists response contains neither a creation timestamp nor a time-interpretable snapshot_id — snapshot_id is an opaque version token, not a timestamp. The specified reconciliation mechanism cannot work as written; two implementers following the spec would either invent incompatible heuristics or silently skip reconciliation. A spec-level alternative (e.g., compare name+description match + absence-before-POST check) is required.",
      "severity": "major"
    },
    {
      "category": "contradiction",
      "text": "§4.4 claims 'Playlist track-add POSTs (POST /playlists/{id}/tracks) are retried on 5xx because batch adds are idempotent when the client replays the same 100-URI window from the last successful offset.' Spotify's POST /playlists/{id}/tracks without a 'position' parameter always appends, so a 5xx that actually reached the server followed by a retry duplicates tracks. The spec neither mandates using the position parameter nor describes how the client disambiguates 'server received and applied' from 'server never received' on a 5xx. The idempotency claim contradicts the endpoint's documented append behavior and directly undermines §9 test 1's assertion that 5xx-on-track-add is safely retried.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§4.5.2 features_mode=false segmented-ordering fallback is specified as 'opener = most-distinctive artists for the vibe; core = mixed; closer = longest tracks'. None of 'most-distinctive', 'mixed', or 'longest tracks' is defined as an algorithm, and the 'artist-rotation key' referenced in 'ascending by artist-rotation key' is never defined anywhere in the spec. Two implementers would produce completely different orderings in the metadata-only mode, violating the §4.5 determinism guarantee in what is explicitly the common case (post-2024 clients without /audio-features access).",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§4.5 secondary dedup pass keys on '(normalized_title, primary_artist_id)' but 'normalized_title' is never defined. The whole point of the fallback is catching re-releases/remasters (e.g., 'Dreams', 'Dreams - 2004 Remaster', 'Dreams (Live at Wembley)'). Without specifying case handling, punctuation stripping, parenthetical suffix removal, whitespace collapsing, or unicode normalization, two implementations will dedup different sets of tracks. §9 test 5 asserts '(normalized_title, primary_artist_id) fallback' works but cannot be written precisely against an undefined normalization function.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§4.2 step 3 describes the capability probe as 'one GET /audio-features?ids=<single-known-track> call' without specifying which track. Is it a hardcoded, well-known URI baked into the binary? The first URI from the user's candidate pool (which may itself be region-locked and return 404 for reasons unrelated to capability)? A fixture URI? This directly affects correctness of the probe (distinguishing capability-403 from market-404) and the reproducibility of §9 test 7, which has no stable probe target to script.",
      "severity": "major"
    },
    {
      "category": "weak-testing",
      "text": "§9 test 6 (run-log replay) claims to 'guard the §4.5 determinism claim', but that claim is explicitly 'conditional on seed honoring'. As written, the test replays 'a cached LLM response' — i.e., it bypasses the LLM entirely — so it validates deterministic scoring/ordering given an identical VibeProfile, not deterministic end-to-end output. The spec describes no test that detects seed-honoring regressions in the LLM adapter, nor any test for the weakened best-effort determinism on non-seed-honoring providers. The determinism promise is therefore materially under-tested relative to how it is advertised in §4.5 and §6 (run-log field seed_honored).",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§4.2 step 6 says 'drop unresolved URIs with a warning, do not abort unless >50% are unresolved'. Underspecified on three axes: (1) does the threshold apply only to seed_artists, or also to exclusions.artist_ids (also artist URIs emitted by the LLM)? (2) at exactly 50% behavior is undefined — strict '>' suggests proceed, but many readers read '>50%' colloquially as ≥. (3) If seed_artists has 1 URI and it fails to resolve, one failure is 100% and the run aborts even though this is a single-failure edge case. The rule needs a concrete denominator, a floor (abort only if unresolved ≥ k AND ratio > 50%), and explicit scope.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§4.5.1 defines mood_score as 'matches / len(mood_keywords), clamped to [0, 1]' but does not define behavior when mood_keywords is empty; the VibeProfile schema in §6 does not mark it required and an empty array is structurally valid, yielding division by zero. Additionally, the tokenization target 'joined-artist-genres' does not specify a join separator, which matters because genres are themselves hyphenated (e.g., 'indie-folk') and the tokenization rule is 'simple whitespace+punctuation split' — implementations that join with ' ' vs '-' vs ',' produce different token sets and therefore different mood_score values.",
      "severity": "major"
    },
    {
      "category": "ambiguity",
      "text": "§4.5.2 segment energy targets read 'opener = max(0, m−0.15), core = m, closer = max(0, m−0.25)'. This gives closer < opener < core, which is an atypical arc (usual openers ramp up and closers taper; here every segment is at or below core). The spec does not justify the asymmetric 0.15/0.25 offsets, so implementers and test authors cannot tell whether these are intentional values or transcription errors. Because §9 test 4 asserts 'per-segment targets', the numbers are test-visible and should be explained or corrected.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "§4.3 specifies two retention policies for cache.db without resolving their interaction: audio features + core metadata are LRU-evicted when the DB exceeds 200 MB; playlist membership and is_playable use a 24h TTL. When the DB exceeds the cap, does eviction preferentially drop expired TTL entries first, or is it pure LRU across all entry types? Symmetrically for the 500 MB LLM cache: does a cache hit bump recency (read-time LRU) or is recency write-time only? These choices affect steady-state behavior and cannot be derived from the spec.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "§4.4 sets a client-side token bucket 'ceiling of 10 requests/second' but never specifies burst capacity (bucket size) or refill cadence. §9 test 2 asserts 'token bucket rate-limits to 10 req/s' — this assertion is not well-defined without a burst spec: a 10-RPS bucket with capacity 10 permits a 10-request burst at t=0, whereas capacity 1 permits only one request per 100 ms. The test will pass or fail depending on which interpretation the implementer picks, and paginated-fetch timing will differ materially in practice.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "§4.5's determinism clause invokes a 'library snapshot' as part of the deterministic input tuple, but the term is never defined. Does it include: the set of followed/owned playlist IDs at t0, each playlist's snapshot_id, the market used, the capability-probe result, cached vs live /artists lookups? Without a concrete definition, §9 test 6 cannot establish the preconditions it must freeze to reproduce identical output, and the determinism guarantee becomes unfalsifiable.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "§4.1 lists OS-specific fallback token files 'where [the keychain is] available' but never specifies the fallback trigger. Is the file used when the keychain API errors, when libsecret is missing, when the user is in a non-interactive session (SSH, cron) where keychain unlock cannot prompt, or only when the user opts in with a flag? The security posture of 0600 files on disk is materially different from a keychain-only policy; leaving the trigger undefined means two implementations can make opposite security choices while both claiming spec compliance.",
      "severity": "minor"
    },
    {
      "category": "weak-testing",
      "text": "§9 does not specify any test for the run-log retention behavior defined in §4.3: the 'vibeify runs prune --keep N' command (default 200) and the startup auto-prune at >1000 run logs. Both are stateful, filesystem-touching behaviors easy to regress silently (off-by-one on --keep, auto-prune firing at the wrong threshold, auto-prune deleting the in-flight run's log). Every other mutating code path is covered; this one is not.",
      "severity": "minor"
    },
    {
      "category": "weak-testing",
      "text": "§9 test 7 (capability-regression) asserts 'the run completes with features_mode=false and a non-empty playlist' but does not specify properties of the candidate-library fixture. If the fixture contains only unplayable-in-market tracks (filtered by §4.5 exclusions), no tracks matching any mood keyword, or fewer than the default 30 tracks, the 'non-empty playlist' assertion is either vacuous (trivially satisfied by any track) or brittle (fails for reasons unrelated to capability handling). A concrete fixture spec with declared minimum sizes, playability, and genre coverage is needed.",
      "severity": "minor"
    },
    {
      "category": "ambiguity",
      "text": "§4.2 step 4 mid-run degradation says 're-score already-fetched tracks under the new weights', but §4.2's own pipeline performs scoring (step 7) strictly AFTER feature fetch (step 4). If degradation occurs mid-fetch, no tracks have been scored yet, so 're-score' is ill-defined. It also leaves unclear whether partially-fetched audio features are (a) discarded wholesale and the run proceeds on metadata only, or (b) retained and used for the tracks that happen to have them while others are scored metadata-only — the two choices produce different rankings for the same run and the same capability event.",
      "severity": "minor"
    }
  ],
  "summary": "Spec is materially stronger than v0.1 — the scoring/segmentation algebra is now precise enough to reproduce across implementations in features_mode=true, and the retry/idempotency policy is thoughtful. Remaining issues cluster in two places. (1) The 5xx recovery story for Spotify POSTs is built on two claims that don't match the Web API: snapshot_id does not encode creation time (breaks the playlist-create reconciliation), and POST /playlists/{id}/tracks is not idempotent without a position parameter (breaks the track-add retry carve-out). (2) The features_mode=false path — which is the expected common case on post-2024 client IDs — is underspecified in exactly the places that drive output identity: the segment-ordering fallback, the normalized-title dedup, the capability-probe target, and the 50%-unresolved threshold. Testing is otherwise thorough, but §9 test 6 is weaker than advertised relative to the determinism claim it guards (it bypasses the LLM via cache and therefore cannot detect seed-honoring regressions), and run-log retention behavior is untested.",
  "suggested_next_version": "v0.3",
  "usage": null,
  "effort_used": "max"
}
<!-- samospec:critique end -->
