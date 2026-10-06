# Technical Case Study

This case study walks through three representative engineering paths in Fluentta AI: an open chat turn, a pronunciation assessment, and progress sync across local storage and Firestore. Names here are descriptive; they are not production identifiers.

## Case 1 — Open chat turn (client and proxy)

### Starting state

The learner opens chat practice from the home or speak tab. The view model inserts a static greeting bubble (localised string, not from the model), reads connectivity, loads optional TTS auto-speak preference from SharedPreferences, and mints a one-off conversation id for analytics.

Setup metadata pulled from onboarding includes a CEFR level code (mapped from the learner's self-reported band), an English-learning goal id, and the app language code used for correction explanations.

### User sends text

1. **Pre-checks:** Network must be available. If the user is not Pro and has no hearts, the UI surfaces an out-of-hearts state without calling the network.
2. **Analytics (submit):** Log message sequence, input method (`text` vs `voice_transcribed`), topic starter type, heart balance before, and estimated cost.
3. **History assembly:** Walk in-memory messages (excluding the pending user line). Map user bubbles to user role and tutor bubbles to assistant role. Empty lines are skipped. The proxy later truncates to its own maximum turn window.
4. **HTTP request:** POST JSON with user text, history array, and optional level, goal and native language fields. Authorization header carries a fresh Firebase ID token.
5. **Proxy path:** Verify token; reject unauthenticated calls. Run crisis-pattern guard; if triggered, return a fixed JSON payload and skip the model. Otherwise build chat messages server-side (system instructions and parsing live in the functions package), call the third-party chat completion API with JSON response mode, parse tutor reply, correction fields and correctness flag.
6. **Client response handling:** On success, charge one heart unless Pro. Append tutor message with optional correction metadata. Increment corrections counter when a correction is shown. Optionally speak the reply via TTS with language segmentation (English practice lines slower; explanation may use Urdu, Spanish or French voice when present in text).
7. **Analytics (reply):** Log latency bucket, success, heart balance after, and whether a correction was shown.

### User sends voice

Voice mode reuses the same send pipeline after transcription:

1. Record to a temp file (WAV preferred).
2. POST audio as base64 to the transcription function (separate from pronunciation assessment). The conversational transcription model is tuned for accented English; it may lightly clean disfluencies.
3. Place returned transcript in the composer; call the same `_sendUserText` path with input method marked as voice-derived.
4. If recording fails, fall back to on-device recognition listening in parallel during capture for a best-effort transcript.

### Session lifetime

Messages are not written to Firestore as chat history. Closing the route disposes the view model and clears the thread. Daily goal minutes accrue from session open time tracked on dispose.

### Design takeaway

The client owns UX state (modes, recording gestures, heart ledger, TTS). The proxy owns secrets, authentication, crisis short-circuit, and provider selection. Splitting responsibilities keeps the mobile binary free of API keys while still allowing rich, multi-turn prompts to remain server-side (not published in this portfolio).

## Case 2 — Pronunciation assessment (audio upload and deterministic scoring)

### Capture

The pronunciation flow primes microphone permission early. For each phrase:

1. TTS plays the target sentence on demand (analytics event for listen).
2. Recording uses the `record` plugin with mono 16 kHz WAV into a cache directory. Amplitude streams drive a simple waveform UI. A maximum duration timer auto-stops capture.
3. On stop, bytes are retained in memory; optional parallel on-device STT may have collected a rough transcript for debugging or fallback display.

### Heart economics

Before assessment, the view model attempts to deduct one heart (unless Pro). If the subsequent network call fails, logic can refund the heart and surface an error string localised in the UI.

### Cloud assessment

The client POSTs base64 audio, mime type, filename, expected phrase (English target sentence), and optional feedback language code.

Server steps (conceptual):

1. Authenticate as in chat.
2. Normalise audio levels when needed (utility module in functions).
3. Call speech-to-text **without** passing the expected phrase as a prompt, so the model cannot simply echo the target.
4. Feed transcript and phrase into deterministic scoring code: tokenise expected and spoken words, dynamic alignment, penalties for insertions, optional per-token confidence from the recogniser when available, tricky-cluster hints for coaching text.
5. Return JSON with overall score, per-word status, whether speech was detected, coaching tip string, and a marker indicating scoring came from transcript alignment rather than a free-form model judgment.

The client maps results into UI bands (success, partial, no speech) and logs analytics with phrase index and attempt count.

### On-device TTS vs cloud STT/TTS

Phrase playback is entirely on-device. Assessment is entirely cloud-side for consistency. This differs from chat, where both cloud transcription and on-device STT participate.

## Case 3 — Progress sync (local first, Firestore merge)

### Local source of truth for UI

SharedPreferences holds dense keys: per-lesson progress structures, XP total, lives, streak dates, daily goal minutes, premium flag, roleplay bonus flags, theme, reminder schedule, chat TTS preference, ad counters, and more. View models read through a singleton storage accessor initialised at app start.

### When Firestore enters

On sign-in, the user repository merges remote profile fields (language, goal, level, setup completion) with local values, then pushes learning stats and heart count. A progress sync service batches writes:

- Lesson completion updates local state immediately, then schedules Firestore updates for aggregates (XP, words learned, lesson counts, corrections).
- Heart changes coalesce through a serialised tail queue to avoid write storms during chat.
- Daily goal minutes flush on meaningful events.

Sign-out clears in-memory auth state but leaves local lesson progress on disk until explicitly reset; remote remains authoritative for cross-device restore when the user returns.

### Offline behaviour

Bundled lesson JSON requires no network. AI endpoints short-circuit with offline errors. Firestore reads on login need connectivity; failures surface localised error strings in profile flows.

### Design takeaway

The app optimises for instant lesson interactions locally, using the cloud as backup and social login anchor rather than as an online-only database. AI usage is intentionally online-only to control cost and keep provider calls authenticated.

## Testing and verification

Automated tests in the repository are minimal: a handful of Dart widget or unit tests and Node tests around pronunciation scoring and audio utilities in the functions package. Most behaviour described here is validated manually and through analytics funnels rather than CI coverage. This portfolio does not claim that tests pass in any particular environment.

## Related reading

- [ARCHITECTURE.md](ARCHITECTURE.md) for layer diagrams and data categories.
- [ENGINEERING_CHALLENGES.md](ENGINEERING_CHALLENGES.md) for entitlement, abuse and privacy trade-offs.
- [PRIVACY_AND_SOURCE.md](PRIVACY_AND_SOURCE.md) for what leaves the device on each AI path.
