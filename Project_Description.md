# Veer Project Description

**Course:** Computer Science Senior Design I
**Advisor:** Dr. Jillian Aurisano
**Team:** Grady Algire, Brady Cooper, Briar Elliot, Dominic Rowland, Aiden Ward
**Version:** 1.0

---

## Overview

Veer is a navigation service that uses AI to determine a driver's route. Instead of typing a destination and accepting whatever path the app returns, the driver talks to Veer while driving. The system interprets what it hears as routing constraints and preferences, then plans or re-plans the route around them.

A driver who says "I'm hungry for McDonald's" gets a route with a McDonald's stop inserted at a low detour cost. A driver who says "I don't want to drive in the rain" gets a route weighted away from active precipitation. The same interface stays open to additional context sources and features as the project grows.

## The problem

Existing navigation apps optimize for a single objective — usually time — and expect the driver to express everything else by hand. Adding a stop, avoiding a highway, or changing a preference mid-trip means touching the screen, which is both unsafe and illegal to do while driving in Ohio. Real driving decisions are messy and stated in plain language, and no mainstream app accepts them that way.

## What Veer does

Veer converts natural speech into structured routing constraints and applies them to a live route.

1. **Listen.** The app captures speech while the vehicle is in motion.
2. **Interpret.** An utterance is transcribed and converted into a structured constraint (`add_waypoint: McDonald's`, `avoid: precipitation`, `avoid: highways`).
3. **Re-plan.** Active constraints are applied to the route and the driver hears a one-sentence confirmation.
4. **Discard.** Raw audio is not retained. Constraints are held only for the length of the trip.

## Users

The primary user is a driver using a phone mounted in their own vehicle. Passengers are present in the cabin but are not users of the app and have not agreed to anything, which is why a visible listening indicator and a one-tap mute are requirements rather than nice-to-haves.

## Scope

**In scope for the first release**

- Voice capture with an explicit listening indicator and mute
- Transcription and constraint extraction from natural language
- Waypoint insertion by business or category ("find a McDonald's on the way")
- Route preference constraints (avoid highways, avoid tolls, avoid weather)
- Turn-by-turn routing and voice guidance
- Mobile application on the driver's existing phone
