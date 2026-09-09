# Project Context

## Purpose
Revive the 2020 **Smart ATX Power Supply** build by replacing its dead control
firmware with an **ESPHome** configuration.

An ESP32 turns a scrap PC (ATX) power supply into an automation-controllable bench
supply by grounding the `PS_ON` line. The original firmware emulated a Philips Hue
bulb via FauxmoESP so Alexa could discover it over SSDP/UPnP; newer Echo devices
(gen4 / Show) stopped discovering it and the build went dark.

The hardware trick is sound and stays. Only the brain is replaced: ESPHome exposes
the supply as a switch entity to Home Assistant, and voice control rides HA rather
than any on-device cloud emulation.

This repo is also the reference implementation linked from a public Sleuth & Solder
article, so it is held to publish quality: reproducible steps, honest safety notes,
no invented numbers.

## Tech Stack
- **ESPHome** (YAML configuration) — firmware
- **ESP32** — DOIT ESP32 DevKit V1, 30-pin (see Important Constraints)
- **Home Assistant** — native ESPHome API; owns automations and voice
- No Arduino sketch, no C++ custom components unless YAML proves insufficient

## Project Conventions

### Code Style
- ESPHome YAML, 2-space indent, snake_case ids
- Every non-obvious block carries a comment explaining *why* — the config is
  published as teaching material, not just working firmware
- Secrets (WiFi SSID/password, API/OTA keys) live in `secrets.yaml`, which is
  gitignored. Never inline credentials; commit a `secrets.yaml.example` instead.

### Architecture Patterns
- The ESP owns exactly one job: drive `PS_ON` safely and report state.
- All logic that can live in Home Assistant does live there.
- Fail-safe by default: any reset, crash, or power loss must leave the PSU **off**.

### Testing Strategy
Hardware project — verification is on the bench, not in CI:
- Measure before asserting. Voltages and currents in docs come from a meter or are
  marked TBD; none are invented.
- Every requirement's scenario is checked against real hardware before its task is
  ticked.

### Git Workflow
- Local repo, commit after every meaningful change so any point is recoverable.
- Never push to a remote without explicit permission.
- Tick `tasks.md` checkboxes in the same commit that delivers the work.
- Archive changes with `openspec archive <id> --yes` once shipped.
- Note: this OpenSpec version installs `/openspec:proposal|apply|archive`
  slash commands (not `/opsx:*`).

## Domain Context
- **ATX `PS_ON`**: the supply's main rails stay off until `PS_ON` (pin 16, green) is
  pulled to ground. The PSU pulls it up internally to `+5VSB`.
- **`+5VSB`** (pin 9, violet): always-on standby rail. It powers the ESP32 so the ESP32
  can ground `PS_ON` — resolving the chicken-and-egg of a controller that needs the
  supply it controls to be on.
- **Float-high / drive-low**: the original sketch held the control GPIO as `INPUT`
  (floats high = off) and switched to `OUTPUT` + `LOW` to turn on. This is the safe
  default and must survive the rewrite — in ESPHome it maps to an **open-drain** output.

## Important Constraints
- **Board is an assumption, not a confirmation.** The old sketch and blog post never
  name the board. `board: esp32doit-devkit-v1` is inferred from evidence documented in
  `docs/old-build-archaeology.md` (Conf 7/10). It is confined to a single YAML key plus
  the status-LED pin so it can be corrected in one edit.
- **Wiring is unchanged** from the 2020 build, by decision. Vin ← +5VSB (pin 9),
  GND ← GND (pin 19), GPIO18 → PS_ON (pin 16).
- **`PS_ON` idles near +5 V and ESP32 GPIOs are not 5 V tolerant.** The original design
  parks a floating input at that potential. It ran for years, but it is out of spec and
  must be measured and documented honestly before publication — see the archaeology doc.
- **Safety**: 12 V rail sources large current; shorting rails during bench work is a real
  hazard. Documented plainly, not hand-waved.

## External Dependencies
- **Home Assistant** with the ESPHome integration (native API)
- **ESPHome** toolchain (dashboard or CLI) for build and OTA
- Old project, for reference only: https://github.com/miswired/SmartATX
- Original writeup: https://miswired.io/esp32/2020/05/25/Smart-ATX/
- Article plan: `~/Dropbox/reports/sleuth-solder-article-guides/smart-atx.html`
