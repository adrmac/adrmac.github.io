---
title: "Building out an interactive spatial audio installation for the Electric Sky festival"
date: 2026-05-11
---

Today is Monday. We had a fairly successful jam hangout session on Saturday. I demoed the wireless raspberry pi distance sensor sending MIDI into both Ableton and VCV. We were able to drive several interesting parameters, including reverb/delay, BPM, and quad panning. 

At a different point in the session, we heard Evan play some interesting polyrhythms from the Oxi One, while he and Tom played modular synths. The result was this kind of energized tonal cloud that sounded good. Tom is new to our group and said he appreciated the opportunity. 

Chris and I were both working solely from Ableton Live workstations, so had some challenges in trying to contribute to that jam. A couple of approaches came to mind:

1. jamming on my own at home, recording highlight clips to ableton, then pitch shifting in the session to match the tonality of other musicians, adjusting bpm
2. incorporating the DJtoGo2 Touch as a session instrument for playing clips and layering variations
3. syncing all inputs to a common MIDI clock

In discussing the Pi, we also understood a couple of things:

1. We want the Pi to serve as an access point for a wifi network for local devices, but it does not need to be connected to the internet. We do not ever want to rely on venue wifi. If the Pi is not reliable or powerful enough for this, perhaps we could source a "travel router" -- either way we only care about connecting devices that are physically in the same space.

2. Connecting a computer owned by a contributor with no previous experience running Python is a potential blocker. We do not want to require computers receiving OSC and/or MIDI to get hung up on python versioning, dependencies, or ideosyncratic local configs. We want all contributing computers to get the signal natively, through a browser ui, or through an app download (last choice).

Finally, for the distance sensor > MIDI idea, there are some conceptual knots to untangle. 

1. In the "spatial audio" topology of our installation piece, what is the relationship between: 
  - a 5.1 multi-channel surround-sound system with quad panning effects;
  - an active sensor field that monitors and responds to occupants' spatial positioning and activity levels? 
  - Given all of our capabilities and interests, what experiments should we try?

2. How does the passive / active listener dynamic compare with the digital / analog contrast we are expressing visually in Processing/Touch Designer by combining MIDI/OSC data with raw microphone outputs? are they related or totally different axes? 
- MIDI distance sensor > crisp, immediate response
- audio mix > fuzzy, indirect

3. The MIDI distance sensor feels like a novelty toy at this point. It gets a satisfying response but its usefulness in a musical composition or performance is still sketchy beyond something to fidget with. Perhaps it is too one-dimensional? If we have 6 sensors in a ring, we can have an interesting conversation about mapping out shapes on the floor in some kind of symbolic design. Listeners participate in a guided composition by moving in unison. 

