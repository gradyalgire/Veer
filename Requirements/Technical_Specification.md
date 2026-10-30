# Veer Technical Specification

**Course:** Computer Science Senior Design I\
**Advisor:** Dr. Jillian Aurisano\
**Team:** Grady Algire, Brady Cooper, Briar Elliot, Dominic Rowland, Aiden Ward\
**Document version:** 1.0 | October 30, 2026\
**Traces to:** [Requirements Document](Requirements_Document.md) **version 1.0** (October 30, 2026)

---

## 1. Version and Change Log

| Version | Date | Change | Author |
| --- | --- | --- | --- |
| D0 1.0 | 2026-10-02 | High-level design: C1 to C4, E1 to E4, interfaces I1 to I9, decisions D01 to D10. | Team |
| D1 1.1 | 2026-10-08 | C2 and C3 detailed: data model, allow-list validation, filter-then-rank selection, I2/I4 contracts, technology stack. | Team |
| D2 1.2 | 2026-10-13 | C1 state machine, C4, and glue components. I1 split into I1a/I1b/I1c and approval replies moved to I2 (R-04): D0's I1 mixed unrelated jobs and could not be tested alone. UsageEvent fields renamed (provider → service, attemptedAt → occurredAt, outcome values ok/error/throttled) and estCost added. MuteError exit and network/GPS-loss edge cases (EC-33, EC-34) added after peer review. | Team |
| 1.0 | 2026-10-30 | Consolidated D0 to D2 into this specification, traced to Requirements Document v1.0. Two small changes since the Week 7 review: (1) auth, config, and usage endpoints now use the `/api/v1/` prefix, because D1 §5.4 requires it for every endpoint and D2 had left it off; (2) the I2 speech stream path is fixed as `/api/v1/trips/{tripId}/speech` so C1 and C2 can be built against it (audio encoding stays open under G-08). GitHub Actions selected for CI. | Aiden Ward |

---

## 2. Architecture Summary

Veer is a client-server system with a layered backend and a pipeline for speech ([D0 Block Diagram](Design_Diagrams/D0_Block_Diagram.png); full design in [D0](Design_Diagrams/D0_High_Level_Design.pdf), [D1](Design_Diagrams/D1_Detailed_Design_Source.pdf), and [D2](Design_Diagrams/D2_Detailed_Design.pdf)).

**Components.** **C1 Mobile Navigation** is the Android app. It is the only component that can change the active route or control the microphone, and it runs the trip state machine (D2 Figure 1). The backend holds three logical modules: **C2 Speech Understanding** turns an utterance into a validated, allow-listed constraint; **C3 Route Planning** turns constraints into a route proposal by filtering then ranking candidates; **C4 Usage Monitor** counts and caps provider calls. Four glue components support them: authentication, notifications, configuration, and logging. External dependencies are **E1** Android, **E2** OpenAI, **E3** Google Maps Platform, and **E4** Open-Meteo.

**Deployment.** C1 installs on the driver's own phone. C2, C3, C4, and the glue components run as one Node.js/TypeScript service on Render with a managed PostgreSQL database. They are modules in one service, not microservices (D0 §6). Provider API keys exist only on the server.

**Data flow** ([D0 Data Flow](Design_Diagrams/D0_Data_Flow.png), [D0 Usage Flow](Design_Diagrams/D0_Usage_Flow.png)).

- *Speech to route.* C1 buffers an utterance in memory (I1a) and streams it to C2 (I2). C2 gets a transcript and candidate constraint from E2 (I3), checks it against the allow-list, and returns it. C1 sends the constraint with trip context to C3 (I4). C3 fetches routes and places from E3 (I5) and, for rain, weather from E4 (I6), then returns a proposal. C1 speaks a confirmation of 15 words or fewer, waits for "okay" (interpreted by C2 over I2), and activates the route within 3 seconds. Audio is discarded after each step.
- *Mute.* A tap or the local phrase "Veer, stop listening" closes the send gate, clears the buffer, cancels in-flight requests, and stops the mic on the phone. No network call is involved.
- *Usage.* Before each paid call C2 and C3 ask C4 for quota; after it they record a usage event (I7, I8). Developers read per-trip totals through I9. No audio, transcript, or location reaches C4.

---

## 3. Interfaces and Data Formats

### 3.1 Component Boundaries

Every network request carries `requestId`. Every failure returns `Error = {code: string, message: string, retryable: boolean}`, and no failure ever applies a route (D0 §4).

