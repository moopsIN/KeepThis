# KeepThis — Project Overview

> **Don't just keep the thing. Keep the reason you kept it.**

**Status:** Pre-build / planning
**Platform:** Android (native Kotlin)
**Purpose:** Technology demonstration and portfolio project (not a business experiment)
**Audience for this document:** Engineers and AI agents planning or building KeepThis

---

## 0. How to use this document

This is a starting point, not a contract. Items are tagged so you know how much freedom you have:

| Tag | Meaning |
|---|---|
| **[Decided]** | Settled. Change only with a good reason and a note explaining why. |
| **[Default]** | A sensible starting choice. Override it if you find something better, and record the tradeoff. |
| **[Open]** | Not yet decided. Propose an answer, prototype it, or ask. |
| **[Later]** | Deliberately deferred. Design interfaces so it can be added, but don't build it now. |

Guidance for agents and engineers:

- When this document and your judgment disagree, surface the disagreement rather than silently picking one.
- Verify library versions, API availability, and platform limits against current documentation before committing. Android location, background, and on-device ML APIs change often.
- Preserve uncertainty. The product's core rule is that the system must never invent information it doesn't have.
- The mockups (Section 12) show look and feel and interaction ideas. They are not pixel specs.
- Where Section 14 (build order) says to prove something first, do that before polishing anything else.

---

## 1. Product summary

KeepThis is a lightweight, local-first Android app for capturing things you want to act on later.

Instead of organizing saved content into folders, bookmarks, or collections, KeepThis captures the **intention** behind a save. A user shares something to KeepThis and optionally adds one natural-language sentence:

- "Try this when I'm in Delhi."
- "Buy this next month."
- "Watch this when I have some time."
- "Send this to Mom when I get home."

KeepThis turns that unstructured thought into a structured, time- and context-aware item, and later **brings it back when it matters**: at the right place, at the right time, with a reminder that explains why.

### Positioning

- **Category:** an *intention inbox* — contextual read-later and "remind me when it matters."
- **Not:** a bookmark manager, a task manager, or a notes app.
- **Pitch:** *"Share it, say why in one sentence, forget it. KeepThis brings it back when it matters."*

| Existing tools | What they do | Gap KeepThis fills |
|---|---|---|
| Instagram saves, Pocket, Google Keep | Store content | They almost never resurface it, and they don't know why you saved it |
| Reminders, Todoist, Google location reminders | Remind you | Need manual form-filling and a task-management mindset |
| **KeepThis** | One sentence at capture, structure inferred, resurfaces by place and time | — |

### The defining experience

> **Capture now → forget about it → KeepThis brings it back when it matters.**

### Success criteria

1. A user can do **See something → Share to KeepThis → write one sentence → done** in under 10 seconds.
2. Weeks later, the right thing resurfaces at the right time or place.
3. The user thinks: *"I didn't tell it much, but somehow it understood what I meant."*

---

## 2. Product principles

1. **Capture is almost frictionless.** Primary entry is the Android share sheet. No folders, tags, or forms. When confidence is high, capture finishes on the first screen.
2. **Natural language over forms.** The user types a sentence. KeepThis interprets it and shows its understanding only when it needs confirmation.
3. **Context is part of the memory.** Every item can carry: *What* (the content), *Why* (intent), *When* (explicit or inferred time), *Where* (explicit or contextual place), *Who* (people), *Condition* (what must be true before it becomes relevant).
4. **Preserve uncertainty; ask instead of guessing.** Unknown stays unknown. A wrong confident guess is worse than a question. This is the best interaction idea in the product, so lean on it.
5. **Resist becoming an archive.** Items must resurface and be resolved (Do / Keep / Reschedule / Forget). The goal is not to accumulate saves; it's to help users act on things they once cared about.
6. **Sparse, meaningful notifications.** Contextual ("You wanted to try this place. You're in Delhi today."), never generic ("Reminder: Try this restaurant.").
7. **Local-first.** Content, notes, interpretations, and location context stay on the device by default. See Section 10 for what honestly leaves the device.

---

## 3. Decisions and scope

### Decided

- **[Decided]** Android only for V1.
- **[Decided]** Native Android: **Kotlin**. (This supersedes the earlier Flutter idea. There is heavy native integration — share intents, geofencing, notifications, ML — so a Flutter layer would add glue without delivering cross-platform value.)
- **[Decided]** **Tier 2 NLP only in V1** (ML Kit + small models + deterministic rules + device context).
- **[Decided]** **Tier 3 (Gemini Nano via AICore) is out of V1.** Leave a clean interface so it can be added later.
- **[Decided]** Network access is acceptable. Users are online when they share from Instagram, YouTube, or a browser. The product does not need to work offline-first at the NLP level, but item storage and the app's core flows should keep working without a connection.
- **[Decided]** No backend is required for the core experience. No accounts.

