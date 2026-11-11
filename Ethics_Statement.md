# Veer Ethics Statement
 
**Course:** Computer Science Senior Design I\
**Advisor:** Dr. Jillian Aurisano\
**Team:** Grady Algire, Brady Cooper, Briar Elliot, Dominic Rowland, Aiden Ward\
**Documentation lead:** Grady Algire\
**Document version:** 1.0 | November 10, 2026\
**Traces to:** [Technical Specification](Requirements/Technical_Specification.md) v1.0 (schema in §3.4, stack in §4), [Constraints Essay](Constraints_Essay.md)
 
Veer listens to the driver while a trip is on, so the microphone is the most sensitive part of the system. This statement says how we protect that data and where our design is still open.
 
---
 
## 1. Privacy summary
 
Every row below is a data item that crosses an arrow on the D0 data flow diagram or in the Technical Specification.
 
| Data item | Source | Purpose | Consent | Storage | Access | Retention |
| --- | --- | --- | --- | --- | --- | --- |
| Raw mic audio (PCM frames, I1a to I2) | Driver's phone mic (C1) | Turn speech into a constraint | Driver: mic permission and first-run notice (planned). Passengers: none, see below | Phone memory only, 15 s cap | C1 app; sent to C2 over TLS | Discarded after each step. Never written to disk, logs, or the database (NFR-01) |
| Approval audio ("okay", I2) | Driver's phone mic | Confirm the proposed route | Same as raw audio | Phone memory only | C1; sent to C2 over TLS | Discarded after the reply |
| Audio sent to OpenAI (I3) | C2 | Transcribe and read intent | Written no-retention setting from the provider (G-03) | Not stored by Veer | OpenAI, under our API key | Provider keeps no audio. Written proof is still open, due Nov 11, 2026 |
| Transcript and intent (I2, I3) | OpenAI reply via C2 | Check against the allow-list | Same as raw audio | Memory during the step | C2 | Cleared at trip end |
| Constraint (type, place name, max detour) | Driver's speech via C2 | Change the route | Driver asked for it by voice | Memory for the trip | C1, C3 | Cleared at trip end |
| Driver location (lat, lon, accuracy, time) | Phone GPS (I1b) | Guidance and ranking |  location permission | Phone memory; sent in each route request | C1, C3; origin sent to Google Maps and Open-Meteo | Not kept in our database. Cleared at trip end |
| Destination and place results (I5) | Driver's entry; Google Maps | Routing | Driver picks destination before driving | Phone memory; provider place data cached 30 days at most (CON-04) | C1, C3 | Trip end; place cache 30 days at most |
| Weather request (location, time, I6) | C3 | Rain avoidance | Covered by location permission | Not stored by us | Open-Meteo | Open-Meteo's own retention not yet checked |
| Route proposal and warnings (I4 reply) | C3 | Show and read the route | Derived from the above | Phone memory | C1 | Trip end |
| Usage events (trip ID, service, operation, units, outcome, time, est. cost; I7, I8) | C2, C3 | Keep paid API use under caps | No personal data. No audio, transcript, or location | PostgreSQL | Developers, via admin key (I9) | 90 days per event; 12 months per trip total (NFR-08) |
| Install record (install ID, created time, status, refresh-token hash) | C1 at register | Anonymous login | No name, email, or account | PostgreSQL. Raw token only in phone Keystore | Server only | No retention rule yet. Team to set |
| Server logs (requestId, status, timing) | Logging glue | Debugging | Not personal. Redaction filter (NFR-05) | Render logs | Team developers | 30 days (Spec §4) |
| Mute event (tap or "Veer, stop listening") | Driver | Stop the mic | Driver's own action | Phone only. No network call | C1 | Not kept |
 
**Consent.** The driver agrees on first run. The app will ask for mic and location permission, then show a short screen. The screen says what Veer hears, that audio is not kept, and that the mic can be muted at any time. The driver taps to accept. This screen is planned and not yet built.
 
**Passengers.** Passengers have not agreed to anything. We do not claim their consent. A visible listening indicator and a one-tap mute are the only protection, and both are required before any in-car demo (CON-02).
 
**Fields removed.** None recorded yet. Any field the team drops because it cannot be justified will be listed here.
 
**When the project ends.** After the final demo, the team will delete the Render service and its PostgreSQL database, revoke the OpenAI and Google keys, and remove the environment variables that hold them.
 
---
 
## 2. Security summary
 
The repo has no application code yet. The mechanisms below come from the Technical Specification and are not tested. Test TC-18.1 (no keys in the app) is marked Not Run.
 
**Default credentials.** No default password or admin account is created. The only admin key is for the usage report (I9). It is set in an environment variable, and the server should refuse to start without it (NFR-06).
 
**Unauthenticated endpoints.** Every endpoint needs an access token except register and refresh. Register is rate-limited per IP and returns 429 when it is hit too often. Access tokens last 3600 seconds (1 hour). Refresh tokens are stored only as hashes in PostgreSQL. The raw token stays in  Keystore. A revoked install gets 403 `token_revoked`.
 
**Secrets in source control.** OpenAI and Google keys live only in Render environment variables. No key goes into the  app. We searched the full git history (all 27 commits, all branches, full clone) for:
 
- OpenAI, Google, and AWS key patterns
- private key blocks
- `password`, `api_key`, `secret`, or `token` assignments
- database URLs with passwords
We also searched every filename ever committed for `.env`, `key`, `secret`, `credential`, `keystore`, `local.properties`, and `config`. **Result: no matches.** The only text hit was a line in `Test_Plan.md` that lists the patterns to search for. Nothing needed fixing.
 
