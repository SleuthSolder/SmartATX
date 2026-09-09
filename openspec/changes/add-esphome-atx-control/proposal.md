# Change: Replace FauxmoESP firmware with an ESPHome ATX power controller

## Why
The 2020 Smart ATX build's control firmware is dead. It relied on FauxmoESP emulating a
Philips Hue bulb so Alexa could discover it over SSDP/UPnP; gen4/Show Echo devices
tightened discovery and stopped finding it. The hardware trick — powering the ESP32 from
`+5VSB` so it can ground `PS_ON` — is still sound.

Replace the brain, keep the body: an ESPHome configuration that exposes the supply as a
switch to Home Assistant, with voice control riding HA rather than on-device emulation.
The result is published as an open project and referenced by a Sleuth & Solder article,
so it must be reproducible by a stranger with a junk PSU.

## What Changes
- **BREAKING**: all FauxmoESP / Hue-emulation / SSDP discovery code is removed. The
  Alexa-direct control path no longer exists; Alexa reaches the device through Home
  Assistant instead. `credentials.h` is replaced by ESPHome `secrets.yaml`.
- Add an ESPHome YAML config for a DOIT ESP32 DevKit V1 that exposes the PSU as a switch
  over the native ESPHome API.
- Preserve the original fail-safe: control GPIO is **open-drain** — driven low for ON,
  high-impedance for OFF — so a reset, crash, or reflash leaves the supply off.
- Add diagnostics: onboard LED mirrors PSU state; uptime, WiFi signal, and free-heap
  sensors; a restart button.
- Add publish-quality documentation: wiring table, the `+5VSB` rationale, the bench test,
  Home Assistant setup, and an honest safety section.
- Measure and document the `PS_ON` idle voltage against the ESP32's non-5V-tolerant
  input rating, rather than repeating the original design silently.

## Impact
- Affected specs: `atx-power-control` (new), `device-diagnostics` (new),
  `build-documentation` (new)
- Affected code: `smart-atx.yaml` (new), `secrets.yaml.example` (new), `.gitignore`,
  `README.md`, `docs/`
- Superseded: <https://github.com/miswired/SmartATX> `Code/fauxmoESP_ATX_Control.ino`
- Hardware: unchanged wiring — Vin ← +5VSB (pin 9), GND ← GND (pin 19),
  GPIO18 → PS_ON (pin 16)
