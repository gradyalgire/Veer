# Veer Test Plan

**Course:** Computer Science Senior Design I\
**Advisor:** Dr. Jillian Aurisano\
**Team:** Grady Algire, Brady Cooper, Briar Elliot, Dominic Rowland, Aiden Ward\
**Test lead:** Grady Algire\
**Document version:** 1.0 | November 11, 2026\
**Traces to:** [Requirements Document](Requirements_Document.md) v1.0 and [Technical Specification](Technical_Specification.md) v1.0 §6 (traceability matrix)

---

## Version and Change Log

| Version | Date | Change | Author |
| --- | --- | --- | --- |
| 1.0 | 2026-11-11 | First test plan. Uses the TC-01 to TC-32 IDs reserved in the traceability matrix. Where a requirement has more than one case, cases are numbered TC-##.1, TC-##.2, TC-##.3 under the same matrix ID, so no matrix row changes. | Team |

---

## 1. Scope

**In scope.** Every Must and Should requirement in Requirements Document v1.0, across C1 to C4 and the glue components (auth, notifications, configuration, logging). Must requirements are the major requirements and get 3 cases each.

| Component | Owner | Requirements tested |
| --- | --- | --- |
| C1 Mobile Navigation | Grady Algire | REQ-03, REQ-04, REQ-15, NFR-02, NFR-07 |
| C1 mic, mute, notifications | Aiden Ward | REQ-10, REQ-11, REQ-12, REQ-13, NFR-09 |
| C2 Speech Understanding | Brady Cooper | REQ-02, REQ-06, NFR-01, NFR-11 |
| C3 Route Planning (incl. weather) | Briar Elliot, Dominic Rowland | REQ-01, REQ-05, REQ-07, REQ-08, NFR-10 |
| C4 Usage Monitor, auth, logging, config | Aiden Ward | REQ-14, NFR-03, NFR-04, NFR-05, NFR-06, NFR-08 |
| Constraints | Grady Algire, Brady Cooper | CON-01 to CON-06 (inspection) |

**Out of scope.**

| Item | Reason |
| --- | --- |
| REQ-09 avoid highways and tolls (TC-09) | Could priority with no user story yet (Q-02). Deferred to after M-23. |
| All Won't items in Requirements §7 (accounts, web planner, iOS, offline maps, sponsored ranking, custom hardware, weather history, cloud mute, microservices) | Not being built this release. |
| Google Maps route quality, OpenAI transcription internals, Open-Meteo forecast accuracy, Android OS mic and TTS, Render uptime | Third-party parts. We test how Veer handles their output and failures, not the services themselves. |
| EC-16 behavior after the 500-event queue fills | Known limitation, design open (Q-05). We test up to 500 queued events only. |
| Real monthly cap amounts | Not set yet (G-07). Tests use a $10 test cap. |

---

## 2. Verification and Validation Strategy

**Verification** (built right, checked against the spec by the team): every REQ and NFR in scope, through the test cases in §4. **Validation** (right product, checked against stakeholder needs): user stories US-01, US-02, US-04, and US-05 by 3 drivers from outside the team in a parked car at M-20 and on supervised drives at M-23, and US-03 by our advisor reviewing the usage report. Each validator does the story's task with no help and rates it 1 to 5.

| Layer | Approx. count | Coverage focus | Tool |
| --- | --- | --- | --- |
| Unit | 80 | Allow-list check, filter-then-rank, C1 state machine, quota math, config schema, redaction | Jest (backend), JUnit (Kotlin) |
| Integration | 30 | I2, I4, I7 to I9, auth endpoints with E2 to E4 mocked | Supertest, recorded fixtures |
| System | 50 | The cases in §4, run on emulator + staging | Manual + scripts |
| Acceptance | 5 | One task per user story, by outside drivers and advisor | Parked car, supervised drive |

**Speech accuracy (ML).** Variables: speaker (3, at least 2 with accents), phrase (10 scripted), cabin noise (engine on, fan on low). Metric: correct constraint rate. Tolerance: 8 of 10 or better per speaker. Trials: 30 (10 phrases × 3 speakers). **Timing.** Every timing case is run 20 times and all 20 must meet the limit, except NFR-10, which is a p95.