### V1 scope

**Capture**
- Android share sheet (`ACTION_SEND`) for URL and text
- Optional one-sentence user note
- Optional current location at capture time
- Metadata enrichment of the shared link (Section 6.4)
- *Secondary:* manual capture by typing or voice, with no shared content (Section 4.8). Built after the core loop is working.

**Interpret**
- Intent classification
- Date/time extraction
- Place extraction
- People extraction
- Condition parsing
- Confidence scoring and the "ask, don't guess" flow

**Context**
- Capture timestamp
- Optional current location (one-shot, event-based)
- User-defined Home (and optionally Work)

**Display**
- Home with a small number of sections
- Item detail
- Search
- Reschedule
- History (completed / dismissed)

**Action**
- Done, Reschedule, Edit, Dismiss

**Notifications**
- Time-based
- Location-aware resurfacing (geofence-driven)
- Notification actions

**Storage**
- Entirely local (Room/SQLite)

### Explicitly out of scope for V1

Social features, accounts, cloud sync, subscriptions, payments, collaborative lists, complex task management, **continuous background location tracking**, desktop app, browser extension, AI chat interface, folders/tag systems, calendar integration, on-device Gemini Nano.

### Later

- **[Later]** Gemini Nano (Tier 3) for ambiguous sentences such as "I definitely want to check this out once I'm back there."
- **[Later]** Calendar integration for conditions like "before my Goa trip."
- **[Later]** On-device embeddings for semantic search.
- **[Later]** Optional cloud features, added independently of the core.
- **[Later]** Home-screen widgets and a Quick Settings tile, including a voice-first quick capture shortcut.

---

## 4. Experience design

### 4.1 Capture

Entry: **Any app → Share → KeepThis**.

The capture screen is deliberately tiny:

```
[ preview card: image + title + source ]
[ text field: "Try this when I'm in Delhi next month." ]
KeepThis thinks:  [Visit] [Delhi] [Next month]      (Edit)
[ Keep this ]
```

Rules:

- The note is optional. An empty note is valid (see metadata inference, Section 6.4).
- Interpretation chips update live as the user types, and are editable.
- If confidence is **high**, tapping Keep saves immediately. No second screen.
- If confidence is **medium or low**, show the inline confirmation flow (Section 5) instead of a form.
- Capture must be dismissable and recoverable: a failed fetch or classification must never lose the user's save.

### 4.2 Interpretation display

The interpretation is shown as small typed chips: intent, place, time, person. Chips are the user's way of seeing and fixing what the system understood. Prefer plain-language questions over raw confidence numbers or bars (a percentage bar reads like a debug view).

### 4.3 Home

The primary question the home screen answers: **"What have I kept that matters now?"**

It must not look like a task manager. Keep the number of sections small.

| Section | Meaning | Example |
|---|---|---|
| **Now** | Conditions currently relevant | "You're in Delhi — you wanted to try this restaurant." |
| **Upcoming** | Becoming relevant soon by time | "Tomorrow: Watch this." |
| **Waiting** | Waiting for a future place/context | "When you're in Goa — 3 things you've kept." |
| **Someday** | Low-confidence or unspecified intent | "You'll see these when the time feels right." |

**[Open]** The mockups show Now / Upcoming / Someday and omit Waiting. Decide whether place-conditioned items live under Upcoming, or whether Waiting is added as a fourth grouping. This affects query logic, so decide before building the home queries. Default recommendation: keep three visible tabs and group place-waiting items inside Upcoming under "When you're in X" headers, or show Waiting only when it has items.

### 4.4 Item card and detail

A card communicates the **original content and the intention separately**:

```
[image]
Restaurant Reel                          (source: instagram.com)
"Try this when I'm in Delhi."            (the user's own words)
[VISIT] [DELHI] [NEXT MONTH]
Saved 18 days ago
```

When the condition becomes true ("You're in Delhi"), the card becomes actionable and moves to **Now**.

Detail screen adds: the user note (editable), free-form notes, related place on a map, and actions: **Done, Reschedule, Edit, More**.

### 4.5 Resurfacing and notifications

- Contextual copy that references the reason: "You wanted to try this place. You're in Delhi."
- Notification actions: **View**, **Done**, and **Later** (snooze). The mockup shows View/Done; add Later.
- Sparse by design: cap notifications per day and per item, and back off if the user dismisses repeatedly. **[Default]** Max 1 per item per relevance window; max 3 total per day.
- A resurfaced item asks, in effect, "Still interested?" with four outcomes: **Do it, Keep it, Reschedule it, Forget it.**

### 4.6 Focus mode

