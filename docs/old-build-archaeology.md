# Old Build Archaeology — identifying the 2020 Smart ATX hardware

*Written 2026-09-09. Source material: [miswired/SmartATX](https://github.com/miswired/SmartATX)
@ `master` (last push 2020-02-17) and the original post,
[Smart ATX Power Supply](https://miswired.io/esp32/2020/05/25/Smart-ATX/) (2020-05-25).*

The 2020 build never recorded which ESP32 board it used. The article plan lists
"which ESP32 board?" as an open info gap. This document reconstructs the answer from
evidence in the surviving code, so the ESPHome rewrite can target a real board instead
of a guess — and so the article can state it with a known confidence rather than
inventing it.

---

## 1. What the old code actually contains

The repo holds three files:

| File | Contents |
|---|---|
| `README.md` | One sentence. No hardware detail. |
| `Code/credentials.h` | Two `#define`s — placeholder SSID and password. |
| `Code/fauxmoESP_ATX_Control.ino` | 149 lines. A lightly modified `fauxmoESP_Basic` demo. |

The control logic is the whole of the custom work (`fauxmoESP_ATX_Control.ino:110-125`):

```cpp
if (state) {
    digitalWrite(LED_BUILTIN, HIGH);
    pinMode(LIGHT_CONTROL_PIN, OUTPUT);      // GPIO18
    digitalWrite(LIGHT_CONTROL_PIN, LOW);    // drive low  = PSU ON
} else {
    digitalWrite(LED_BUILTIN, LOW);
    pinMode(LIGHT_CONTROL_PIN, INPUT);       // high-Z     = PSU OFF
}
```

Everything else is FauxmoESP boilerplate: WiFi station setup, `fauxmo.setPort(80)`
(the gen3 Echo workaround, already present), a `addDevice()` call, and a free-heap
print in `loop()`.

**Board identification appears nowhere** — no `#define`, no comment, no `platformio.ini`,
no Arduino `.json` sketch metadata. The blog post's "Gear Needed" list says only
"ESP32 Boards", with no link.

## 2. Evidence

### 2.1 `LED_BUILTIN` is used unguarded — this is the strongest clue

The sketch references `LED_BUILTIN` at lines 65, 66, 115 and 122 with no `#ifdef` guard.
For the sketch to compile, the selected board's variant header must define it.

I fetched all 63 `variants/*/pins_arduino.h` files from `espressif/arduino-esp32` at tag
**1.0.4** (the core current in early 2020) and checked which define `LED_BUILTIN`:

- **44 of 63 variants define it.**
- **The generic `esp32` variant — Arduino IDE's "ESP32 Dev Module" — does not.**

So the board selected in the IDE was *not* the generic Dev Module. That eliminates the
single most common default and narrows the field to a named board with an onboard LED.

Variants defining `LED_BUILTIN` in 1.0.4:

```
alksesp32 d1_mini32 d32 d32_pro doitESP32devkitV1 esp320 esp32-gateway esp32thing
espea32 espectro32 espino32 feather_esp32 firebeetle32 fm-devkit gpy
heltec_wifi_kit_32 heltec_wifi_lora_32 heltec_wifi_lora_32_V2 heltec_wireless_stick
hornbill32dev intorobot-fig lolin32 lopy lopy4 magicbit mhetesp32devkit
mhetesp32minikit Microduino-esp32 nano32 node32s nodemcu-32s odroid_esp32
onehorse32dev oroca_edubot pocket_32 sparkfun_lora_gateway_1-channel t-beam
ttgo-lora32-v1 ttgo-t1 turta_iot_node vintlabsdevkitv1 widora-air wipy3 xinabox
```

*Confidence: 10/10 — verified directly against the 1.0.4 tag.*

### 2.2 The post numbers the board's header pins

The wiring list in the original post reads:

> Vin (**Pin1**) = +5VSB (Pin 9, Violet Wire)
> GND (**Pin2**) = GND (Pin19, Black Wire)
> Control (**Pin18**) = PS_ON (Pin 16, Green Wire)

"Pin1" and "Pin2" are positions on the dev board's header, and they are `VIN` then `GND`,
adjacent, at a corner. That is the pin order of the 30-pin DOIT-style layout
(`VIN, GND, D13, D12, D14, D27, …` down one side).

It rules out the 38-pin NodeMCU-32S / `node32s` family, whose corner begins
`3V3, EN, VP, VN, …` with `VIN` at the opposite end.

*Confidence: 7/10 — the pin numbering is unambiguous in the post; mapping it to a
specific silkscreen relies on the standard DOIT layout.*

### 2.3 The post's setup instructions point at Random Nerd Tutorials

For environment setup the post links three RNT guides (getting started, Mac/Linux
install, Windows install). RNT's ESP32 material is written around the **DOIT ESP32
DEVKIT V1** — the board a reader following those guides in 2020 would most likely have
bought and selected.

*Confidence: 6/10 — circumstantial, but it agrees with 2.1 and 2.2.*

### 2.4 Consistency check

`doitESP32devkitV1/pins_arduino.h` defines `LED_BUILTIN = 2`, exposes GPIO18 on the
header (it is `SCK` in the variant, unused here), and matches the VIN/GND corner order.
No contradiction with any evidence above.

## 3. Conclusion

**DOIT ESP32 DevKit V1, 30-pin.** ESPHome target:

```yaml
esp32:
  board: esp32doit-devkit-v1
```

with the onboard LED on **GPIO2** and PSU control on **GPIO18**.

**CONFIRMED (2026-09-14)** by reading the silkscreen on the physical unit: ESP32
DEVKIT V1. No change to the configuration was needed.

The inference above was published at confidence 7/10 before the board was checked. It is
left intact rather than rewritten, because the reasoning is the useful part — the
`LED_BUILTIN` evidence (2.1) and the header pin ordering (2.2) each independently
narrowed the field, and the circumstantial Random Nerd Tutorials link (2.3) turned out to
point the right way. A reader facing the same problem on a different dead project can run
the same method.

## 4. Finding not in the original writeup: `PS_ON` idles above the ESP32's rating

The ATX specification has the power supply pull `PS_ON` up to **+5VSB** internally
(commonly through roughly 10 kΩ). The original design's OFF state parks GPIO18 as a
**floating input at that potential**.

ESP32 GPIOs are **not 5 V tolerant**. Their absolute maximum input is tied to the 3.3 V
supply plus a small margin, so a pin sitting at ~5 V is outside the datasheet limit. The
pin's ESD protection diode conducts into the 3.3 V rail, and the PSU's series pull-up
resistor limits that to a few hundred microamps — which is why the build survived years
of use rather than failing outright. It worked; it was not in spec.

This matters because the rewrite is being published as a reproducible project. Options:

1. **Measure and document.** Meter on the green wire with the PSU in standby. If it reads
   ~5 V, say so plainly in the article and note the caveat.
2. **Series resistor** (e.g. 1–10 kΩ) between GPIO18 and `PS_ON`, bounding the diode
   current explicitly rather than relying on the PSU's internal pull-up value.
3. **Small-signal N-channel MOSFET** as a level-safe open-drain buffer — gate from the
   GPIO, drain to `PS_ON`, source to ground. Fully removes the ESP32 from the 5 V node
   and keeps the fail-safe (gate low at reset = off).

Current decision: **wiring stays as built** (option 1) — measure, then document honestly.
Options 2 and 3 are noted for readers who want the in-spec version.

### Bench measurements, 2026-09

| Node | Reading |
|---|---|
| `+5VSB` (pin 9, violet) | **4.99 V** |
| `PS_ON` (pin 16, green), supply in standby | **~3.8 V** |

`+5VSB` at 4.99 V is unambiguous — a solid 5 V rail for the board's regulator.

**`PS_ON` at ~3.8 V requires care in interpretation**, and the article should not state it
flatly without resolving this. Two different situations produce that same reading:

1. **Control wire disconnected from the ESP32.** Then 3.8 V is the supply's genuine
   open-circuit pull-up voltage. It sits just above the ESP32's absolute-maximum input
   (VDD + 0.3 V = 3.6 V), so the exposure is real but small, and the original design was
   closer to in-spec than the ATX specification alone would suggest.

2. **Control wire still connected to the ESP32.** Then the measurement is of the ESP32
   *clamping* the line, not of the supply driving it. The pad's ESD diode conducts into
   the 3.3 V rail and holds the node at roughly 3.3 V + one diode drop — which lands at
   about 3.8 V. The supply's true open-circuit voltage would be higher, plausibly the
   ~5 V the ATX specification implies, and the diode would be conducting continuously
   for as long as the supply sits in standby.

The numerical coincidence between "a supply that pulls up to 3.8 V" and "an ESP32 clamping
something higher" is close enough that the two cannot be told apart from the reading alone.

**To resolve:** disconnect the green wire from the ESP32 entirely, leave the supply plugged
in and in standby, and measure green-to-black with nothing else attached. Still ~3.8 V
confirms case 1. A jump toward ~5 V confirms case 2.

*Confidence: 8/10 on the ATX specification pulling `PS_ON` up to `+5VSB`. The measurement
above is recorded as taken; its interpretation is open pending the test described.*

## 5. What carries into the rewrite

| From the old build | Status |
|---|---|
| `+5VSB` powers the ESP32 | **Keep** — the core insight |
| GPIO18 → `PS_ON`, float-high / drive-low | **Keep** — maps to ESPHome open-drain |
| Onboard LED mirrors PSU state | **Keep** — free status indicator (GPIO2) |
| Wiring: Vin←pin 9, GND←pin 19, GPIO18←pin 16 | **Keep** — unchanged by decision |
| FauxmoESP / Hue emulation / SSDP discovery | **Delete** — this is what died |
| `fauxmo.setPort(80)` gen3 workaround | **Delete** — obsolete with the library |
| Hand-rolled WiFi station setup | **Delete** — ESPHome handles it |
| `credentials.h` with inline SSID/password | **Delete** — replaced by `secrets.yaml` |
| Free-heap print in `loop()` | **Replace** — ESPHome has uptime/heap sensors |

## 6. Erratum in the original post (for the article)

The post says "install the FauxMoESP library", then instructs the reader to download and
add **AsyncTCP**. AsyncTCP is a *dependency* of FauxmoESP, not the library itself — a
reader following the steps literally would end up without FauxmoESP. Worth correcting if
the setup section is quoted.

---

## References

- Old code: <https://github.com/miswired/SmartATX> — `Code/fauxmoESP_ATX_Control.ino`
- Old writeup: <https://miswired.io/esp32/2020/05/25/Smart-ATX/>
- arduino-esp32 variants @ 1.0.4: <https://github.com/espressif/arduino-esp32/tree/1.0.4/variants>
- ESPHome pin schema (`open_drain`, `OUTPUT_OPEN_DRAIN`):
  <https://esphome.io/guides/configuration-types/>
- FauxmoESP: <https://bitbucket.org/xoseperez/fauxmoesp/src/master/>
