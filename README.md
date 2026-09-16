# SmartATX

Turn a scrap PC power supply into a bench supply you can switch from Home Assistant.

An ESP32 grounds the ATX supply's `PS_ON` line on command. It runs from the supply's
always-on standby rail, so it stays awake even while the supply is off.

> **Full build walkthrough:** *(website link — TBD)*
>
> Wiring photos, the flashing walkthrough with screenshots, Home Assistant setup, and the
> story of why the original version stopped working all live there. This repo is the
> reference configuration that article points to.

---

## This is a rewrite

The original 2020 version was an Arduino sketch that used
[FauxmoESP](https://bitbucket.org/xoseperez/fauxmoesp/src/master/) to emulate a Philips Hue
bulb, so Alexa could discover it over SSDP/UPnP. It worked for years. Then newer Echo
devices tightened discovery and stopped finding it, and the project went dark.

The hardware trick was never the problem — only the software was. This version throws out
the emulation entirely and rebuilds control on **ESPHome**, with voice control riding
Home Assistant instead of anything running on the ESP32.

The original sketch is preserved at the [`v1-fauxmoesp`](../../tree/v1-fauxmoesp) tag.

---

## Safety

An ATX supply is not a hobby power brick.

- **The 12 V rail can source tens of amps.** A short across it vaporises wire rather than
  politely tripping anything. Fuse your loads and use appropriately rated wire.
- **`+5VSB` is live whenever the supply is plugged in**, regardless of the main rails or
  the rear rocker switch. Unplug it from the wall before touching any wiring.
- **Never plug USB into the ESP32 while the supply is connected to mains.** On most ESP32
  devkits `VIN` and USB `VBUS` meet at the regulator input with no isolation diode, so a
  live supply ties your computer's 5 V rail to `+5VSB`.
- **Do not open the supply's case.** Mains voltage on the primary side, and the bulk
  capacitors stay charged after unplugging. Everything here happens at the 24-pin
  connector, outside the case.

## How it works

An ATX supply's main rails stay off until `PS_ON` (pin 16, green) is pulled to ground. The
supply holds that line high itself through an internal pull-up.

That's a chicken-and-egg problem: a controller can't ground `PS_ON` without power, and
there's no power until `PS_ON` is grounded.

`+5VSB` (pin 9, violet) solves it. That rail is always live while the supply is plugged in
— it's what keeps a PC's power button alive. It feeds the ESP32 through the dev board's
onboard regulator, so the ESP32 is always awake and can ground `PS_ON` on command.

## Wiring

| ESP32 pin | ATX 24-pin | Wire colour | Purpose |
|-----------|------------|-------------|---------|
| `Vin`     | pin 9      | violet      | `+5VSB` — always-on standby rail |
| `GND`     | pin 19     | black       | ground |
| `GPIO18`  | pin 16     | green       | `PS_ON` — pull low to start the supply |

Built and tested on an **ESP32 DEVKIT V1**.

The control pin is **open-drain**: it can pull `PS_ON` low or release it, but never drive
it high. The supply's own pull-up defines the off state, so any reset, crash, reflash, or
loss of WiFi leaves the supply **off**. The firmware cannot fail into an on state.

### A caveat worth knowing

**ESP32 GPIOs are not 5 V tolerant** — absolute maximum input is VDD + 0.3 V ≈ 3.6 V.

On the supply used here, `PS_ON` idles at a **measured ~3.8 V** with **under 1 mA** flowing.
That is fractionally above the rated maximum, which is why the original build ran for years
without damage — but it is still outside the datasheet envelope.

**Measure your own supply before assuming it behaves the same.** The ATX specification has
`PS_ON` pulled up to `+5VSB`, and a supply that actually does that would put ~5 V on a pin
rated for 3.6 V. Meter between the green wire and any black wire with the supply in standby
and the ESP32 disconnected.

If you want the in-spec version: put a series resistor (1–10 kΩ) between the GPIO and
`PS_ON` to bound the current explicitly, or use a small-signal N-channel MOSFET as a
level-safe open-drain buffer — gate from the GPIO, drain to `PS_ON`, source to ground. The
MOSFET keeps the same fail-safe: gate low at reset means off.

## Quick start

```bash
# 1. Install ESPHome
pipx install esphome            # or: uv tool install esphome

# 2. Fill in your secrets
cp secrets.yaml.example secrets.yaml
openssl rand -base64 32         # for api_encryption_key
$EDITOR secrets.yaml

# 3. Check it
esphome config smart-atx.yaml

# 4. Flash over USB (first flash must be wired; OTA after that)
esphome run smart-atx.yaml
```

Then adopt it in Home Assistant under **Settings → Devices & Services**, using the API
encryption key from your `secrets.yaml`.

Use `esphome run`, not `esphome upload` — `upload` flashes the last binary that was built
and does not recompile, which will happily write stale credentials to the board.

## What you get in Home Assistant

| Entity | Type |
|---|---|
| **Power** | switch — the supply itself |
| Uptime | sensor (diagnostic) |
| WiFi Signal | sensor (diagnostic) |
| Heap Free | sensor (diagnostic) |
| Reset Reason | text sensor (diagnostic) |
| Restart | button (diagnostic) |

The onboard LED mirrors the supply's state, so you can see what it's doing at the bench.

Voice control comes from Home Assistant's own Alexa or Google integrations — expose the
`Power` switch through whichever you use. Nothing on the ESP32 talks to a voice assistant
directly, which is why this version won't rot the way the last one did.

## Adapting it

Three values in `smart-atx.yaml`, all commented in place: `board`, `status_led_pin`, and
`ps_on_pin`. If you change the control pin, check the ESP32 strapping-pin table first — it
must boot high-impedance or the supply may kick on during boot. GPIO18 is safe; GPIO0, 2,
5, 12 and 15 are not.

## Repository layout

```
smart-atx.yaml          the ESPHome configuration
secrets.yaml.example    template — copy to secrets.yaml and fill in
openspec/               the requirements this firmware is built against
```

## Licence

[MIT](LICENSE).