A full-screen, distraction-free view for an item that has become relevant ("You wanted to try this restaurant."), with the user's original sentence and an **Open link** button. This is a distinctive moment; keep it.

### 4.7 Search

Search across saved content, user notes, extracted entities, places, people, intents, dates, and status. Example queries: "restaurants", "things for Delhi", "things I wanted to buy", "stuff for Goa". V1 uses keyword/structured search over indexed fields (Room FTS). Semantic search with embeddings is **[Later]**.

### 4.8 Manual capture (secondary feature)

**Priority: secondary.** KeepThis is a share-first product. Manual reminders exist so the app is still useful on days when nothing is being shared, but they must not become the headline, the marketing story, or the default mental model. The share sheet remains the primary entry point and the main demo.

**[Default]** Manual capture is the same capture flow with no shared content, not a separate feature:

- The **Add** tab opens the capture screen without the preview card: one text box, live chips, one **Keep this** button.
- The same interpreter, chips, confidence rules, and resurfacing apply. "Call Mom when I get home" becomes a `Call` item with a Home condition.
- `KeepItem.content` is optional. For manual items, the user's sentence is the primary content.
- **Voice input** is a mic button on the same text field, using Android's `SpeechRecognizer` (on-device recognition where available). The transcript lands in the editable text field and the chips update live. **Never auto-save from speech**; transcription errors on names and places are common, and the user must confirm with Keep.
- With no link or metadata, intent leans toward `Remember`, `Do`, `Call`, or `Send`. Seed phrases for these intents need extra care.

Guardrails:

- **No task-manager drift.** No due-date pickers, priority fields, subtasks, or categories at creation. One sentence only; the structured edit view is available on demand.
- Manual items appear in the same Now / Upcoming / Someday sections, with a simple text card (no image) and the same chip treatment.
- Build order: after the core share → interpret → resurface loop works (Section 14). Voice is a stretch within this feature.
- Quick Settings tile and widget shortcuts are **[Later]**.

---

## 5. Confidence and the "ask, don't guess" model

This is the signature interaction. Every interpretation field (intent, place, time, person) carries a confidence score and a **source tag** (see 6.4).

| Level | Default threshold **[Default, tune with eval data]** | Behavior | Example |
|---|---|---|---|
| **High** | ≥ 0.80 | Save silently. Show chips only. | `Visit · Delhi · Next month` |
| **Medium** | 0.50 – 0.79 | Inline yes/no confirmation. | "Did you mean Delhi?" **[Yes] [Change]** |
| **Low** | < 0.50 | Ask an open question or put into Someday. | "What did you mean by "there"?" |

Rules:

- **Prefer asking to a confident wrong guess.**
- Ask **at most one** question at capture time. Defer other uncertainties to a later gentle prompt or leave them unknown.
- Unknown is valid. Do not force every item to have a date, location, or intent.
- Per-field confidence matters more than one overall number. An item can have a high-confidence intent and an unresolved place.
- Explain where a value came from when helpful: *"Delhi, from the post's caption."*

---

## 6. Interpretation engine (Tier 2)

### 6.1 Architecture principle

> **ML layers produce predictions and entities. Deterministic application code constructs the final KeepItem.**

The interpreter and resolver are **pure Kotlin modules with no Android dependencies**, so they are unit-testable on the JVM and can be swapped or extended later (including by a Gemini Nano implementation).

```
Raw input (note + shared payload + fetched metadata + device context)
   → Normalizer
   → Signal extractors (ML Kit, intent model, NER, condition grammar)
   → Resolver (merge signals, apply device context, score confidence)
   → InterpretationResult (per-field values, confidence, source)
   → KeepItem builder (deterministic)
```

Define a single interface so Tier 3 can plug in later without changing callers:

```kotlin
interface Interpreter {
    suspend fun interpret(input: InterpretationInput): InterpretationResult
}
// V1: Tier2Interpreter.  Later: SemanticInterpreter (Gemini Nano), used as an escalation
// when Tier2 returns low confidence on important fields.
```

### 6.2 Components and suggested building blocks

**[Default]** Use prebuilt, free, on-device-capable pieces. Nothing off-the-shelf does intent plus condition end to end, so assemble it. Verify current versions and licensing before committing.

