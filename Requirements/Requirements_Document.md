# Veer Requirements Document

**Course:** Computer Science Senior Design I\
**Advisor:** Dr. Jillian Aurisano\
**Team:** Grady Algire, Brady Cooper, Briar Elliot, Dominic Rowland, Aiden Ward\
**Document version:** 1.0 | October 30, 2026

---

## 1. Version and Change Log

| Version | Date | Change | Author |
| --- | --- | --- | --- |
| 0.1 | 2026-09-26 | User stories US-01 to US-05, use cases UC-01 and UC-02, acceptance criteria AC-01.1 to AC-02.2 (User_Stories.md). | Team |
| 0.2 | 2026-10-02 | D0: explicit spoken "okay" required; silence no longer approves a route (decision D07). Supersedes UC-01 step 6. | Team |
| 0.3 | 2026-10-13 | D2: non-functional requirements NFR-01 to NFR-09 and MoSCoW priorities for stories and acceptance criteria (D2 §4). | Team |
| 1.0 | 2026-10-30 | Consolidated into this document. Functional requirements given stable REQ-## IDs; acceptance criteria restated as requirements; NFR-01 to NFR-09 kept with their D2 IDs and given thresholds; NFR-10 (performance) and NFR-11 (speech accuracy) added; constraints given CON-## IDs. Priorities rebalanced so Must is half of the set: REQ-02, REQ-04, REQ-05, REQ-14, and NFR-09 moved from Must (D2) to Should, because each is either covered by another Must or not needed for a working first demo. Usage retention of 90 days / 12 months approved (closes G-05). | Aiden Ward |

---

## 2. Purpose and Scope

**Purpose.** This document states what Veer must do and for whom. It is the single list of requirements that the Technical Specification (Technical_Specification.md) traces to and that the Week 10 test plan verifies. It adds no new design.

**Product.** Veer is an Android navigation app for a driver using a phone mounted in their own car. The driver speaks requests such as "I'm hungry for McDonald's" or "avoid the rain." Veer turns each request into a typed routing constraint, proposes a revised route, reads back a one-sentence summary, and switches routes only after the driver says "okay." Raw audio is never stored, and a passenger can mute the microphone with one tap.

**In scope for the first release** (Project_Description.md, D0 §1):

- Voice capture with a visible listening indicator and mute
- Transcription and constraint extraction from natural speech
- Waypoint insertion by business or category
- Route preferences: avoid highways, avoid tolls, avoid rain
- Turn-by-turn routing with spoken guidance
- Per-trip AI, Maps, and weather usage tracking for the development team
- A mobile app on the driver's existing Android phone

**Out of scope:** see the Won't list in Section 7.

---

## 3. Stakeholders and User Goals

From User_Stories.md (Week 4).

| Category | Stakeholder | Goal | Story |
| --- | --- | --- | --- |
| Primary | Driver | Add a food stop by saying what they want, without leaving the route by more than a few minutes. | US-01 |
| Primary | Driver | Tell Veer to avoid rain and spend less time driving in bad weather. | US-02 |
| Secondary | Veer development team | See how many AI and Maps calls each trip uses, to stay within budget. | US-03 |
| Secondary | Google Maps Platform | Routes and places used within its terms (caching, base map). | CON-04 |
| Hidden | Passenger | Know when the app is listening and be able to stop it. | US-04 |
| Hidden | Driver with an accent or speech difference | Have Veer confirm what it heard before changing the route. | US-05 |
| Hidden | Ohio law (ORC 4511.204) | No handheld phone use while driving. | CON-03 |
| Hidden | Businesses shown as stops | Selected only by stated preference and detour cost. | CON-06 |

---

## 4. Glossary

Each term is defined once here and used with this meaning in both documents.