Risk order: mute and audio privacy (REQ-11, REQ-12, NFR-01), approval (REQ-03), and keeping the route on failure (NFR-02) are tested first, then budget and data integrity (NFR-04, REQ-14), then the rest.

---

## 3. Success Criteria

| Area | Metric | Target | How measured | Req ID |
| --- | --- | --- | --- | --- |
| Accuracy | Stop detour | ≤ 600 s | `extraSeconds` in I4 reply | REQ-01 |
| Usability | Words per spoken reply | ≤ 15 | Word count of TTS debug log | REQ-02, NFR-07 |
| Reliability | Routes activated without "okay" | 0 of 20 trials | `tripRevision` before and after | REQ-03 |
| Performance | "okay" to route active | ≤ 3.0 s | Debug log timestamps | REQ-04 |
| Performance | Request to no-match prompt | ≤ 3.0 s | Debug log timestamps | REQ-05 |
| Performance | Speech end to repeat prompt | ≤ 2.0 s | Debug log timestamps | REQ-06 |
| Usability | Mic state visible during trip | 100% of samples | Screenshot every 30 s for 10 min | REQ-10 |
| Performance | Mute to mic stopped | ≤ 1.0 s | `micState` change in debug log | REQ-11 |
| Reliability | Force-stop time; audio after mute | ≤ 2.0 s; 0 frames | Debug log; server frame counter | REQ-12 |
| Security | Raw audio files stored | 0 | File scan of phone, DB dump, logs | NFR-01 |
| Reliability | Route kept on failure | 34 of 34 edge cases | Active `routeId` unchanged | NFR-02 |
| Security | Provider keys in APK | 0 | Decompile and grep | NFR-03 |
| Reliability | Spend over cap | $0.00 | `monthly_budget.used` vs. cap | NFR-04 |
| Security | Log lines with private data | 0 | Grep logs for audio, transcript, GPS, tokens | NFR-05 |
| Reliability | Retention | 90 days / 12 months | Rows left after retention job | NFR-08 |
| Performance | p95 latency, 5 trips | I2 ≤ 1.5 s, I4 ≤ 2.5 s | k6 load test | NFR-10 |
| Accuracy | Correct constraint rate | ≥ 8 of 10 per speaker | Scripted phrase set | NFR-11 |
| Usability | Validation task success | ≥ 4 of 5 tasks, rating ≥ 3 | Outside driver sessions | US-01 to US-05 |

---

## 4. Test Cases

**Setup S1** (used by most cases): staging backend, Pixel 7 emulator (§5), mock GPS route R1 Cincinnati (39.1031, -84.5120) to Columbus (39.9612, -82.9988), destination entered, trip started, mic listening. Clips A-01 to A-14 are played into the emulator mic (§5). "Debug log" means the C1 debug-build event log.

### C3 Route Planning

