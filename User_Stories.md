# Veer User Stories and Use Cases

**Team:** Grady Algire, Brady Cooper, Briar Elliot, Dominic Rowland, Aiden Ward

**Advisor:** Dr. Jillian Aurisano

---

## Stakeholders

| Category | Stakeholder | Why they matter |
| --- | --- | --- |
| Primary | Driver using a phone mounted in their own car | Talks to Veer and follows the route |
| Secondary | Veer dev team (deploys and supports the app) | Keeps the app running on free-tier AI and Google Maps credits |
| Secondary | Google Maps Platform (downstream system) | Supplies routes and places; its terms limit caching and map display |
| Hidden | Passengers in the car | Are recorded by the microphone |
| Hidden | Drivers with accents or speech differences (accessibility) | Speech-to-text may misunderstand them |
| Hidden | Ohio law / ORC 4511.204 (compliance) | Bans handling a phone while driving |
| Hidden | Businesses shown as stops | Veer picks which business a driver visits |

---

## User Stories

**US-01 (Driver):**
As a driver on a road trip,
I want to add a food stop by saying what I'm hungry for,
so that I can eat without leaving my route by more than a few minutes.

**US-02 (Driver):**
As a commuter driving home,
I want to tell Veer to avoid rain,
so that I spend less time driving in bad weather.

**US-03 (Dev team):**
As a Veer developer supporting the app,
I want to see how many AI and Maps API calls each trip uses,
so that we stay within our financial limits.

**US-04 (Passenger):**
As a passenger riding in a Veer driver's car,
I want to know when the app is listening and be able to stop it,
so that my private conversations are not recorded.

**US-05 (Accessibility):**
As a driver with a strong accent,
I want Veer to confirm what it heard before changing my route,
so that a misheard request doesn't send me the wrong way.

---

## INVEST Self-Check

| Story | I | N | V | E | S | T |
| --- | --- | --- | --- | --- | --- | --- |
| US-01 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| US-02 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| US-03 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| US-04 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| US-05 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

---

## Use Cases

### UC-01: Add a Food Stop by Voice
**Expands:** US-01

**Primary actor:** Driver

**Secondary actors:** Google Maps Platform, AI model (speech-to-text and constraint parser)

**Preconditions**
1. Veer is running with an active route.
2. The car is moving.
3. The listening indicator is on and the mic is not muted.
4. The phone has a data connection.

**Main Success Flow**
1. Driver says "I'm hungry for McDonald's."
2. System transcribes the speech and turns it into `add_waypoint: McDonald's`.
3. System searches for McDonald's locations near the route.
4. System picks the one with the lowest detour time.
5. System says a one-sentence confirmation, e.g. "Adding McDonald's, 4 minutes extra."
6. Driver says nothing or "okay."
7. System inserts the stop and updates turn-by-turn directions.
8. System deletes the raw audio.

**Alternate Flow - Driver rejects the stop:**
Driver says "no" or "not that one."
System offers the next-best location in one sentence.
Return to step 6.

**Exception Flow - No match within detour limit:**
No McDonald's adds 10 minutes or less.
System says "No McDonald's close by. Want another place?"
Route stays the same.

**Exception Flow - Speech not understood:**
System can't turn the speech into a constraint.
System says "Sorry, say that again?"
Route stays the same.

**Postcondition:**
The route includes the chosen stop (or is unchanged after an exception), and no raw audio is stored.

---

### UC-02: Mute Listening
**Expands:** US-04

**Primary actor:** Passenger (or driver when stopped)

**Secondary actors:** None

**Preconditions**
1. Veer is running and the listening indicator is on.

**Main Success Flow**
1. Passenger taps mute or says "Veer, stop listening."
2. System stops the mic stream.
3. System changes the indicator to muted.
4. System says "Listening off."

**Alternate Flow - Unmute:**
Mic is already muted; passenger taps unmute (voice can't unmute since the mic is off).
System turns the mic back on and shows the listening indicator.

**Exception Flow - Mic fails to stop:**
The mic stream is still active after mute.
System kills the audio process and shows an error.
System drops any audio captured after the mute request.

**Postcondition:**
No audio is captured or sent while muted.

---

## Acceptance Criteria

**UC-01**

AC-01.1 (Main flow):
Given an active route and an unmuted mic,
When the driver says "I'm hungry for McDonald's,"
Then Veer adds a McDonald's stop with a detour of 10 minutes or less and speaks a confirmation of 15 words or fewer within 3 seconds.

AC-01.2 (Main flow):
Given a stop was just added,
When the trip data is checked,
Then 0 raw audio files are stored on the phone or any server.

AC-01.3 (Exception E1):
Given no McDonald's adds 10 minutes or less,
When the driver asks for McDonald's,
Then the route does not change and Veer asks for a different place within 3 seconds.

AC-01.4 (Exception E2):
Given the speech can't be parsed into a constraint,
When the driver finishes speaking,
Then the route does not change and Veer asks the driver to repeat within 2 seconds.

**UC-02**

AC-02.1 (Main flow):
Given the listening indicator is on,
When a passenger says "Veer, stop listening,"
Then the mic stops within 1 second and the indicator shows muted.

AC-02.2 (Exception E1):
Given mute was requested,
When the mic stream does not stop,
Then Veer force-stops it within 2 seconds and 0 seconds of audio after the request are sent or stored.