| Boundary | Inputs | Outputs | Format | Protocol | Error behavior |
| --- | --- | --- | --- | --- | --- |
| I1a C1 ↔ E1 Mic | micControl (start, stop, forceStop); MuteEvent | PCMFrame[]; micState (listening, muted, error) | 16-bit mono PCM in memory; typed events | Android OS calls and callbacks | Mic still on 1 s after mute: force-stop, enter MuteError, drop post-request audio (EC-04). |
| I1b C1 ↔ E1 Location | locationControl (start, stop) | LocationFix | Typed record | Android OS callbacks | GPS lost: keep guiding on lastLocation, say "GPS signal lost" once, send no I4 until a fix returns (EC-34). |
| I1c C1 ↔ E1 Driver I/O | spokenText; destination entry (before driving) | Speech, screen, notifications | Strings; LatLng | Android TTS and UI | Notification permission denied: spoken notice only (EC-24). |
| I2 C1 → C2 | tripId, utteranceId, control (audio, cancel), audio, AudioMetadata | transcript, intent (propose, accept, reject, unknown), Constraint?, confidence (0 to 1), Error? | Binary audio frames plus JSON | Authenticated WSS (TLS) | invalid_audio 400; unsupported_constraint 422; unclear_speech 422; quota 429; provider_timeout 504. C1 asks for a repeat within 2 s and keeps the route. |
| I3 C2 → E2 | audio, metadata, allowedConstraintTypes, allowedIntents | transcript, intent, Constraint?, confidence, ProviderUsage | Provider audio stream plus JSON | Authenticated WSS (TLS) | Timeout, quota, or invalid output becomes an I2 error. Provider must not retain audio (G-03). |
| I4 C1 → C3 | tripId, LocationFix, destination, currentRoute?, Constraint[], tripRevision | RouteProposal? or Error | JSON | HTTPS POST | no_match 404/422; weather_unavailable 503; quota 429; timeout 504; stale_revision 409. All keep the active route. |
| I5 C3 → E3 | origin, destination, waypoints, query?, avoidHighways, avoidTolls, alternatives | RoutePlan[], Place[], attribution | Google JSON normalized to typed records | HTTPS | No match within 600 s: no_match. Quota or timeout: preserve guidance (EC-12). |
| I6 C3 → E4 | {location, expectedAt}[] | WeatherSample[], fetchedAt | Open-Meteo JSON normalized to typed records | HTTPS | Stale or missing data: warn and do not claim rain avoidance. |
| I7 C2 → C4, I8 C3 → C4 | checkQuota(service); recordUsage(tripId, service, operation, units, outcome) | allow, allow_warn, deny; Unit | Typed records | In-process call | Deny returns quota to C1. Database down: queue up to 500 events, retry every 30 s, allow calls (EC-16). |
| I9 Developer → C4 | from, to, groupBy; admin key | Counts and estimated cost per trip, day, or service | JSON | HTTPS GET | 400 invalid_range; 401 unauthorized; 403 forbidden; 503 usage_store_unavailable. |

### 3.2 Endpoint Table

| Endpoint | Interface | Auth | Success response | Errors |
| --- | --- | --- | --- | --- |
| `WSS /api/v1/trips/{tripId}/speech` | I2 | Access token | Interpretation result per utterance | See I2 |
| `POST /api/v1/trips/{tripId}/route-proposals` | I4 | Access token | `{proposal: RouteProposal}` | See I4 |
| `POST /api/v1/auth/register` | Auth | None, rate-limited per IP | accessToken, refreshToken, expiresIn (3600) | 429 rate_limited; 503 auth_unavailable |
| `POST /api/v1/auth/refresh` | Auth | Refresh token | accessToken, expiresIn | 401 invalid_refresh; 403 token_revoked |
| `GET /api/v1/config/client` | Config | Access token | version and client config values (no secrets) | 401 unauthorized; 503 config_unavailable (app uses cache) |
| `GET /api/v1/usage/report?from=&to=&groupBy=` | I9 | Admin key | Usage summaries | See I9 |

All paths use the `/api/v1/` prefix (D1 §5.4). A breaking change to any of them requires `/api/v2/`.

### 3.3 Message Payloads

Shared types follow D0: string = UTF-8, instant = ISO 8601 UTC, durations in seconds, distances in meters, `LatLng = {lat, lon}` in WGS84, `?` = optional.

