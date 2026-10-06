# Privacy and Source

## What this repository is

A portfolio showcase for a production application named Fluentta AI. It explains software engineering choices; it is not a source distribution, not legal advice, and not the app's privacy policy. The app is **not currently publicly released** on app stores.

Language-learning features do not certify proficiency. Crisis messaging in AI chat is a fixed-response safety net, not counselling or emergency services.

## What is published here

- Written descriptions of architecture, AI boundaries, persistence and trade-offs
- Original SVG diagrams in `assets/diagrams/`
- Real screenshots from the Android build in `assets/screenshots/`, cropped to app content (status bar and ad-placeholder strips removed), captured with sample or placeholder data, metadata removed
- The app icon resized to WebP in `assets/branding/`

## What is intentionally not published

- Production Flutter and functions source code
- Firebase security rules, indexes and project configuration
- API keys, function secrets, keystores and signing material
- Store product identifiers, ad unit identifiers and mediation account details
- Internal endpoint URLs, function names as deployment identifiers, and collection or field names in Firestore
- System prompts, model identifiers and provider account details for the third-party language-model API
- Exact heart quotas, ad frequency constants and XP tuning tables
- User data of any kind

Line, file and test counts in the README are static counts gathered from the production tree (Dart files excluding generated localisation, Node functions excluding tests). No download counts, ratings, revenue, model accuracy or latency benchmarks are claimed.

Diagram labels are conceptual and do not map one-to-one to production identifiers.

## Notes on figures

Screenshots were re-encoded to WebP without EXIF or XMP metadata. Diagrams were authored for this repository and validated as well-formed SVG.

## Data handling (engineering summary)

The authoritative privacy policy for a released build will live inside the app and its store listing. At engineering level, the following categories apply:

### Stored primarily on device

- Lesson and roleplay progress structures, saved words, XP, streak dates, daily goal minutes, hearts balance, theme and reminder preferences, tutorial flags, ad counters, and premium boolean cached from store purchases.
- Open chat message text for the current session only (cleared when the screen is disposed unless the OS keeps memory snapshots outside app control).

### Stored in Firestore when signed in

- Profile and setup fields (name, email, language choice, English goal and level, onboarding flags).
- Aggregated learning stats, corrections count, heart count mirrors, and daily goal snapshots synced from local services.

Photos are not part of the learning model. Chat logs are not written to Firestore as a dedicated subcollection in the described client.

### Sent to HTTPS Cloud Functions (with Firebase ID token)

| Feature | Categories leaving the device |
|---|---|
| **Open chat** | User message text, recent turn text (user and assistant roles), optional CEFR level string, goal id, app language code |
| **Chat transcription** | Audio recording bytes (base64), mime type, filename |
| **Pronunciation** | Audio bytes, expected English phrase, optional feedback language code |
| **Vocabulary translate** | English meaning and example strings, target language code |

Functions forward necessary portions to a third-party language-model and speech API using server-side credentials. The mobile app never embeds those credentials.

### Sent to Firebase Analytics (typed events)

Event names and parameters include feature funnels (lesson complete, chat submit, pronunciation record, heart gate), conversation id and sequence for correlating turns within a session, heart balances before and after actions, error classes, and duration buckets. This portfolio does not enumerate every parameter key.

Firebase Messaging may deliver push payloads configured in the console; handling code exists in the app project.

### Not claimed

- End-to-end encryption of learner content beyond platform TLS in transit and platform storage defaults at rest.
- GDPR or HIPAA compliance labels.
- Anonymity of AI requests: calls are authenticated to a Firebase user id.
- Zero retention by third-party providers (subject to provider and operator policies).

Local data is not additionally encrypted by the app beyond operating-system defaults, and no in-app PIN lock is described.

## Third-party services (identification only)

Platform SDKs linked from the app include Firebase (Auth, Firestore, Analytics, Messaging, Remote Config), Google Sign-In, Apple Sign-In, Google Mobile Ads, and store in-app purchase APIs. Speech output uses the device TTS engine via a Flutter plugin. Speech input uses a device recognizer plugin as fallback while primary chat transcription uses the cloud proxy.

## Access

Source access for hiring or diligence can be arranged at the owner's discretion. This public repository remains descriptive only.