| Job | Suggested approach |
|---|---|
| Dates, times, addresses | **ML Kit Entity Extraction.** Downloads its model once, then runs on-device. |
| Intent (Buy / Visit / Watch / Read / Try / Send / Call / Do / Remember) | Small sentence-embedding model (MiniLM-class, via MediaPipe Text Embedder or TFLite/ONNX) with nearest-neighbor matching against ~30 seed phrases per intent. No training needed to start. Swap in a fine-tuned classifier later if eval data justifies it. |
| Places and people | Quantized BERT-style NER (TFLite/ONNX) for person and location, validated against a **GeoNames** gazetteer for places. Optionally match people against Contacts (permission-gated, optional). |
| Conditions ("when I'm in X", "when I get home", "next time I'm at the mall") | **Deterministic grammar/regex.** This is the core IP; no model gives it to you. |
| Place → coordinates | Android `Geocoder`, or Photon/Nominatim. Needs network. |
| Relative time ("next month", "this weekend", "when I have some time") | Deterministic temporal engine with ML Kit date-time output as input. "When I have some time" maps to Someday, not a date. |

### 6.3 Intent taxonomy

`Buy, Visit, Watch, Read, Try, Send, Call, Do, Remember, Unknown`

Notes:

- `Unknown` is a legitimate output.
- **Visit vs Try** overlap (the mockup labels "Try this restaurant" as Visit). **[Default]** Use *Visit* when a place is the target (restaurant, cafe, exhibition, stay). Use *Try* for things you use or taste (recipe, product, app, food). Document the rule and test it in the eval set.
- Metadata types can inform intent: a `Recipe` page suggests Try; a `Restaurant` suggests Visit; a `Product` suggests Buy; a `Movie`/`VideoObject` suggests Watch; an `Article` suggests Read.

### 6.4 Metadata enrichment (inferring from the shared link)

The user's note is not the only signal. The shared link often tells you what the user meant, especially when they write nothing.

**What the share payload provides**

- `EXTRA_TEXT`: usually the URL, sometimes with a title (YouTube often includes one).
- `EXTRA_SUBJECT` / `EXTRA_TITLE`: sometimes populated, not reliably.
- Instagram typically sends only a link, with no caption or hashtags.

**What can be fetched from the link (from the device, after the share)**

- **Open Graph** tags: `og:title`, `og:description`, `og:image`. Works for most pages. Public Instagram posts may yield a partial caption, but login walls and blocking make it unreliable.
- **Schema.org JSON-LD**: the best signal when present. Types such as `Restaurant`, `Recipe`, `Product`, `Movie`, `Article` map almost directly to intent and often include location.
- **YouTube oEmbed**: title, channel, thumbnail, no API key. Tags and full descriptions require the YouTube Data API and a key; optional, not V1.
- **Instagram** official oEmbed requires an app token. Treat Instagram as **best-effort**. Never build a core flow that depends on it.

**Rules for using metadata**

1. **The user note always wins.** Metadata only fills gaps and never overrides typed intent.
2. **Source-tag every extracted entity** (`user_note` or `page_metadata`) and weight it accordingly. Metadata-derived values start at lower confidence.
3. **Empty note is the big win.** The user shares and taps Keep with no text. KeepThis proposes a chip such as `Visit · Delhi` as a medium-confidence suggestion they can accept or edit.
4. **Do not auto-create geofences from metadata alone.** A restaurant in Delhi doesn't mean the user wants a reminder there. Ask once, or park in Someday until confirmed.
5. **Fail soft.** If the fetch fails or times out, capture still saves instantly with whatever the share payload had. Enrichment can complete in the background and update the item.
6. **Disclose it.** Fetching a URL tells that site the user opened it. Do it only after a share, from the device, and mention it in privacy copy.

### 6.5 Contextual resolution

NLP isn't responsible for everything. Combine language with device context to resolve references:

- "here" → current location at capture (if permission granted; otherwise ask).
- "when I get home" → configured Home place. If Home isn't set, ask once and offer to set it.
- "tonight", "tomorrow" → relative to capture timestamp and time zone.
- "there", "back there" → **cannot be resolved in Tier 2**. Ask (low-confidence flow) or leave unresolved. This is exactly the case Tier 3 is meant to improve later.
- "before my Goa trip" → needs calendar knowledge. In V1, capture it as an unresolved condition and ask for a date or place, or place it in Someday.

### 6.6 Condition semantics

**[Open — recommended default below; decide before building the scheduler.]**

"Try this when I'm in Delhi next month" contains both a place and a time. Proposed rule:

| Condition shape | Fires when |
|---|---|
| Place only | The user is at/near the place, at any time |
| Time only | The time arrives |
| Place **and** time window | The user is at the place **during** the window |
| Window ends without the place being reached | Send one gentle "Still want to do this?" nudge, then offer Reschedule/Forget |
| Unresolved condition | Never fires. Item sits in Someday until clarified. |

The card must make the rule visible (e.g., "In Delhi · Next month") so users aren't surprised.

### 6.7 Interpretation output contract (illustrative)

**[Default]** Shape this however fits the code, but keep per-field value, confidence, and source.

