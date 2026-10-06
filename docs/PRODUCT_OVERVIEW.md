# Product Overview

Fluentta AI is a mobile English-practice application aimed at non-native speakers. Most learning happens through structured, offline-capable content: CEFR-labelled vocabulary, grammar and reading lessons, plus six roleplay scenarios (job interviews, ordering food, airport, doctor visit, small talk, business meeting). Generative AI is reserved for open-ended conversation, pronunciation feedback on recorded speech, translating bundled English vocabulary glosses into the learner's chosen app language, and transcribing voice input in chat.

**Release status:** the app is **not currently publicly released**. This document describes the engineering of the production codebase, not a live store listing.

## Who it is for

Learners pick an app UI language (English, Spanish, French or Urdu), a self-reported English level, and a practice goal (work, travel, exam preparation, everyday conversation, or general fluency). Those choices steer lesson recommendations, roleplay unlock order, and the context sent to the AI tutor. The product does not certify proficiency or predict exam scores.

## Primary navigation

The shell uses four tabs:

| Tab | Purpose |
|---|---|
| **Home** | Daily goal, streak, hearts balance, shortcuts into AI chat and pronunciation |
| **Learn** | CEFR level selector (A1 through C2), module tiles for vocabulary, grammar, reading and saved words |
| **Role Play** | Scenario carousel with progress rings; each scenario opens dialogue, vocabulary and comprehension tracks |
| **Profile** | Account, subscription, reminders, language, theme, support links, optional weekly stats report for Pro users |

A first-run tutorial highlights learn paths, roleplay, hearts, AI chat and pronunciation.

## Bundled curriculum (no AI)

**Lessons:** A manifest lists 180 lessons spanning CEFR bands. Each lesson is JSON describing phases (for example vocabulary cards, grammar drills, reading passages, dialogue lines). Dialogue phases show scripted "You" and partner lines with speaker icons; learners can listen via on-device text-to-speech and are nudged to speak lines aloud locally without sending audio to the cloud.

**Roleplay:** Six scenarios ship as separate asset trees. Within a scenario, learners progress through staged dialogue lessons (often five per theme), vocabulary lists and comprehension checks. XP milestones and prior-level completion gates control which scenario stages unlock. Premium entitlement expands which scenarios are available without an XP gate; free users still receive a starter subset.

**English basics modules:** Additional bundled tracks target exam, work, travel and everyday themes for focused vocabulary.

**Saved words:** Learners bookmark terms from lessons into a local list for review.

All of the above functions without calling the language-model API. Internet is only needed for optional cloud sync and analytics, not for presenting lesson content.

## AI-powered features

All AI features require the learner to be signed in so the app can attach a Firebase ID token. Unsigned users can still consume bundled lessons where entitlements allow.

### Open chat practice

A dedicated chat screen hosts a multi-turn English conversation with an AI tutor persona. Input modes:

- **Text:** keyboard entry with send.
- **Voice:** hold-to-speak (with lock-to-record-hands-free), audio captured to a temporary file, uploaded for cloud transcription, then submitted as text. If capture fails, an on-device recognizer can supply a fallback transcript.

The tutor returns a conversational reply plus an optional single correction (original utterance, corrected form, short explanation in the app language when the model marks an error). Replies can auto-play through on-device TTS; learners can toggle that preference.

Quick starters (everyday, work, travel, grammar focus, open topic) seed the first user message topic for analytics. Each screen session generates a private conversation identifier used only to correlate analytics events, not stored as a durable chat log in the cloud.

**Hearts:** On the free tier, each successful tutor reply consumes one heart after the response arrives. Timeouts and errors do not charge. Pro subscribers bypass heart checks.

**Safety:** Before any model call, the proxy runs deterministic pattern matching on the user's message for explicit crisis wording and returns a fixed support response instead of model output when matched. Model-level safety instructions provide a second layer but are not treated as sufficient alone.

### Pronunciation practice

Learners work through phrase lists (for example five phrases per session). They can listen to the target phrase via TTS, record themselves (WAV at 16 kHz mono), and submit audio to the cloud. The server transcribes without seeing the expected phrase, aligns spoken words to the target sentence, scores each word, and returns an overall score band plus a short tip in the selected feedback language. The UI highlights weak words and supports retry.

Hearts follow the same charge-on-success pattern as chat. Failed assessments can refund the heart.

### Vocabulary translation

Bundled lesson content stores English meanings and examples. When the app UI language is not English, the client can request a one-off translation of meaning and example through the proxy (language code only, no persistent server cache described in the client).

## Progress, streaks and daily goal

XP accrues from lesson completion, roleplay modules and boosts. Streaks track consecutive days with qualifying activity. A daily minutes goal accumulates time while chat sessions are open and from other study actions. Counts sync to Firestore for signed-in users so reinstalls can restore stats; hearts and premium flags also merge on login.

The **weekly progress report** (Pro) is a simple on-device summary card list: streak, XP, lesson count, words learned, corrections count and daily goal progress. It is not an AI-generated narrative.

## Monetisation and entitlements

The free tier shows ads (with Remote Config-controlled cadence after lesson milestones), limits AI through the heart system, caps some roleplay scenarios, and restricts access to higher CEFR lesson paths. Pro subscription (verified through store APIs, entitlement stored locally and synced) removes ads, unlocks unlimited hearts, opens remaining roleplay and B2+ paths, and exposes the weekly report. Consumable purchases can add hearts; rewarded ads can refill hearts up to a daily cap configured in product logic.

Entitlement checks run on the client; the AI proxy today validates identity but does not re-check Pro status or heart balances server-side.

## What the app is not

- Not an official exam prep guarantee or score predictor.
- Not a human tutoring marketplace.
- Not a medical or mental-health service (crisis routing is a fixed-message safety net, not counselling).
- Not fully offline for AI, billing or sync.

For engineering detail on how these pieces connect, see [ARCHITECTURE.md](ARCHITECTURE.md) and [TECHNICAL_CASE_STUDY.md](TECHNICAL_CASE_STUDY.md).