| Record | Fields |
| --- | --- |
| Constraint | type: add_waypoint, avoid_precipitation, avoid_highways, avoid_tolls; value: string?; maxDetourSeconds: integer? |
| LocationFix | position: LatLng; accuracyMeters: number; capturedAt: instant |
| AudioMetadata | encoding: string; sampleRateHz: integer; channels: integer; utteranceEndedAt: instant |
| Place | placeId: string; name: string; location: LatLng |
| RoutePlan | routeId: string; geometry: LatLng[]; durationSeconds; distanceMeters; steps: RouteStep[] (instruction, location, sequence) |
| RouteProposal | tripRevision: integer; route: RoutePlan; selectedPlace: Place?; extraSeconds: number; confirmationText: string (≤ 15 words); warnings: string[] |
| WeatherSample | location: LatLng; validAt: instant; precipitationProbability: number? (0 to 1); precipitationRateMmPerHour: number? |

Constraint as a JSON Schema (the allow-list C2 enforces, D1 §3.1):

```json
{
  "type": "object",
  "required": ["type"],
  "properties": {
    "type": {"enum": ["add_waypoint", "avoid_precipitation", "avoid_highways", "avoid_tolls"]},
    "value": {"type": "string", "minLength": 1},
    "maxDetourSeconds": {"type": "integer", "minimum": 0, "maximum": 600}
  },
  "if": {"properties": {"type": {"const": "add_waypoint"}}},
  "then": {"required": ["value"]},
  "additionalProperties": false
}
```

I4 request and reply (D1 §5.4):

```json
POST /api/v1/trips/trip-17/route-proposals
{"requestId": "req-18", "tripId": "trip-17",
 "location": {"position": {"lat": 39.1031, "lon": -84.5120}, "accuracyMeters": 8, "capturedAt": "2026-10-30T21:00:00Z"},
 "destination": {"lat": 39.9612, "lon": -82.9988}, "currentRoute": null,
 "constraints": [{"type": "add_waypoint", "value": "McDonald's", "maxDetourSeconds": 600}],
 "tripRevision": 4}

200 OK
{"proposal": {"tripRevision": 4,
  "selectedPlace": {"placeId": "place-123", "name": "McDonald's", "location": {"lat": 39.18, "lon": -84.47}},
  "extraSeconds": 420, "confirmationText": "McDonald's adds about seven minutes.", "warnings": []}}
```

### 3.4 Database Schema

C1 keeps its TripSession and AudioBuffer (15 s cap) in phone memory only. C2 and C3 routing state is in memory and cleared at trip end. PostgreSQL holds only the tables below (D2 §2.2, §2.3).

```sql
CREATE TYPE service_t AS ENUM ('ai', 'maps', 'weather');

CREATE TABLE trip_usage (            -- one row per trip; deleted after 12 months
  trip_id        TEXT PRIMARY KEY,
  ai_calls       INTEGER NOT NULL DEFAULT 0,
  maps_calls     INTEGER NOT NULL DEFAULT 0,
  weather_calls  INTEGER NOT NULL DEFAULT 0,
  est_cost_total NUMERIC NOT NULL DEFAULT 0);

CREATE TABLE usage_event (           -- one row per call attempt; deleted after 90 days
  event_id    TEXT PRIMARY KEY,      -- caller-supplied; deduplicates retries
  trip_id     TEXT NOT NULL REFERENCES trip_usage(trip_id),
  service     service_t NOT NULL,
  operation   TEXT NOT NULL,
  occurred_at TIMESTAMPTZ NOT NULL,
  units       INTEGER NOT NULL,
  est_cost    NUMERIC NOT NULL,
  outcome     TEXT NOT NULL CHECK (outcome IN ('ok', 'error', 'throttled')));
CREATE INDEX ON usage_event (trip_id);

CREATE TABLE monthly_budget (        -- kept with no limit; holds no trip data
  service service_t NOT NULL,
  month   CHAR(7) NOT NULL,          -- e.g. '2026-10'
  used    NUMERIC NOT NULL DEFAULT 0,
  state   TEXT NOT NULL CHECK (state IN ('OK', 'Warning', 'Exhausted')),
  PRIMARY KEY (service, month));

CREATE TABLE installation (
  install_id         TEXT PRIMARY KEY,
  created_at         TIMESTAMPTZ NOT NULL,
  status             TEXT NOT NULL CHECK (status IN ('active', 'revoked')),
  refresh_token_hash TEXT NOT NULL); -- raw token never stored

CREATE TABLE alert_state (           -- stops duplicate budget alerts
  service      service_t NOT NULL,
  month        CHAR(7) NOT NULL,
  level        SMALLINT NOT NULL CHECK (level IN (80, 100)),
  last_sent_at TIMESTAMPTZ NOT NULL,
  PRIMARY KEY (service, month, level));
```

The cap check is one atomic statement that refuses to pass the cap (D2 R-07), for example `UPDATE monthly_budget SET used = used + $cost WHERE service = $s AND month = $m AND used + $cost <= $cap RETURNING used;` where zero rows returned means deny.

---