```json
{
  "intent":   { "value": "VISIT",  "confidence": 0.91, "source": "user_note" },
  "places":   [{ "text": "Delhi", "resolved": { "lat": 28.61, "lng": 77.21, "type": "CITY" },
                 "confidence": 0.88, "source": "user_note" }],
  "people":   [],
  "temporal": { "type": "RELATIVE_RANGE", "value": "2026-11-01/2026-11-30",
                "confidence": 0.84, "source": "user_note" },
  "condition":{ "type": "PLACE_AND_TIME", "parameters": { "placeRef": 0, "window": "temporal" },
                "confidence": 0.82 },
  "questions":[]
}
```

(Dates above are examples only.)

---

## 7. Data model

Conceptual model. The model must support **unknown values**; do not require date, location, or intent.

```
KeepItem
  id
  createdAt
  updatedAt

  content                       // optional: absent for manual reminders
    url
    title
    description
    previewImage
    sourceApp
    metadataFetchedAt / metadataStatus

  userNote                      // the user's own sentence, verbatim

  intent
    type
    confidence
    source                      // user_note | page_metadata | user_edit

  entities
    places[]      (text, resolved?, confidence, source)
    people[]
    objects[]
    activities[]

  temporalContext
    type                        // none | absolute | relative | range | vague
    value
    confidence

  locationContext
    type                        // capture_location | explicit | user_place | none
    coordinates
    placeName
    confidence

  condition
    type                        // none | place | time | place_and_time | unresolved
    parameters

  status                        // active | completed | dismissed | expired
  snoozeUntil?
  history[]                     // created, edited, surfaced, completed, dismissed, postponed
```

Notes:

- Store both the **raw user text** and the **parsed structure**. Parsing will improve over time; raw text lets you reinterpret.
- Keep the **history** as an event log; it powers "surfaced N times, snoozed twice" back-off logic and the interpreter eval later.
- Separate **UserPlace** (Home, Work, saved places) from item places.
- **[Default]** Room (SQLite) with FTS for search. Index status, condition type, and next-relevance time for home-screen queries.

---

## 8. Resurfacing: time and location

This is the riskiest and most important part of the product. **Prove it first** (Section 14, M0).

### 8.1 Time-based

- **WorkManager** for non-exact reminders.
- Exact alarms only when truly needed (they require special permission on recent Android versions). Most KeepThis reminders don't need minute precision.
- Re-schedule on reboot, time-zone change, and app update.

### 8.2 Location-based

- **Play Services Geofencing API** plus one-shot **Fused Location** calls. **No continuous tracking.**
- Constraints to design around:
  - Geofences are capped (**100 per app per device user**). Register only the nearest/most relevant ones and refresh the set on significant events.
  - Geofences are cleared on reboot and some other events. Re-register on boot and app start.
  - Background location permission is a separate, scarier user prompt on modern Android.
  - Doze, battery optimizations, and OEM-specific restrictions can delay or drop events. Test on more than one device brand.
- **Place-type-aware radius [Default; verify by prototyping]:**
  - Specific venue (restaurant, cafe): ~200–500 m
  - Neighborhood: ~1–3 km
  - City: large coarse geofence as a trigger, then verify with a one-shot location and reverse geocode before notifying
- **Mockup note:** screen 6 shows a raw "Notification range" slider (2 km). A km slider is wrong for city-level places. Offer presets (**Nearby / Neighborhood / City**), default by place type, and keep a km value as an advanced detail only if needed.

### 8.3 Event-based location checks

Beyond geofences, use one-shot location checks at meaningful moments only: capture, app open, notification action, and occasional low-frequency WorkManager checks if needed. Document battery impact.

### 8.4 Permissions strategy

Location and notifications are friction points. Design for them explicitly:

- Ask **in context** (e.g., first time a place-based condition is created), with a clear rationale screen before the system prompt.
- Location is **optional**. KeepThis must remain useful with time-based and unresolved items if location is denied.
- Request the minimum: approximate vs precise, foreground first, background only when a geofence is actually needed.
- Handle denial gracefully, and show how to enable later from Settings.
- Notification permission (Android 13+) asked at the moment the first reminder is created, not at launch.
- Microphone permission (manual voice capture only) asked the first time the user taps the mic, never at launch. Typing must always work without it.

---

## 9. Architecture

### 9.1 Layers

```
Presentation (Compose UI, ViewModels)
        │
Application / Use cases
        │
Domain  ── Intent engine ── Context engine ── Temporal engine ── Resolution engine
        │   (pure Kotlin, no Android dependencies)
Infrastructure
        ├── Room database
        ├── ML Kit / embedding / NER models
        ├── Metadata fetcher (HTTP)
        ├── Geocoding
        ├── Location + Geofencing
        ├── Notifications + WorkManager
        └── Share intent receiver
```

**[Decided]** The domain layer must not depend on Android APIs.