**HTTPS and limits.** All traffic uses TLS (HTTPS and WSS). Each trip has spending caps through C4, and register is rate-limited.
 
**Self-audit questions**
 
| Question | Answer |
| --- | --- |
| Does the app contain any API key? | No, by design. TC-18.1 not yet run |
| Can any endpoint except register and refresh be called without a token? | No, by design |
| Is the refresh token stored in plain text on the server? | No. Only its hash |
| Is there a default username or password? | No. No accounts exist |
| Was any secret found in git history? | No |
| Is raw audio written to disk, logs, or the database? | No, by design. Storage scan (TC-16) is planned |
| Can the usage report be read without the admin key? | No. Returns 401 |
 
---
 
## 3. Licensing summary
 
**Our license: MIT.** It lets anyone reuse the code if they keep the license notice. It works with every dependency below.
 
The repo has no package or build files yet, so this list comes from Technical Specification §4. Versions get pinned when code is added.
 
| Name | Version | License (SPDX) | Permissive or copyleft | Obligation on us |
| --- | --- | --- | --- | --- |
| Node.js | Not pinned | MIT | Permissive | Keep license notice if redistributed |
| Express | Not pinned | MIT | Permissive | Keep license notice if redistributed |
| TypeScript | Not pinned | Apache-2.0 | Permissive | Keep notices; include NOTICE if one ships |
| Kotlin | Not pinned | Apache-2.0 | Permissive | Keep notices |
| PostgreSQL (via Render) | Managed | PostgreSQL | Permissive | None for hosted use |
|  SDK | Not pinned | Not SPDX.  SDK terms | Proprietary | Accept Google's terms; do not redistribute the SDK |
| Google Maps Platform | API | Not SPDX. Google Maps Platform terms | Proprietary | Use our key; cache place data 30 days at most; show Google attribution; no competing base map (CON-04) |
| OpenAI API | API | Not SPDX. OpenAI terms | Proprietary | Use no-retention setting (G-03); pay per use |
| Open-Meteo weather data | API | CC-BY-4.0 | Permissive, with attribution | Credit "Weather data by Open-Meteo.com" with a link |
| Open-Meteo code | Not used | AGPL-3.0-or-later | Copyleft | None while we use only the hosted API |
 
**Flags and decisions**
 
- **No copyleft code in our stack.** We do not copy or self-host Open-Meteo's code. If we ever self-host and change it, AGPL would require us to share those changes. **Decision: use the hosted API only.**
- **Open-Meteo is non-commercial only on the free tier.** Its terms limit the free API to non-commercial use. Veer is a school project, so this is fine. Any paid or commercial use needs a subscription. **Decision: keep the free tier for development and the course demo.**
- **Google and OpenAI are not open source.** Their terms set the limits we design around (keys on the server, 30-day cache, no-retention setting).
**Compatibility.** Every row fits an MIT project. Apache-2.0, PostgreSQL, and CC-BY-4.0 data are all compatible when attribution is given.
 
---
 
## 4. Attribution list
 
**Libraries**
 
- Node.js, MIT: https://github.com/nodejs/node
- Express, MIT: https://github.com/expressjs/express
- TypeScript, Apache-2.0: https://github.com/microsoft/TypeScript
- Kotlin, Apache-2.0: https://github.com/JetBrains/kotlin
- PostgreSQL, PostgreSQL License: https://www.postgresql.org/about/licence/
**Datasets**
 
- Open-Meteo weather data, CC-BY-4.0. Collection: Open-Meteo gathers open data from national weather services. Restriction: free tier is non-commercial. Credit: "Weather data by Open-Meteo.com" with a link to https://open-meteo.com.
- Google Maps Platform data. Proprietary. Show Google's required attribution on any map or place result.
**Adapted code.** None yet. Any snippet copied from a tutorial or another repo will be named here with its source URL.
 
**Non-code assets.** None yet (no icons, fonts, or images). Each one will be listed with its author and license.
 
The README links to this section.
 
---
 
## 5. AI-use disclosure
 
Syllabus policy: "Generative AI tools may be used for drafting and code assistance but not for the design decisions or written analysis that are assessed in this course."
 
| Tool | Used by | Task | Column | How verified |
| --- | --- | --- | --- | --- |
| Claude | Grady Algire | Ran git history secret search and license lookup | Generally OK | Commit count (27) and search output checked by hand |
| ChatGPT | Brady Cooper | Proofread grammar and wording in team-written docs | Generally OK | Read every edit; meaning unchanged |
| ChatGPT | Briar Elliot | Formatted Markdown tables for the Technical Specification | Generally OK | Table contents compared to the original notes |
| ChatGPT | Dominic Rowland | Drafted README template headings | Generally OK | Edited by hand to match the repo |
| Claude | Aiden Ward | Summarized official API docs for team reference | Generally OK | Summary checked against the official docs |
 
**Confirmation.** Each member confirms that the architecture, data model, priorities, and design rationale in this statement are the team's own.
 
- Grady Algire:      Grady Algire       Date: 11/11/2026
- Brady Cooper:      Brady Cooper       Date: 11/11/2026
- Briar Elliot:      Briar Elliot       Date: 11/11/2026
- Dominic Rowland:   Dominic Rowland    Date: 11/11/2026
- Aiden Ward:        Aiden Ward         Date: 11/11/2026