## 4. Technology Stack

| Layer | Choice | Justification |
| --- | --- | --- |
| Mobile app | Android, Kotlin | Runs on the driver's existing phone (CON-01); Keystore for the refresh token (NFR-03); foreground service keeps mic and network alive (NFR-09); local mute path (REQ-11). |
| Backend | Node.js, TypeScript, Express | Integration work is I/O-bound with bounded timeouts (NFR-10); static types match the allow-list and contracts (REQ-06). MIT licensed. |
| Database | PostgreSQL (Render managed) | Atomic cap update (NFR-04); reliable trip and event IDs for usage records (REQ-14); retention job (NFR-08). |
| Speech and interpretation | OpenAI API | Hosted transcription and structured interpretation without training a model (REQ-01, REQ-06; CON-01). Accepted only with a written no-retention setting (NFR-01, G-03); fallback is on-device transcription (RK-01). |
| Routes, places, map | Google Maps Platform | Routes, place search, and base map (REQ-01, REQ-09, REQ-15); terms bound the design (CON-04). |
| Weather | Open-Meteo | Location- and time-indexed precipitation at low cost (REQ-08; CON-01). |
| Hosting | Render | One small web service plus database, smallest practical tier (CON-01); secrets in environment variables (NFR-03, NFR-06). |
| Authentication | Anonymous install tokens; 1-hour access token | No accounts needed; keys stay on the server (NFR-03). |
| Configuration | Versioned config file checked against a schema at startup | Tunable timeouts, thresholds, caps, retention (NFR-06). |
| Logging | JSON lines on Render, 30 days, redaction filter; Logcat in debug builds only | Traceable by requestId with no private data (NFR-05, NFR-01). |
| Transport | TLS for HTTPS and WSS | Protects audio and tokens in transit (NFR-01, NFR-03). |
| CI | GitHub Actions | Free for the team's GitHub repository (CON-01); runs tests with mocked providers on every push so contract changes are caught before integration (NFR-02, RK-08). |

---

## 5. Known Constraints

| ID | Constraint | Rules out | Forces |
| --- | --- | --- | --- |
| CON-01 | Team-funded; free tiers and Maps credits; driver's own phone. | Purpose-built hardware; paid services beyond the caps; a distributed cloud setup. | One Render service and smallest database; C4 caps and 80/100% alerts (NFR-04); mocks in tests (RK-05). |
| CON-02 | No raw audio retained; indicator and one-tap mute before any in-car demo. | Audio files, audio or transcripts in logs, debug recordings, providers that retain audio. | In-memory AudioBuffer capped at 15 s; local mute path (D02); E2 no-retention gate (G-03) with on-device fallback; trip-scoped constraints. |
| CON-03 | ORC 4511.204: voice-operated use only while driving. | Any feature that needs a screen tap while moving, including tap-to-approve. | Spoken "okay" approval (D07); destination chosen before driving; tap mute is for passengers or a stopped driver. |
| CON-04 | Google Maps terms: place data cached 30 days at most; no competing base map. | Swapping in a cheaper tile provider; storing provider data by default. | Google base map; any future cache expires within 30 days; if in-app guidance is not allowed, hand off to the Google Maps app (G-04). |
| CON-05 | One short sentence per spoken reply. | Long spoken explanations. | 15-word confirmation limit; EC-05 replaces long text with a place-and-minutes summary. |
| CON-06 | Rank businesses only by stated preference and detour cost. | Paid placement; popularity ranking. | C3 filter-then-rank with a deterministic tie-break on lower extraSeconds, then shorter duration (D1 §3.2). |

---

## 6. Traceability Matrix

Every REQ, NFR, and CON in Requirements Document v1.0 has one row. No row is orphaned. Test case IDs TC-01 to TC-32 are reserved for the Week 10 test plan, one per row.