### 9.2 Suggested stack **[Default]**

- Kotlin, Jetpack Compose, Coroutines/Flow
- Room (with FTS), WorkManager
- Play Services Location and Geofencing
- ML Kit Entity Extraction; MediaPipe or TFLite/ONNX for embeddings and NER
- A lightweight HTTP client and HTML/JSON-LD parser for metadata
- DI: Hilt or Koin (team preference)
- Testing: JUnit for domain, Robolectric/instrumented for Android pieces

### 9.3 Suggested module layout **[Default]**

```
:app                     // navigation, DI wiring
:core:model              // KeepItem, enums, value types
:core:interpreter        // pure Kotlin: grammar, temporal, resolver, interfaces
:core:ml                 // Android: ML Kit + embedding + NER adapters implementing interfaces
:data                    // Room, repositories
:platform                // share receiver, location, geofencing, notifications, work
:feature:capture
:feature:home
:feature:detail
:feature:search
:feature:settings
```

### 9.4 Cross-cutting requirements

- **Never lose a save.** Persist the raw capture first, then enrich.
- **Interpretation is idempotent and re-runnable.** Re-interpret an item when the engine improves.
- **Everything the interpreter decides must be explainable** (source, confidence), and surfaced in a debug view (Section 13).
- **Model downloads** (ML Kit) must degrade gracefully: first run may need network; show a state, and fall back to rules if the model isn't ready.

---

## 10. Privacy

**Claim: local-first.** Honest wording matters. Do **not** claim "fully local" unless a place gazetteer and every model are bundled and nothing calls out.

Stays on the device by default:

- Captured URLs, notes, entities, interpretations, location context, history, and the database.

What may touch the network (disclose all of it):

