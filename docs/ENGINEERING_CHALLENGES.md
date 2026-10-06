# Engineering Challenges

This document records deliberate trade-offs and open edges in Fluentta AI as implemented. It is not a security audit and does not describe exploitation steps.

## 1. Client-side entitlements vs server-enforced quotas

**Situation:** Pro status, heart balances and roleplay locks are evaluated in Dart before opening AI features or premium content. Store purchase verification updates local flags and Firestore, but Cloud Functions currently authenticate users without re-checking hearts or subscription tier on each AI call.

**Why it was done:** Shipping speed and simplicity — the proxy already centralises secrets and crisis handling; pushing entitlement logic server-side requires durable server state, receipt validation hooks, and conflict rules with the heart ledger on device.

**Trade-off:** A modified client could theoretically spam AI endpoints while authenticated. Mitigations today are Firebase auth (per-user identity), provider billing on the backend account, and the absence of published API keys in the app — not cryptographic proof of payment on each request.

**Natural next step:** Bind AI calls to server-side counters or Pro claims verified from store receipts, similar to patterns used in other portfolio apps with a single proxy gate.

## 2. Heart ledger split across device and cloud

Hearts decrement locally after successful AI responses, then async writes update Firestore for signed-in users. Race conditions between devices could diverge counts briefly. The UI treats local storage as authoritative for gating mid-session.

Refill rules (daily reset calendar, rewarded ads, consumable packs) are implemented entirely client-side with dated keys in SharedPreferences.

## 3. Chat history privacy vs model context

Open chat keeps messages in RAM for one screen session. Each request still transmits a slice of recent turns to the proxy so the model can stay coherent. That slice is ephemeral on the server in the sense of no dedicated chat collection, but provider logs may retain prompts under their policies.

Learners are not shown a separate "data sent to AI" screen beyond the app privacy policy link; this portfolio summarises categories in [PRIVACY_AND_SOURCE.md](PRIVACY_AND_SOURCE.md).

## 4. Pronunciation scoring limits

Scoring aligns transcript tokens to expected words. It is not phoneme-level acoustic analysis. Accents, background noise, or homophones can produce mismatches despite intelligible speech. The product copy positions feedback as practice guidance, not authoritative pronunciation certification.

Blind transcription reduces bias from hinting the target phrase, but cannot eliminate all ASR errors; silence hallucinations are filtered with a denylist on the server.

## 5. Crisis detection scope

Deterministic regex-style guards catch explicit English crisis phrases before model calls. They cannot cover every language, indirect wording, or typos. Model instructions add a second layer but remain probabilistic. The feature is a consumer-app safety net, not a clinical intervention.

## 6. Offline and low-connectivity UX

Lessons and roleplay remain usable offline. AI features hard-require network and fail with localised offline errors. Partial uploads for long recordings could fail near size limits; the client validates empty audio before upload.

Cloud Functions cold starts can add latency to the first chat message after idle periods; production can configure minimum instances at a monetary cost (commented in server source).

## 7. Ads, consent and ATT

The app integrates mediated ads with Remote Config cadence (for example interstitials after every few completed lessons on free tier). iOS App Tracking Transparency is requested through a consent service before analytics/ad initialisation paths run. Exact ad unit identifiers and mediation waterfalls are omitted here.

Premium subscribers skip ad surfaces tied to entitlements checks in view models.

## 8. Localisation and content

UI strings exist for four languages, while lesson JSON remains English. Vocabulary translation calls add latency the first time a non-English UI learner expands a word card. Failures fall back to English strings.

Mixed-language tutor replies (English practice plus Urdu explanation) require TTS segmentation logic to pick voices per script run.

## 9. Test coverage gap

Production includes:

- About 95 lines across two Dart test files in the app project.
- Node unit tests for pronunciation scoring and audio normalisation helpers.

There is no broad golden-test suite for lesson parsing, sync merge logic, or chat view models documented here. Regression risk is mitigated through manual QA and staged releases, not automated CI gates described in this repo.

## 10. Unreleased product constraints

Because Fluentta AI is not publicly released, store-specific behaviours (subscription grace periods, regional pricing, production Remote Config values) may still change. This documentation reflects the codebase structure rather than a committed go-to-market snapshot.

## Summary table

| Area | Current approach | Follow-up if scaling |
|---|---|---|
| AI auth | Firebase ID token on each call | Rate limits per uid |
| AI billing gate | Client hearts | Server quota + Pro flag |
| Chat retention | Session memory only | Optional export if product requires |
| Pronunciation | Transcript alignment | Dedicated ASR vendor features if needed |
| Curriculum | Bundled JSON | CMS only if content velocity demands |