| Req ID | Description | Design Component (D0/D1/D2) | Status | Test Case |
| ------ | ----------- | --------------------------- | ------ | --------- |
| REQ-01 | Add a stop by voice, detour ≤ 600 s | D1 §3.2 filter-then-rank; C2 add_waypoint; I4; I5 | Designed | TC-01 |
| REQ-02 | Confirmation ≤ 15 words | RouteProposal.confirmationText; config confirmation.maxWords; D2 EC-05 | Designed | TC-02 |
| REQ-03 | Activate only after "okay" | D2 Figure 1 AwaitingApproval; onApprovalReply; D0 D07 | Designed | TC-03 |
| REQ-04 | Route active ≤ 3 s after "okay" | D2 Figure 1 Activating; activation.maxSec; D1 §3.2 proposal prepared before approval | Designed | TC-04 |
| REQ-05 | No match: keep route, prompt ≤ 3 s | C3 no_match; D2 EC-09; C1 Recovery | Designed | TC-05 |
| REQ-06 | Unparseable: keep route, prompt ≤ 2 s | D1 §3.1 allow-list validation; unclear_speech; D2 EC-07; recovery.promptSec | Designed | TC-06 |
| REQ-07 | "No" gives next candidate | D2 Figure 1 AwaitingApproval → Planning; I4 | Designed | TC-07 |
| REQ-08 | Avoid rain, warn if no data | D1 §3.2 weather ranking; I6; E4; weather.* config; D2 EC-12 | Designed | TC-08 |
| REQ-09 | Avoid highways and tolls | D1 §3.2 avoidance flags; I5 avoidHighways/avoidTolls | Designed | TC-09 |
| REQ-10 | Mic state always shown | D2 TripSession.micState; I1a; foreground notification (§2.3.2) | Designed | TC-10 |
| REQ-11 | Mute ≤ 1 s by tap or voice | D2 mute(); I1a; D0 D02; Figure 1 Muted | Designed | TC-11 |
| REQ-12 | Force-stop ≤ 2 s, 0 s post-mute audio | D2 MuteError; AudioBuffer; EC-04 | Designed | TC-12 |
| REQ-13 | Unmute by tap only | D2 unmute(); Figure 1 Muted → Listening | Designed | TC-13 |
| REQ-14 | Per-trip usage counts and report | D2 §2.2 UsageEvent, TripUsage, recordUsage; I7, I8, I9 | Designed | TC-14 |
| REQ-15 | Turn-by-turn spoken guidance | D0 C1; I1b, I1c; I5 RouteStep | Designed | TC-15 |
| NFR-01 | 0 raw audio files; no-retention provider | D2 AudioBuffer; logging redaction; D0 D03; I3 provider gate | Designed | TC-16 |
| NFR-02 | Keep route on any failure | D2 Figure 1 Recovery; EC-01 to EC-34; D0 D08 | Designed | TC-17 |
| NFR-03 | No keys in app; Keystore; 1 h tokens | D2 §2.3.1 Installation, /api/v1/auth/register, /api/v1/auth/refresh | Designed | TC-18 |
| NFR-04 | Alert at 80%, deny at 100%, no overrun | D2 §2.2 MonthlyBudget, checkQuota; R-07; EC-13, EC-14 | Designed | TC-19 |
| NFR-05 | JSON logs with no private data | D2 §2.3.4 logger, requestId, redaction filter; EC-29 | Designed | TC-20 |
| NFR-06 | All tunables in validated config | D2 §2.3.3 config schema; GET /api/v1/config/client; EC-25 to EC-28 | Designed | TC-21 |
| NFR-07 | ≤ 15-word replies; no taps while moving | D0 D09; confirmation.maxWords; I1c; notifyDriver | Designed | TC-22 |
| NFR-08 | 90-day / 12-month retention | D2 §2.2 retention job; retention.* config | Designed | TC-23 |
| NFR-09 | Mic and network work in background | D2 §2.3.2 foreground notification; I1a | Designed | TC-24 |
| NFR-10 | p95 latency for I2 and I4 under load | D1 §3.1 and §3.2 complexity and timeouts; timeout.interpretSec, planSec | Planned | TC-25 |
| NFR-11 | ≥ 8 of 10 phrases correct, with accents | D1 §3.1 allow-list validation; task T10; milestone M-18 | Planned | TC-26 |
| CON-01 | Team-funded, existing phone | D0 §6 project constraints; D01; C4 caps | Designed | TC-27 |
| CON-02 | No raw audio; indicator and mute | D0 D02, D03; D2 AudioBuffer, mute() | Designed | TC-28 |
| CON-03 | Voice-only use while moving | D0 D07; D2 I1c | Designed | TC-29 |
| CON-04 | Google Maps caching and base-map terms | D0 D05; D1 §6.5 | Designed | TC-30 |
| CON-05 | One short sentence per reply | D0 D09; D2 EC-05 | Designed | TC-31 |
| CON-06 | Rank by preference and detour only | D0 D10; D1 §3.2 filter-then-rank | Designed | TC-32 |

---

## 7. Advisor Review Record

Fill this in after the review meeting with Dr. Aurisano, then add a v1.1 row to both change logs listing each edit the review produced.

| Item | Record |
| --- | --- |
| Date drafts sent to advisor | |
| Date of review meeting | |
| Attendees | |
| Documents reviewed | Requirements Document v1.0; Technical Specification v1.0 (including the traceability matrix) |
| What was discussed | |
| Changes it produced | |