| ID | Description | Steps | Expected Result | Status |
| --- | --- | --- | --- | --- |
| TC-01.1 | Add a named stop (REQ-01) | 1. S1. 2. Play A-01 "I'm hungry for McDonald's." 3. Read I4 reply in debug log. | Proposal with `selectedPlace` named McDonald's and `extraSeconds` ≤ 600. | Not Run |
| TC-01.2 | Add a category stop (REQ-01) | 1. S1. 2. Play A-02 "find a gas station." 3. Read I4 reply. | Place is a gas station; `extraSeconds` ≤ 600. | Not Run |
| TC-01.3 | Edge: only match is too far (REQ-01, EC-09) | 1. S1 with Maps fixture F-FAR (nearest McDonald's adds 900 s). 2. Play A-01. | `no_match`; no proposal; `routeId` unchanged; prompt for another place. | Not Run |
| TC-05 | No match prompt time (REQ-05) | 1. S1 with F-FAR. 2. Play A-01, 20 times. | Prompt starts ≤ 3.0 s after request in 20 of 20. | Not Run |
| TC-07 | "No" gives next candidate (REQ-07) | 1. TC-01.1 steps 1-2. 2. Play A-04 "no." | A different place is proposed in one sentence. | Not Run |
| TC-08 | Avoid rain, and warn on stale data (REQ-08) | 1. S1 with weather fixture W-RAIN. 2. Play A-05 "avoid the rain." 3. Repeat with W-STALE. | W-RAIN: chosen route has lowest rain exposure. W-STALE: warning spoken; no claim of avoiding rain. | Not Run |
| TC-25 | Latency under load (NFR-10) | 1. Run `k6 run load/veer.js` against staging: 5 virtual trips, 100 I2 and 100 I4 requests each. | p95 I2 ≤ 1.5 s; p95 I4 ≤ 2.5 s. | Not Run |

### C1 Mobile Navigation

| ID | Description | Steps | Expected Result | Status |
| --- | --- | --- | --- | --- |
| TC-03.1 | "okay" activates route (REQ-03) | 1. TC-01.1 steps 1-2. 2. Play A-03 "okay." | `tripRevision` +1; new route on map. | Not Run |
| TC-03.2 | Edge: silence never approves (REQ-03) | 1. TC-01.1 steps 1-2. 2. Wait 30 s silent. 3. Repeat 20 times. | `tripRevision` and `routeId` unchanged in 20 of 20. | Not Run |
| TC-03.3 | Failure: unclear reply (REQ-03) | 1. TC-01.1 steps 1-2. 2. Play A-06 "um, maybe." 3. Repeat 20 times. | Route unchanged in 20 of 20; driver asked again. | Not Run |
| TC-04 | Activation time (REQ-04) | 1. TC-03.1, 20 times. | Route active ≤ 3.0 s after "okay" ends in 20 of 20. | Not Run |
| TC-15.1 | Spoken turn-by-turn (REQ-15) | 1. S1. 2. Play GPS track R1.gpx at 1x. | Every `RouteStep` instruction is spoken before its turn. | Not Run |
| TC-15.2 | Edge: no destination (REQ-15, EC-03) | 1. Open app, skip destination. 2. Tap Start. | Trip does not start; app asks for a destination. | Not Run |
| TC-15.3 | Failure: GPS and network lost (REQ-15, EC-33, EC-34) | 1. S1 and play R1.gpx. 2. Set network to offline. 3. Pause GPS for 60 s. | Guidance continues; "No connection" and "GPS signal lost" each said once. | Not Run |
| TC-17.1 | Keep route on C2 failure (NFR-02, EC-06, EC-08) | 1. S1 with E2 mock set to timeout, then to 503. 2. Play A-01 for each. | `routeId` unchanged; guidance continues. | Not Run |
| TC-17.2 | Keep route on C3 failure (NFR-02, EC-10 to EC-12) | 1. S1 with E3 mock set to timeout, then 503, then stale `tripRevision`. 2. Play A-01 for each. | `routeId` unchanged; stale reply ignored. | Not Run |
| TC-17.3 | Full edge-case suite (NFR-02, EC-01 to EC-34) | 1. Run `npm run test:edge` on staging plus the C1 cases in `edge/README.md`. | 34 of 34 pass, each logged by EC ID. | Not Run |
| TC-22.1 | Reply length (NFR-07) | 1. Collect TTS debug log from all §4 runs. 2. Count words per reply. | Every reply ≤ 15 words. | Not Run |
| TC-22.2 | Edge: long confirmation (NFR-07, EC-05) | 1. S1 with C3 mock returning a 25-word `confirmationText`. 2. Play A-01. | Spoken reply is a place-and-minutes summary ≤ 15 words. | Not Run |
| TC-22.3 | No taps while moving (NFR-07) | 1. S1 with R1.gpx at 1x (speed > 0). 2. Do each in-trip action by voice only. | Every in-trip action works by voice; none needs a tap. | Not Run |

### C1 Mic and Mute

| ID | Description | Steps | Expected Result | Status |
| --- | --- | --- | --- | --- |
| TC-10.1 | State shown while listening (REQ-10) | 1. S1. 2. Screenshot screen and notification shade every 30 s for 10 min. | All 20 screenshots show "Listening." | Not Run |
| TC-10.2 | State shown when muted (REQ-10) | 1. S1. 2. Tap mute. 3. Check screen and notification. | Both show "Muted." | Not Run |
| TC-10.3 | Failure: error state (REQ-10) | 1. S1 with debug flag `mic_ignore_stop`. 2. Tap mute. | Both show "Error" within 2 s. | Not Run |
| TC-11.1 | Tap mute time (REQ-11) | 1. S1. 2. Tap mute. 3. Read time from tap to `micState=muted` in debug log. 4. Repeat 20 times. | ≤ 1.0 s in 20 of 20. | Not Run |
| TC-11.2 | Voice mute time (REQ-11) | 1. S1. 2. Play A-07 "Veer, stop listening." 3. Read time from phrase end to muted. 4. Repeat 20 times. | ≤ 1.0 s in 20 of 20. | Not Run |
| TC-11.3 | Edge: mute with no network (REQ-11) | 1. S1. 2. Set network offline. 3. Do TC-11.1 steps 2-4. | ≤ 1.0 s in 20 of 20. | Not Run |
| TC-12.1 | Failure: mic will not stop (REQ-12, EC-04) | 1. S1 with `mic_ignore_stop`. 2. Tap mute. 3. Repeat 20 times. | Mic force-stopped ≤ 2.0 s in 20 of 20; "Error" shown. | Not Run |
| TC-12.2 | No audio after mute (REQ-12) | 1. TC-12.1 steps 1-2 while playing A-01. 2. Read server frame counter for the trip. | 0 frames received after mute time; phone buffer empty. | Not Run |
| TC-12.3 | Edge: mute mid-request (REQ-12) | 1. S1. 2. Play A-01 and tap mute 0.5 s in. | In-flight I2 request cancelled; no proposal arrives. | Not Run |
| TC-13 | Unmute by tap only (REQ-13) | 1. S1, tap mute. 2. Play A-08 "Veer, start listening." 3. Tap unmute. | Step 2: still muted. Step 3: "Listening." | Not Run |
| TC-24 | Background mic and network (NFR-09) | 1. S1. 2. Press Home, wait 10 min. 3. Play A-01. | Notification visible the whole time; proposal arrives. | Not Run |

### C2 Speech Understanding

| ID | Description | Steps | Expected Result | Status |
| --- | --- | --- | --- | --- |
| TC-02 | Confirmation before activation (REQ-02) | 1. TC-01.1 steps 1-2. 2. Read TTS debug log. | Confirmation ≤ 15 words is spoken before approval is possible. | Not Run |
| TC-06.1 | Request outside allow-list (REQ-06) | 1. S1. 2. Play A-09 "play some jazz." 3. Repeat 20 times. | `unsupported_constraint`; route unchanged; repeat prompt ≤ 2.0 s in 20 of 20. | Not Run |
| TC-06.2 | Edge: noise only (REQ-06, EC-07) | 1. S1. 2. Play A-10 (5 s road noise). | `unclear_speech`; route unchanged; prompt ≤ 2.0 s. | Not Run |
| TC-06.3 | Failure: corrupt audio (REQ-06, EC-07) | 1. Run `scripts/send_corrupt_audio.ts` against staging I2. | `invalid_audio` (400); C1 prompts ≤ 2.0 s. | Not Run |
| TC-16.1 | No audio on phone (NFR-01) | 1. S1, play A-01 to A-05. 2. Force-stop app mid-utterance. 3. `adb shell find /data /sdcard -name "*.pcm" -o -name "*.wav" -o -name "*.m4a"`. | 0 audio files. | Not Run |
| TC-16.2 | No audio on server (NFR-01) | 1. After TC-16.1, dump staging DB and logs. 2. Grep for audio bytes and transcript text. | 0 matches. | Not Run |
| TC-16.3 | Provider does not keep audio (NFR-01) | 1. Open the G-03 written proof in the repo. 2. Check the staging E2 setting matches it. | Proof exists and setting is no-retention. | Not Run |
| TC-26 | Speech accuracy (NFR-11) | 1. Play clip set P-01 to P-10 for each of 3 speakers (S1, cabin noise). 2. Compare constraint to answer key. | ≥ 8 of 10 correct for each speaker. | Not Run |

### C4, Auth, Logging, Config

| ID | Description | Steps | Expected Result | Status |
| --- | --- | --- | --- | --- |
| TC-14 | Usage report (REQ-14) | 1. Run a trip with 3 AI and 2 Maps calls (1 failed). 2. `GET /api/v1/usage/report` with admin key. 3. Repeat with no key. | Step 2: trip shows 3 AI, 2 Maps, cost > 0. Step 3: 401. | Not Run |
| TC-18.1 | No keys in app (NFR-03) | 1. `apktool d veer.apk`. 2. Grep for `AIza` and `sk-`. | 0 matches. | Not Run |
| TC-18.2 | Token storage and expiry (NFR-03) | 1. Register. 2. Check shared prefs has no token. 3. Set server clock +61 min; call I4. 4. Refresh. | Token only in Keystore; step 3 gets 401; step 4 works. | Not Run |
| TC-18.3 | Edge: register rate limit (NFR-03, EC-17) | 1. Send config limit + 1 register calls from one IP in 1 min. | Last call gets 429 `rate_limited`. | Not Run |
| TC-19.1 | Alert at 80% (NFR-04) | 1. Set AI test cap $10. 2. Spend to $8.00, then $8.50. | 1 alert at $8.00; no second alert (EC-21). | Not Run |
| TC-19.2 | Deny at 100% (NFR-04, EC-13) | 1. Spend to $10.00. 2. Make 1 more AI call. | 429 `quota`; route unchanged. | Not Run |
| TC-19.3 | Edge: concurrent spend (NFR-04, EC-14) | 1. Set used to $9.90. 2. Fire 20 concurrent $0.05 calls. | ≤ 2 allowed; `used` ≤ $10.00. | Not Run |
| TC-20 | Log redaction (NFR-05) | 1. After all §4 runs, grep staging logs for transcripts, lat/lon, tokens, keys. 2. Parse every line as JSON. | 0 matches; every line is JSON with `requestId`. | Not Run |
| TC-21 | Bad config stops startup (NFR-06, EC-27) | 1. Delete `timeout.planSec` from config. 2. Start backend. | Startup refused; `config_invalid` logged. | Not Run |
| TC-23 | Retention (NFR-08) | 1. Seed events aged 89 and 91 days and totals aged 11 and 13 months. 2. Run retention job. | Only the 91-day events and 13-month totals are deleted. | Not Run |

### Constraints

| ID | Description | Steps | Expected Result | Status |
| --- | --- | --- | --- | --- |
| TC-27 | Team-funded (CON-01) | 1. Check monthly bills for Render, OpenAI, Google, Open-Meteo. | Each is within its cap; app runs on a stock phone. | Not Run |
| TC-28 | No audio; indicator and mute (CON-02) | 1. Confirm TC-10, TC-11, TC-16 passed. | All pass. | Not Run |
| TC-29 | Voice-only while moving (CON-03) | 1. Confirm TC-22.3 passed. | Pass. | Not Run |
| TC-30 | Google Maps terms (CON-04) | 1. Check DB schema for place data. 2. Check the base map. | No place table; Google base map only. | Not Run |
| TC-31 | Short replies (CON-05) | 1. Confirm TC-22.1 and TC-22.2 passed. | Both pass. | Not Run |
| TC-32 | Fair ranking (CON-06) | 1. S1 with fixture F-SPONSOR (one place marked sponsored, longer detour). 2. Play A-01. | Shorter-detour place chosen; sponsored flag ignored. | Not Run |

---

## 5. Test Environment

| Environment | Used for | Setup |
| --- | --- | --- |
| Development | Unit, integration | Laptop; Node.js 20 LTS; PostgreSQL 16 in Docker; Android Studio with Pixel 7 emulator, Android 14 (API 34); E2 to E4 mocked with recorded fixtures. GitHub Actions runs the same on every push. |
| Staging | System, load | Render web service + managed PostgreSQL 16, separate from production; real E2 to E4 keys with the $10 test caps; Pixel 7 emulator as above. |
| Pilot | Acceptance, validation | Team Android phone (Android 14 or later), car mount, team member's car; production build on Render. Parked car at M-20, supervised drives with a non-driving tester at M-23. |

**Test data** (in `test/fixtures/`): audio clips A-01 to A-14 and accuracy set P-01 to P-10 from 3 speakers (16 kHz mono WAV); GPS track R1.gpx; Maps fixtures F-NEAR, F-FAR, F-SPONSOR; weather fixtures W-RAIN, W-STALE; DB seed script `seed_test.sql` (one installation, budget rows at 0%, 79%, 99%). **Network:** emulator set to LTE for normal runs and to offline for TC-11.3 and TC-15.3. **Config:** `config/test.json`, checked in.

---

## 6. Pass and Fail Criteria

**Per test case.** *Pass:* every expected result in §4 is met in every trial, at the §3 threshold. *Fail:* any trial misses any threshold, or the active route changes when it should not. *Blocked:* the case cannot run because of an environment or dependency problem (logged with the reason). *Not Run:* not yet tried.

**Defect severity.** Sev 1: privacy or safety (audio kept, mute fails, route changes without "okay"). Sev 2: a Must case fails. Sev 3: a Should case fails. Sev 4: cosmetic.

**Whole plan.** *Pass:* 100% of Must cases pass, at least 90% of Should cases pass, 0 open Sev 1 or Sev 2 defects, and validation meets its §3 target. *Fail:* any of these is missed at M-23.

---

## 7. Schedule

Milestones are from the Week 8 plan.

| Milestone | Test activity | Owner | Evidence |
| --- | --- | --- | --- |
| M-10 Nov 11 | This plan submitted | Grady Algire | Test_Plan.md in repo |
| M-15 Dec 2 | Unit tests and CI for config and C4 tables (TC-21, TC-23) | Aiden Ward | Green GitHub Actions run |
| M-17 Feb 1 | Unit + integration: auth and logging (TC-18, TC-20) | Aiden Ward | CI run; test log |
| M-18 Feb 12 | Integration: C2 (TC-02, TC-06, TC-16); first TC-26 run | Brady Cooper | CI run; accuracy sheet |
| M-19 Feb 15 | Integration: C3 (TC-01, TC-05, TC-07, TC-08) | Briar Elliot, Dominic Rowland | CI run; test log |
| M-20 Mar 3 | System in parked car: mute, approval, guidance (TC-03, TC-04, TC-10 to TC-13, TC-15, TC-22, TC-24); first validation sessions | Aiden Ward, Grady Algire | Test log; screen recordings; validation notes |
| M-21 Mar 15 | System: full edge-case suite and load (TC-17, TC-25, TC-26) | Each component owner | Edge-case report; k6 report |
| M-22 Mar 16 | System: usage and caps (TC-14, TC-19) | Aiden Ward | Test log; alert email |
| M-23 Apr 2 | Acceptance: supervised drives and advisor review; constraints (TC-27 to TC-32) | Grady Algire | Drive log; validation results; defect list |
| M-24 Apr 8 | Full regression on the v1.0 tag | Grady Algire | Test results table |

---

## 8. Definition of Done

**Feature done** when:

- [ ] Unit tests written and passing in CI
- [ ] Its §4 cases pass, including the edge case
- [ ] Code reviewed by one other team member
- [ ] Docs and config updated (Tech Spec, README if needed)
- [ ] No open Sev 1 or Sev 2 defects on it
- [ ] Component owner signs off in the PR

**Project done** when:

- [ ] All §4 cases run; whole-plan pass criteria in §6 met
- [ ] Traceability matrix Status column updated for every row
- [ ] All docs match the v1.0 build
- [ ] 0 open Sev 1 or Sev 2 defects; Sev 3 and 4 listed in the final report
- [ ] Team and advisor sign off at M-23
