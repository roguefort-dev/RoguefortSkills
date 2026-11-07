# What's new

## 7 November 2026

John Wick is a new cleanup skill that sweeps a whole codebase and reports what it finds, without changing code unless you ask.

Its first pass finds implicit state machines: places where something like a loading, connection or checkout lifecycle is tracked with loose flags, status strings or nullable fields instead of one clear state. Each finding lists where it lives, the states and transitions it actually has, what can go wrong, and a suggested explicit model.
