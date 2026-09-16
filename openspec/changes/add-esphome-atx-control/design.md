## Context
The 2020 build (`miswired/SmartATX`) controls an ATX supply by grounding `PS_ON` with an
ESP32 powered from the always-on `+5VSB` rail. Its FauxmoESP-based control path is dead.
The hardware is unchanged and stays that way by decision; only the firmware is replaced.

Two facts constrain the design and neither was recorded in the original project:

1. **The board was never identified.** Reconstructed as a DOIT ESP32 DevKit V1 at
   confidence 7/10 from evidence in the dead sketch, later confirmed against the
   physical board.
2. **`PS_ON` idles near +5 V and ESP32 GPIOs are not 5 V tolerant.** The original OFF
   state parks a floating input at that potential.

The output is published, so decisions must survive a stranger reproducing them.

## Goals / Non-Goals
- **Goals**
  - Switch entity in Home Assistant over the native ESPHome API
  - Preserve the fail-safe: any fault or reset leaves the supply off
  - Publish-quality, commented configuration and honest documentation
  - Keep board-specific values isolated so a wrong board guess is a one-line fix
- **Non-Goals**
  - Matter, SinricPro, or any direct-to-Alexa path — voice rides Home Assistant
  - Any hardware rework or rewiring in this change
  - Current or power measurement of the rails (a separate concern if pursued)
  - Custom C++ components; YAML only unless it demonstrably cannot express the behaviour

## Decisions

### Decision: Open-drain GPIO rather than push-pull
The original sketch toggled `pinMode()` between `INPUT` (float = off) and
`OUTPUT`+`LOW` (drive = on). ESPHome expresses this directly as an open-drain output, so
the pin is never driven high and the supply's internal pull-up defines the OFF state.

- **Alternatives considered**
  - *Push-pull output with `inverted: true`* — would actively drive `PS_ON` to 3.3 V
    against the supply's 5 V pull-up, sourcing current into the PSU's control input and
    fighting it. Rejected.
  - *Replicating the `pinMode()` toggle via lambdas* — reimplements in ESPHome what the
    pin schema already provides. Rejected as unnecessary complexity.
- **Verified**: ESPHome's pin schema supports `open_drain`, with `OUTPUT_OPEN_DRAIN` and
  `INPUT_OUTPUT_OPEN_DRAIN` modes (esphome.io, checked 2026-09-09).

### Decision: `restore_mode: ALWAYS_OFF`
The supply must never come back on by itself after a power blip or crash — it may be
feeding a load nobody is standing next to. Restoring the previous state would violate the
fail-safe the original design deliberately had.

- **Alternatives considered**: `RESTORE_DEFAULT_OFF` — convenient, but reintroduces the
  possibility of an unattended self-start. Rejected for a bench supply.

### Decision: GPIO18 retained as the control pin
Keeps the wiring untouched, as decided. GPIO18 is not an ESP32 strapping pin and boots
high-impedance, which is exactly the required OFF state during the boot window before
ESPHome configures anything.

### Decision: Board identity isolated to two values
`board: esp32doit-devkit-v1` and the status LED on GPIO2 are the only board-dependent
values, both commented as assumptions. If the silkscreen later contradicts the inference,
correction is a two-line edit and no requirement changes.

### Decision: The 5 V exposure is documented, not silently reproduced
The as-built wiring stays (per the "same wiring" decision), but the article and README
state the measurement and its implication, and document the in-spec alternatives — a
series resistor, or a small-signal N-channel MOSFET as a level-safe open-drain buffer.
Publishing an out-of-spec design without saying so would fail the honesty bar this repo
is held to.

## Risks / Trade-offs
- **Board inference is wrong (Conf 7/10)** → confined to two values; a wrong guess costs
  one edit, and the silkscreen settles it in seconds.
- **`PS_ON` at ~5 V stresses GPIO18** → measure and disclose; alternatives documented.
  The as-built arrangement has years of service behind it, but service life is not a spec.
- **`+5VSB` current capacity may be marginal** on some supplies for an ESP32's WiFi
  transmit peaks → note the ESP32's peak draw and tell readers to check their supply's
  `+5VSB` rating on its label rather than assuming.
- **Home Assistant becomes a hard dependency** for voice control where the old build had
  none → accepted; it is what removes the fragility that killed the original.

## Migration Plan
1. Measure `PS_ON` idle voltage before touching the firmware — this can change the
   documentation, not the wiring.
2. Flash the ESPHome build over USB once (OTA needs a working image first).
3. Verify the fail-safe on the bench before reconnecting any real load: confirm the supply
   is off at boot, off after a reset, off with no WiFi.
4. Adopt the device in Home Assistant, confirm the switch and diagnostics.
5. Rollback: the old sketch remains at `miswired/SmartATX` and can be reflashed over USB.
   No wiring change means rollback is firmware-only.

## Open Questions
- Which repository does this publish to — a rewrite of `miswired/SmartATX`, or a new repo?
- Which open-source licence?
- What load is used for the demonstration photos?
