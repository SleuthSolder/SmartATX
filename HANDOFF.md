# esp-light — Handoff

**Read this first.** This is the starting brief for reviving Jason's old "Smart ATX" project by
completely rewriting it for **ESPHome**, well-documented, so it can be **publicly posted**.

---

## What this project is

Jason built a **Smart ATX Power Supply** in 2020: an ESP32 turns a free PC (ATX) power supply into a
voice-/automation-controllable bench supply. It worked for years, then the control software **died** —
it relied on FauxmoESP emulating a Philips Hue bulb so Alexa could discover it on the LAN, and newer
Echoes (gen4/Show) stopped discovering it after Amazon tightened SSDP/UPnP handling.

**The hardware trick is still perfect. The brain rotted. This project replaces the brain.**

Goal: **rip out the FauxmoESP/Alexa code entirely and rebuild control on ESPHome + Home Assistant**
(what Jason runs now), documented well enough to publish as an open project + a Sleuth & Solder article.

## Sources — the old project

- **Old code (dead):** https://github.com/miswired/SmartATX — default branch `master`, last push 2020-02-17.
  - `Code/fauxmoESP_ATX_Control.ino` — the ESP32 sketch (FauxmoESP-based)
  - `Code/credentials.h` — WiFi SSID/pass defines
  - `README.md` — one line
- **Old writeup:** https://miswired.io/esp32/2020/05/25/Smart-ATX/ — the original tutorial-style post.
- **Article guide (Sleuth & Solder):** `~/Dropbox/reports/sleuth-solder-article-guides/smart-atx.html`
  — the writing plan for the *article* about this revival. This code project is the thing that article
  will link to. Keep them consistent.

## The hardware (unchanged — keep it)

- **ATX PSU:** off until `PS_ON` (pin 16, green wire) is pulled to GND. Rails: Orange 3.3V, Red 5V,
  Yellow 12V; −12/−5V exist but weak. Bench test = short green→black, PSU spins up.
- **The +5VSB trick:** the ESP32 can't ground `PS_ON` until it has power, but the PSU is off. Solution:
  power the ESP32 from **+5VSB (pin 9, violet)** — the always-on standby rail — via the board's onboard
  5V→3.3V regulator. So the ESP is always alive and grounds `PS_ON` on command.
- **Connections (from old build):**
  - `Vin`  ← `+5VSB` (pin 9, violet)
  - `GND`  ← `GND` (pin 19, black)
  - control GPIO → `PS_ON` (pin 16, green) — old sketch used GPIO18
- **Control scheme worth preserving:** old sketch holds the pin as **INPUT (floats high via pullup = OFF)**
  and switches to **OUTPUT + drive LOW = ON**. Safe default (supply stays off if the ESP resets/hangs).
  In ESPHome this maps most cleanly to an **open-drain GPIO output / switch** (drive low = on, high-Z = off)
  rather than a push-pull output that would actively drive `PS_ON` high — confirm the safe-default
  behavior survives the rewrite. **Verify on the bench, don't assume.**

## What to build

- A clean **ESPHome YAML** config for an ESP32 that:
  - exposes the PSU as a **switch** entity to Home Assistant (native ESPHome API, no FauxmoESP),
  - preserves the safe float-high/drive-low default on `PS_ON`,
  - is powered from +5VSB, so it survives reboots and reports state to HA,
  - optional niceties: status LED, uptime/heap sensors, a physical button, boot-state = OFF.
- Voice control (Alexa/Google) then rides **Home Assistant**, not the ESP — no local emulation hacks.
- **Documentation to publish:** wiring (reuse the pinout figures), the +5VSB rationale, the full YAML with
  comments, HA setup, and safety. Aim it at someone reproducing it from a junk PSU.

## Alternatives to mention (article covers these; not what we're building)

- **Matter-native** — ESP32 speaks Matter directly to Alexa/Google/Apple, no HA required. Cleanest modern
  path; note as an option (not verified here). Conf 7/10.
- **SinricPro** — cloud service, robust across firmware, but adds an account/dependency.
- **Patched FauxmoESP fork** (e.g. USN/`Basic:1` fix + port 80) — only if someone wants to keep Alexa-direct.

## Constraints & workflow (per Jason's global rules)

- **OpenSpec first.** This is a code project → spec it before writing code: `/opsx:propose` → `/opsx:apply`
  → `/opsx:archive`. Tick `tasks.md` in the same commit that delivers the work.
- **Git:** this folder must be a local repo (init if not). **Commit locally on every change. Never push
  without explicit permission.**
- **Safety:** high-current 12V rail, shorting rails during bench work — document it honestly.
- **Accuracy:** don't invent current figures, board models, or pin numbers — pull from the real build or
  mark as TBD. Verify ESPHome/HA/Matter specifics against current docs (knowledge may be stale).
- **Public-post quality:** this is meant to be published, so hold it to that bar — clean YAML, real photos,
  reproducible steps, references.

## Open questions for Jason

- Which ESP32 board is on the current unit (or a fresh one)?
- Keep it ESPHome-only, or also ship a Matter variant as a "modern option"?
- Reuse the existing PSU/wiring as-is, or rebuild cleanly for photos?
- Open-source license for the repo? Rewrite in the `SmartATX` repo, or a new one?

---
*Handoff written 2026-09-09. If anything here conflicts with what Jason tells you in-session, he wins —
this is a starting point, not gospel.*
