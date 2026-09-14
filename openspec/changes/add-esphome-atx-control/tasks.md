## 1. Bench verification (before writing firmware)
- [x] 1.1 Confirm the board from the silkscreen and record it in `docs/old-build-archaeology.md`
- [ ] 1.2 Measure `PS_ON` idle voltage with the supply in standby and the ESP32 disconnected; record the value
- [ ] 1.3 Record the supply's `+5VSB` current rating from its label
- [ ] 1.4 Bench-test the supply: short green to black, confirm 3.3 V / 5 V / 12 V on the rails

## 2. ESPHome configuration
- [x] 2.1 Add `.gitignore` covering `secrets.yaml` and ESPHome build artifacts
- [x] 2.2 Add `secrets.yaml.example` documenting every required key
- [x] 2.3 Write `smart-atx.yaml`: esp32 board, WiFi, native API, OTA, logger
- [x] 2.4 Add the PSU switch on GPIO18 as open-drain, `restore_mode: ALWAYS_OFF`, with comments
- [x] 2.5 Add the onboard-LED status indicator mirroring the switch state
- [x] 2.6 Add diagnostic entities: uptime, WiFi signal, free heap, restart button
- [x] 2.7 Validate the config with `esphome config smart-atx.yaml`

## 3. Hardware verification
- [ ] 3.1 Flash over USB and confirm the device appears in Home Assistant
- [ ] 3.2 Verify switch on drives the supply on; switch off drives it off
- [ ] 3.3 Verify fail-safe: supply is off at boot, after a reset, and with no WiFi available
- [ ] 3.4 Verify the supply stays off through an OTA update
- [ ] 3.5 Verify the status LED tracks the switch, and diagnostics report in Home Assistant

## 4. Documentation
- [x] 4.1 Write `README.md`: what it is, the `+5VSB` insight, wiring table, quick start
- [x] 4.2 Write the safety section, placed ahead of the wiring steps
- [x] 4.3 Document the Home Assistant setup and how voice control now routes through HA
- [ ] 4.4 Document the `PS_ON` 5 V finding, the measured value, and the in-spec alternatives
      (finding and alternatives written up; blocked on the measurement from task 1.2)
- [x] 4.5 Record the alternatives not taken (Matter, SinricPro, patched FauxmoESP) with one line each
- [x] 4.6 Confirm no credentials appear in any tracked file or in history

- [x] 4.7 Document why the Home Assistant add-on cannot flash over USB on a plain-http instance
- [x] 4.8 Document the USB / `+5VSB` backfeed hazard in the safety section
- [x] 4.9 Write `docs/setup-and-flash-guide.md` with real command transcripts under `docs/logs/`
- [ ] 4.10 Capture the 7 screenshots listed in the setup guide (GUI steps; requires a desktop session)

## 5. Close out
- [ ] 5.1 Cross-check the finished config and docs against the article guide at
      `~/Dropbox/reports/sleuth-solder-article-guides/smart-atx.html`
- [ ] 5.2 Resolve the open questions in `design.md` (target repo, licence, demo load)
- [ ] 5.3 Archive this change with `openspec archive add-esphome-atx-control --yes`