- One-time **ML Kit model download**.
- **Geocoding** of place names, depending on provider (Android's `Geocoder` may call Google services).
- **Metadata fetching** for shared links (the destination site sees the request), plus preview image fetch.
- **Speech-to-text** (manual voice capture only): some Android speech recognition sends audio to Google servers unless on-device recognition is available. Prefer on-device recognition and disclose the behavior.
- Optional future: YouTube Data API, cloud features.

Principles:

- No accounts, no analytics requiring identity, no mandatory backend.
- Minimal permissions, requested in context.
- No continuous location tracking.
- Provide a clear in-app privacy summary and a "delete everything" action.

---

## 11. Quality, evaluation, and testing

The ML work is part of the portfolio objective, so treat evaluation as a first-class deliverable.

### 11.1 Interpreter eval set

- Build a labeled set of **~200 sentences** (grow over time) covering all intents, places, people, time expressions, and condition shapes, including messy/ambiguous and empty-note cases.
- Label per field: intent, place, person, temporal, condition.
- Track **per-field accuracy**, confusion between similar intents (Visit/Try, Do/Remember), and **calibration** (do "high confidence" results actually turn out right ~80%+ of the time?).
- Include a "should ask" label: cases where the correct behavior is to ask a question, not guess.
- Include **manual-style sentences with no link** ("remind me to call Mom when I get home"), since the phrasing differs from share-style notes ("try this when I'm in Delhi"). Include speech-style transcripts with typical recognition errors on names and places.
- Re-run on every interpreter change. Record results in the repo.

### 11.2 Interpreter test screen (debug build)

A screen where you type a sentence (and optionally pick a metadata fixture) and see: parsed entities, intent, per-field confidence, source tags, the condition, and which question (if any) would be asked. This is the best debugging and demo tool in the project.

### 11.3 Testing priorities

- JVM unit tests for grammar, temporal parsing, and resolver (largest share).
- Instrumented tests for share intent, database, and notifications.
- A **manual resurfacing test plan** on at least two device brands/Android versions: geofence enter, reboot, Doze, battery saver, permission revoked.

### 11.4 Honest failure documentation

Keep a "what didn't work and why" section in the README (e.g., Instagram metadata limits, city-level geofencing quirks, "there" resolution). For a portfolio, this is more convincing than a flawless feature list.

---

## 12. Mockups and what to take from them

![KeepThis high-fidelity mockups](./mockups/keepthis-mockups.png)

*File: `mockups/keepthis-mockups.png`. Twelve high-fidelity screens that convey look and feel and the intended flow. They are directional, not exact specs. Where this document and a mockup disagree, treat the notes below as the intended correction.*

### 12.1 Screen-by-screen

| # | Screen | Take from it | Adjust / watch for |
|---|---|---|---|
| 1 | **Home – Now** | Warm greeting-free header ("Things you want to do later."), filter pills with counts, a "You're in Delhi" context line, cards that show the user's own words plus intent/place chips and "2 weeks ago" | Add or decide on **Waiting** (Section 4.3). Counts on pills should reflect real queries. |
| 2 | **Share from any app** | The proof of the core flow: KeepThis sits in the system share sheet (first position in the mockup) | Real share-sheet ordering is controlled by Android; don't promise placement. Consider a Direct Share target. |
| 3 | **Capture – one sentence** | Strongest screen. Preview card, one text box, live "KeepThis thinks:" chips with Edit, a single **Keep this** button | This must be the **only** screen when confidence is high. Show a counter only if a limit is needed. |
| 4 | **Interpreted details (Review & confirm)** | The idea of a structured, editable view (Intent, Place, When) | **Do not show this for every capture.** It breaks the 10-second promise. Show only for medium/low confidence, and replace the percentage bar with inline questions ("Did you mean Delhi?"). Keep as the full edit view from Edit/More. |
| 5 | **Item detail** | Content and intent separated; user quote with edit icon; chips; free-form notes; related place map; Done / Reschedule / Edit / More | Place map is fine; confirm fetched images are cached locally. |
| 6 | **Location & geofence** | The idea of showing the user the place and what will notify them, with a preview of the notification | Replace the km slider with **Nearby / Neighborhood / City** presets (Section 8.2). Keep the notification preview, it builds trust. |
| 7 | **Upcoming** | Time-based list with intent and "This week" chips; mix of content types | Place-conditioned items need a home here or in Waiting. |
| 8 | **Someday** | Gentle copy: "You'll see these when the time feels right." Low-confidence items live here without shame | Make sure Someday items still get an occasional "Still interested?" so it doesn't become the graveyard. |
| 9 | **Search** | Filter chips (All / Places / Articles / Videos), results show the intent+place chips | Keep search results grouped by meaning (intent/place), not just by content type. |
| 10 | **Contextual notification** | Copy references the reason and the place. Buttons: View, Done | Add **Later**. Keep it sparse. |
| 11 | **Focus mode** | Full-screen "You wanted to try this restaurant." with the user's sentence and Open link. Distinctive, emotional, on-brand | Treat as a V1.1 polish item if time is tight; core flow doesn't depend on it. |
| 12 | **Reschedule** | Options: In 1 day, In 1 week, In 1 month, **Next time I'm in Delhi**, Pick a date, Pick a location, Dismiss | "Next time I'm in X" is a great context-aware snooze. Reuse the condition engine for it. |

### 12.2 Design language to carry forward **[Default, not rigid]**

- Warm off-white/cream background with a deep green primary color for actions.
- Serif display type for the wordmark and emotional headlines (focus mode); clean sans for UI text.
- **Typed chips** as a core visual primitive: intent (e.g., Visit), place, and time chips with distinct soft colors, used identically in capture, cards, detail, and search.
- Generous whitespace, large rounded corners, soft shadows. Image-forward cards using the preview image.
- Calm, personal tone in copy ("Things you want to do later.", "You wanted to try this place."). Avoid task-manager vocabulary like "overdue" or "pending."
- Support **dark mode** and dynamic type/accessibility (contrast on chips, tap target sizes). The mockups show light mode only.

### 12.3 Screens still missing (design needed)

1. **Medium and low confidence states**: "Did you mean Delhi?" **[Yes] [Change]** and the open question for "there." This is the signature interaction and has no screen yet. **Design first.**
2. **Onboarding with permission rationale** for notifications and location (Section 8.4).
3. **Set Home / Work** places.
4. **Empty states** for each home section.
5. **Completed / Dismissed history.**
6. **Settings** (privacy summary, permissions status, delete all data, notification limits).
7. **Metadata-failed state** (no preview available).
8. **Model downloading / interpreter-not-ready state.**
9. **Edit interpretation** (the full structured view from screen 4, reached on demand).
10. **Manual capture variant** (secondary): capture screen without a preview card, with a mic button and a recording state. Text-only item card for items with no shared content.

---

## 13. Debug and demo tooling

- **Interpreter lab** (Section 11.2).
- **Fixtures** for metadata (sample OG/JSON-LD pages) so the pipeline can be demoed without network.
- **Time and location simulator** (debug build) to fake "it's next month" or "I'm in Delhi" so resurfacing can be demonstrated without traveling.
- A **log view** of surfacing decisions: why did/didn't this item notify?

---

## 14. Suggested build order

Each milestone has an exit criterion. Reorder if you learn something, but do M0 first.

| Milestone | Goal | Exit criterion |
|---|---|---|
| **M0 – Resurfacing spike** | Share a link, type "try this when I'm in Delhi", get a geofenced notification. Ugly UI is fine. | Notification fires reliably on two different devices, survives reboot, and you've documented where it fails. |
| **M1 – Capture + storage** | Share receiver, capture screen, Room schema, instant save | Share → Keep in a few seconds, offline-safe, nothing is ever lost. |
| **M2 – Interpreter v1** | Temporal engine, condition grammar, intent via embeddings, place/person extraction; interpreter lab; first eval set | Eval numbers recorded; lab screen works. |
| **M3 – Confidence UX** | High/medium/low flows at capture; source tags; metadata enrichment and empty-note inference | "Did you mean Delhi?" works end to end; empty-note shares propose a sensible chip. |
| **M4 – Home, detail, actions** | Now / Upcoming / Someday (and Waiting decision), item detail, Done / Reschedule / Dismiss, search | Full loop usable day to day. |
| **M5 – Notifications and permissions** | Contextual copy, actions (View/Done/Later), back-off rules, onboarding rationale, Home/Work setup | Permission denial paths all behave; notification volume is sane. |
| **M5b – Manual capture (secondary)** | Add tab with typed entry reusing the capture screen; optional mic button for voice; content-less items | A typed or spoken sentence becomes a normal item through the same interpreter, with no extra fields. Skip or trim if time is short. |
| **M6 – Polish and portfolio** | Focus mode, dark mode, accessibility pass, eval write-up, README with failure notes, demo script | A new person can watch a 2-minute demo and understand the promise. |

---

## 15. Risks and open questions

### Risks

| Risk | Why it matters | Mitigation |
|---|---|---|
| **Location resurfacing is flaky** | It is the defining experience. Doze, OEM battery rules, permission friction, geofence caps. | Prototype first (M0), test on multiple devices, document limits, keep a time-based fallback. |
| **Tier 2 NLP is thinner than it sounds** | ML Kit covers dates/addresses, not people, intents, or general place names. | Embedding-based intent + NER + rules; invest in the eval set; keep expectations honest. |
| **Instagram metadata is unreliable** | The flagship source often provides only a link. | Treat as best-effort; rely on the user note; degrade gracefully. |
| **Place resolution and the privacy claim** | Geocoding may call external services. | Disclose; consider a bundled gazetteer later. |
| **City-level conditions** | A "city" geofence is imprecise and noisy. | Coarse trigger + one-shot verification; place-type-aware radius. |
| **Notification fatigue** | The product promises sparse, meaningful reminders. | Per-item and per-day caps, back-off on dismissals. |
| **Scope creep** | The out-of-scope list is the product's discipline. | Hold it. |
| **Manual reminders pull the app toward a task manager** | It would blur the positioning and dilute the share-first story. | Keep it secondary, one sentence only, no due-date or priority fields (Section 4.8). |
| **Speech transcription errors** | Names and places are often misheard, which corrupts entities. | Never auto-save voice input; show the editable transcript and chips; test with speech-style eval data. |

### Open questions

1. **[Open]** Condition semantics for place + time (Section 6.6).
2. **[Open]** Waiting as a section vs. grouping within Upcoming (Section 4.3).
3. **[Open]** Geocoding provider and the exact privacy wording.
4. **[Open]** Visit vs Try boundary, and whether to add a "Go" or "Stay" intent (the mockups show "Stay").
5. **[Open]** DI framework and minimum Android version (affects geofencing/notification permission handling).
6. **[Open]** How to handle shared content that is not a URL (plain text, images). V1 can start with URL/text only.
7. **[Open]** Whether to support multiple conditions on one item ("when I'm in Delhi" and "next month").
8. **[Open]** Where the intent seed phrases and eval data live in the repo, and how contributors extend them.
9. **[Open]** Whether voice input ships in V1 or after (typed manual capture is the minimum), and whether manual items need any visual distinction from shared items beyond the missing preview image.

---

## 16. Glossary

- **Intention:** the reason a user kept something, in their own words.
- **Condition:** what must be true for an item to become relevant (place, time, or both).
- **Resurfacing:** bringing an item back to the user at a relevant moment.
- **Tier 2:** the V1 on-device interpreter (ML Kit + small models + rules + context).
- **Tier 3:** the later optional interpreter using Gemini Nano via AICore for ambiguous language.
- **Source tag:** where a value came from (`user_note`, `page_metadata`, `user_edit`).
- **Chip:** the small typed label (intent, place, time, person) used throughout the UI.

---

## 17. Appendix: starting artifacts worth producing next

These are good first documents or code files for the team or an agent to create:

1. **Intent seed phrase list** (~30 per intent, including edge cases).
2. **Condition grammar spec** (patterns for "when I'm in X", "when I get home", "next time I'm at X", "before X", time windows) with test cases.
3. **Room schema** and migration plan.
4. **Confidence weighting and source-tagging rules** as a short spec.
5. **Eval set template** (CSV/JSON) and the evaluation script.
6. **M0 spike plan** with the device test matrix.