| Term | Definition |
| --- | --- |
| Active route | The route the driver is following now. Only C1 can replace it. |
| Allow-list | The fixed set of constraint types and intents C2 accepts from the AI provider. Anything else is rejected. |
| Approval | The driver saying "okay" after a route proposal. Silence or an unclear reply is not approval. |
| C1 to C4 | Components the team builds: C1 Mobile Navigation, C2 Speech Understanding, C3 Route Planning, C4 Usage Monitor. |
| Constraint | A typed routing rule from speech: `add_waypoint`, `avoid_precipitation`, `avoid_highways`, or `avoid_tolls`. |
| Detour (extra time) | Added travel time, in seconds, of a proposed route compared with the active route. |
| Developer | A member of the Veer development team using the usage report. |
| Driver | The person driving with Veer running on a phone mounted in their own car. The primary user. |
| E1 to E4 | External dependencies: E1 Phone Platform (Android), E2 AI Provider (OpenAI), E3 Google Maps Platform, E4 Weather Provider (Open-Meteo). |
| Edge case (EC-##) | A boundary, timing, bad-input, or dependency failure case listed in D2 §3.1. |
| I1 to I9 | Interfaces between components, defined in D0 §4 and revised in D2 (I1 split into I1a, I1b, I1c). |
| Installation | One anonymous app install, identified by a random installId. Veer has no driver accounts. |
| Listening indicator | The on-screen and notification display of the mic state: listening, muted, or error. |
| Monthly cap | The team-set spending limit per service (AI, Maps, weather) per calendar month. |
| Mute | Stopping the microphone by one tap or the phrase "Veer, stop listening." |
| MuteError | The state entered when the mic does not stop after a mute request. |
| Passenger | A person in the car who is not a user of Veer and has not agreed to anything. |
| Raw audio | Microphone samples. Never written to disk, logs, or any server store. |
| Recovery | The C1 state that speaks a short prompt and keeps the active route after any failure. |
| Route proposal | A candidate route from C3, with optional place, extra time, confirmation text, and warnings. Not active until approved. |
| Trip | One drive from start to arrival or end. Constraints and route state last only for the trip. |
| Trip revision | A counter that goes up by 1 each time a route is activated. Replies with an older revision are ignored. |
| Usage event | A record of one AI, Maps, or weather call attempt: service, operation, units, estimated cost, outcome. |
| Utterance | One spoken request, capped at 15 seconds of audio. |

---

## 5. Functional Requirements

Verification methods: **T** = test, **D** = demonstration, **I** = inspection, **A** = analysis. "Source" gives the story and, where one exists, the acceptance criterion or design decision the requirement restates.

| ID | Requirement | Priority | Source | Verification |
| --- | --- | --- | --- | --- |
| REQ-01 | The system shall, when the driver asks by voice for a business or category of stop, propose a route that adds a matching place with a detour of 600 seconds (10 minutes) or less. | Must | US-01; AC-01.1 | T |
| REQ-02 | The system shall speak a confirmation of 15 words or fewer for every route proposal before it can be activated. | Should | US-05; AC-01.1; EC-05 | T |
| REQ-03 | The system shall activate a proposed route only after the driver says "okay"; silence or an unclear reply shall leave the active route unchanged in 100% of trials. | Must | US-05; D07 | T |
| REQ-04 | The system shall make an approved route active in both the map and turn-by-turn guidance within 3 seconds after the driver finishes saying "okay." | Should | US-01; AC-01.1 | T |
| REQ-05 | The system shall, when no matching place adds 600 seconds or less, keep the active route and ask for a different place within 3 seconds of the request. | Should | US-01; AC-01.3 | T |
| REQ-06 | The system shall, when speech cannot be turned into an allow-listed constraint, keep the active route and ask the driver to repeat within 2 seconds after speech ends. | Must | US-05; AC-01.4 | T |
| REQ-07 | The system shall, when the driver says "no" to a proposal, offer the next-best candidate in one sentence. | Should | US-01 (UC-01 alternate flow) | T |
| REQ-08 | The system shall, when the driver asks to avoid rain, rank candidate routes by lower precipitation exposure within the configured weather detour limit, and shall warn rather than claim avoidance when weather data is unavailable or stale. | Should | US-02 | T (replayed weather data) |
| REQ-09 | The system shall apply spoken "avoid highways" and "avoid tolls" requests by ranking routes that satisfy the flag ahead of routes that do not. | Could | Project_Description.md first-release scope (no story yet; see Q-02) | T |
| REQ-10 | The system shall show the mic state (listening, muted, or error) on screen and in the trip notification for 100% of the time a trip is active. | Must | US-04 | I, D |
| REQ-11 | The system shall stop the microphone within 1 second of a mute tap or a locally recognized "Veer, stop listening," and then show muted. | Must | US-04; AC-02.1 | T |
| REQ-12 | The system shall, if the microphone has not stopped, force-stop it within 2 seconds of the original mute request, show an error, and send or store 0 seconds of audio captured after the request. | Must | US-04; AC-02.2 | T |
| REQ-13 | The system shall restart listening only after an unmute tap. | Should | US-04 (UC-02 alternate flow) | D |
| REQ-14 | The system shall record every AI, Maps, and weather call attempt (including retries and failures) against its trip, and return per-trip call counts and estimated cost to an authenticated developer through the usage report. | Should | US-03 | T |
| REQ-15 | The system shall provide turn-by-turn guidance with spoken directions from the current location to a destination chosen before driving. | Must | US-01 and US-02 (precondition: an active route); Project_Description.md scope | D |

---

## 6. Non-Functional Requirements

NFR-01 to NFR-09 keep the IDs and priorities the team set in D2 §4.3; each has been restated with a measurable condition. NFR-10 and NFR-11 are new in this version.

| ID | Type | Requirement | Priority | Source | Verification |
| --- | --- | --- | --- | --- | --- |
| NFR-01 | Security (privacy) | The system shall write 0 raw audio files to the phone, backend, logs, or database, and shall use E2 only under a no-retention setting confirmed in writing. | Must | US-04; AC-01.2 | T (storage scan after trip), A (provider terms) |
| NFR-02 | Reliability | The system shall keep the active route and continue guidance after any failed interpretation, planning, authentication, or provider call, in 100% of edge cases EC-01 to EC-34. | Must | US-01, US-02, US-05 | T |
| NFR-03 | Security | The app shall hold no provider API keys; it shall store only a refresh token, in the Android Keystore, and access tokens shall expire after 1 hour. Registration shall be rate-limited per IP address. | Must | US-03 | I, T |
| NFR-04 | Reliability (cost) | The system shall alert developers once at 80% and deny paid calls at 100% of each service's monthly cap, and concurrent calls shall never push spending past the cap. | Must | US-03 | T |
| NFR-05 | Security (logging) | Every server log line shall be JSON with a requestId, and 0 log lines shall contain audio, transcripts, raw GPS coordinates, tokens, or API keys. | Should | US-03 | T (redaction test) |
| NFR-06 | Maintainability | All timeouts, weather thresholds, spending caps, and retention periods shall be configuration values, and the backend shall refuse to start if any value is missing or out of range. | Should | US-02, US-03 | I, T |
| NFR-07 | Usability (distraction) | Every spoken reply shall be one sentence of 15 words or fewer, and no feature shall require the driver to touch the screen while the car is moving. | Must | US-01; CON-03; CON-05 | I, T |
| NFR-08 | Security (retention) | The system shall delete usage events after 90 days and per-trip totals after 12 months. | Should | US-03 | T |
| NFR-09 | Reliability | The microphone and network shall keep working when the app is in the background during a trip, with the foreground trip notification visible for the whole trip. | Should | US-04 | T |
| NFR-10 | Performance | C2 shall return an I2 result within 1.5 seconds of utterance end, and C3 shall return an I4 result within 2.5 seconds of the request, for 95% of requests with 5 concurrent trips. These budgets leave room inside the 2-second and 3-second prompt deadlines in REQ-05 and REQ-06. | Should | US-01, US-05 | T (load test) |
| NFR-11 | Usability (accuracy) | The system shall turn at least 8 of 10 scripted test phrases into the correct constraint, including phrases from accented speakers. | Should | US-05 | T |

---

## 7. MoSCoW Summary

12 of 26 requirements (46%) are Must. The Must set is the smallest version of Veer that is safe to demo in a car: add a stop by voice, never change the route without "okay," keep the route on failure, show and honor mute, keep no audio, keep provider keys off the phone, and stay under budget.

**Must:** REQ-01, REQ-03, REQ-06, REQ-10, REQ-11, REQ-12, REQ-15, NFR-01, NFR-02, NFR-03, NFR-04, NFR-07

**Should:** REQ-02, REQ-04, REQ-05, REQ-07, REQ-08, REQ-13, REQ-14, NFR-05, NFR-06, NFR-08, NFR-09, NFR-10, NFR-11

**Could:** REQ-09

**Won't (this release).** Each item is a decision the team already made; it stays out unless this list is changed and logged.

- Driver accounts, saved trips, or history across devices (D0 §1; D2 R-05)
- A web trip planner (D0 §1)
- An iOS app (D1 §6.1)
- Offline map or routing (D0 D08)
- Sponsored or popularity-based ranking of businesses (D0 D10; CON-06)
- Custom in-dash or vehicle hardware (CON-01)
- Storing raw audio or trip conversations for debugging (D0 D03)
- Storing weather history (D1 §2)
- Any mute path that depends on the cloud (D0 D02)
- Microservice deployment of C2 to C4 (D0 §6)

---

## 8. Inherited Constraints

From Constraints_Essay.md (Week 3). The Technical Specification §5 states what each one rules out or forces.

| ID | Type | Constraint |
| --- | --- | --- |
| CON-01 | Economic | Veer is funded by the team. It uses free-tier hosted inference and Google Maps Platform credits, and runs on the driver's existing phone rather than purpose-built hardware. |
| CON-02 | Security | Raw audio is not retained: each utterance is transcribed, turned into a constraint, and discarded, and only the constraint is kept for the trip. A visible listening indicator and one-tap mute are required before any in-car demo. |
| CON-03 | Legal | Ohio ORC 4511.204 makes handling a phone while driving a primary offense but permits voice-operated use. No feature may ask the driver to tap the screen while moving. |
| CON-04 | Legal | Google Maps Platform terms bar caching most place data beyond 30 days and showing its results over a competing base map. |
| CON-05 | Ethical | Every response competes with the road for attention, so replies are one short sentence; longer content goes to the screen for a passenger or a stop. |
| CON-06 | Ethical | Businesses are ranked only by the driver's stated preference and detour cost. |

---

## 9. Assumptions and Open Questions

**Assumptions** (from UC-01 preconditions and D0/D1):

- A1. The driver chooses a destination before driving (D0 §4 notes).
- A2. The phone has a data connection for interpretation and replanning; guidance continues on the stored route without one (EC-33).
- A3. The phone runs Android and is mounted where the mic can hear the driver (D1 §6.1).
- A4. Hosted inference meets the no-retention requirement; if not, transcription moves to the phone (RK-01 fallback).

**Open questions.** Owners and due dates follow D2 §1.2 and the Week 8 plan.

| ID | Question | Affects | Owner | Due |
| --- | --- | --- | --- | --- |
| G-01 | Can Android recognize "Veer, stop listening" on the phone within 1 second? Fallback: tap-only mute. | REQ-11 | Dominic Rowland (T18) | 2026-11-11 |
| G-02 | Rain threshold, weather freshness window, and maximum weather detour. | REQ-08, NFR-06 | Dominic Rowland (T16) | 2026-11-23 |
| G-03 | Written proof of the OpenAI no-retention setting for audio. | NFR-01 | Briar Elliot (T6) | 2026-11-11 |
| G-04 | Do Google Maps terms allow turn-by-turn guidance in our own app? Fallback: hand off to Google Maps. | REQ-15, CON-04 | Brady Cooper (T11) | 2026-11-11 |
| G-05 | Usage retention. Closed in v1.0: 90 days for events, 12 months for trip totals. | NFR-08 | Aiden Ward | Closed 2026-10-30 |
| G-06 | Does a foreground service keep the mic and network alive in the background? | NFR-09 | Dominic Rowland (T18) | 2026-11-11 |
| G-07 | Monthly spending cap amounts for AI, Maps, and weather. | NFR-04 | Aiden Ward (T25) | 2027-02-25 |
| G-08 | Audio encoding and wire format for I2/I3. | NFR-10 | Briar Elliot (T6) | 2026-11-11 |
| Q-01 | Must set rebalanced in v1.0 (Section 7). Revisit if a Should becomes blocking. | All | Team | 2027-01-11 (start of build) |
| Q-02 | Add a user story for avoiding highways and tolls (REQ-09), which is in the project scope but has no story. | REQ-09 | Brady Cooper | 2026-11-11 |
| Q-03 | Check the NFR-10 latency targets against first OpenAI and Google Maps measurements. | NFR-10 | Briar Elliot, Brady Cooper | 2027-02-12 (M-18) |
| Q-04 | User_Stories.md UC-01 step 6 still lets silence approve a route, which D07 superseded. Update the use case. | REQ-03 | Aiden Ward | 2026-11-11 |
| Q-05 | What happens when C4's 500-event queue fills during a database outage (D2 §3.4)? | NFR-04 | Aiden Ward | 2027-02-15 (T22) |
