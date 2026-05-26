# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-page brass tuning calculator for drum corps. The app produces section-specific tuning frequencies (trumpets, mellophones, baritones, tubas/contras) that pre-compensate for the temperature/humidity delta between the indoor tuning room and the outdoor performance field - so the ensemble lands in tune on the field rather than at the tuning room.

## Project layout

The entire app is one file: `index.html` (~740 lines). HTML, CSS, and JavaScript all live there. No build step, no package manager, no dependencies to install - Alpine.js 3.13.5 is loaded from a CDN at runtime.

To run it: open `index.html` in a browser, or serve the directory (`python3 -m http.server` etc.). There are no tests, linters, or build commands.

## Architecture

Everything hangs off a single Alpine component defined in the `app()` function at the bottom of `index.html`. The model is small and worth understanding as a whole before editing:

- **State**: `units` (`F`/`C`), `indoorT`/`indoorRH`, `outdoorT`/`outdoorRH`, `targetHz`, and a `buffers` object keyed by section.
- **Persistence**: `units`, `targetHz`, and `buffers` are persisted to `localStorage` under the key `brass-tuner-v1`. Indoor/outdoor temp and humidity are intentionally _not_ persisted - they're conditions for one performance, not user preferences.
- **Core physics** (`speedOfSound`, `sectionResult`): speed of sound is computed in Celsius; the F/C unit toggle is presentation-only and converts existing input values when flipped. The per-section result models the horn's air column as a two-part blend: a fraction `(1 − β)` held near breath temperature (`cBreath`, fixed at 35°C / 100% RH) and a fraction `β` tracking ambient. Effective speeds are computed for indoor and outdoor conditions separately (`cInEff`, `cOutEff`), and the shift `adjRatio = cOutEff / cInEff` is applied as `tuneTo = targetHz / adjRatio`. β being a literal "fraction of the column at ambient" is load-bearing for the Guide's framing - don't switch back to the older `1 + β·(cOutdoor/cIndoor − 1)` form that conflates breath-temp with indoor-temp.
- **Section list** is derived in the `sections` getter from `buffers`. To add or rename a section, edit `buffers` defaults (in two places: initial state and `resetBuffers()`) and the `sections` getter.

The "Guide" tab is static content explaining the protocol, gotchas, and the buffer-factor concept - kept in-file deliberately so the calculator and its explanation ship together.

## Conventions worth knowing

- **Buffer factor (β)** is not a standard acoustics term - it's a coefficient this project coined to capture, in one number per section, how much of a horn's air column tracks ambient temperature. Defaults (0.70 / 0.75 / 0.80 / 0.90) are physics-informed estimates, not measurements. The Guide tab explains the calibration recipe; don't change defaults without a reason that fits that framing.
- **Cents formatting** (`formatCents`): positive values get a `+` prefix; negative values rely on the minus sign from `toFixed`. Don't "fix" this by always prepending a sign.
- **Temperature input range** is intentionally wide (-10°F / -23°C to 110°F / 43°C) so extreme conditions aren't blocked by the UI. A single `tempMax` getter covers both indoor and outdoor; don't reintroduce separate caps unless there's a real reason.
