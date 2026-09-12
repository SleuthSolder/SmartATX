# Smart ATX Bench Supply — ESPHome

Turn a scrap PC power supply into a bench supply you can switch from Home Assistant.

An ESP32 grounds the ATX supply's `PS_ON` line on command. It's powered from the
supply's always-on standby rail, so it stays awake even while the supply is off.

This replaces the [2020 build](https://miswired.io/esp32/2020/05/25/Smart-ATX/), whose
firmware emulated a Philips Hue bulb so Alexa could discover it over SSDP. Newer Echo
devices stopped finding it and the project went dark. The hardware trick was never the
problem — only the software was. Voice control now routes through Home Assistant instead
of any on-device emulation.

---

## Safety — read this before wiring anything

An ATX supply is not a hobby power brick. Treat it accordingly.

- **The 12 V rail can source tens of amps.** A short across it does not politely trip a
  breaker — it vaporises wire, melts insulation, and throws molten metal. Fuse your
  loads. Use wire rated for the current you intend to draw.
- **`+5VSB` is live whenever the supply is plugged in**, regardless of whether the main
  rails are on and regardless of the rear rocker switch on some units. Unplug the supply
  from the wall before touching any wiring — the ESP32 included.
- **Never plug USB into the ESP32 while the supply is connected to mains.** On most ESP32
  devkits, `VIN` and USB `VBUS` meet at the regulator input, frequently with no isolation
  diode between them. With the PSU live, your `Vin` lead is a 5 V source — so connecting
  USB ties your computer's 5 V rail to the supply's `+5VSB` rail and lets the two fight
  it out. Unplug the supply from the wall, or lift the `Vin` lead, before every flashing
  session. Check your own board's schematic if you want to know whether it protects you;
  assume it doesn't until you've looked.
- **Do not open the supply's case.** The primary side holds mains voltage, and its bulk
  capacitors stay charged after unplugging. Everything in this project is done at the
  24-pin connector, outside the case.
- **Never short rails to test them.** Use a meter. The "short green to black" test below
  connects a *signal* line to ground, which is what it's designed for — that is not the
  same as shorting a power rail.
- Supplies vary. Check your unit's own label for its per-rail current ratings rather than
  trusting any number you read online, this document included.

## How it works

An ATX supply's main rails stay off until `PS_ON` (pin 16, green) is pulled to ground.
The supply holds that line high itself through an internal pull-up.

That creates a chicken-and-egg problem: a controller can't ground `PS_ON` without power,
and there's no power until `PS_ON` is grounded.

`+5VSB` (pin 9, violet) resolves it. That rail is always live whenever the supply is
plugged in — it's what keeps a PC's power button and wake-on-LAN alive. It powers the
ESP32 through the dev board's onboard 5 V→3.3 V regulator, so the ESP32 is always awake
and can ground `PS_ON` whenever it's told to.

**Bench test before you build anything:** with the supply unplugged, connect the green
wire to any black wire. Plug in. The fan should spin and you should read roughly
**3.3 V** on orange, **5 V** on red, and **12 V** on yellow. (−12 V and −5 V exist on
some supplies but source very little current.) If that works, the supply is good.

## Wiring

Three connections, unchanged from the original build:

| ESP32 pin | ATX 24-pin | Wire colour | Purpose |
|-----------|------------|-------------|---------|
| `Vin`     | pin 9      | violet      | `+5VSB` — always-on standby rail |
| `GND`     | pin 19     | black       | ground |
| `GPIO18`  | pin 16     | green       | `PS_ON` — pull low to start the supply |

The control pin is configured **open-drain**: it can pull `PS_ON` low or release it
entirely, but it can never drive it high. The supply's own pull-up defines the off state.

| Switch | Pin | `PS_ON` | Supply |
|---|---|---|---|
| off | high-impedance, floating | held high by the supply's pull-up | **off** |
| on | driven to GND | low | **on** |

That means any reset, crash, reflash, or loss of WiFi leaves the supply **off** — the
firmware cannot fail into an on state. Two details make that true rather than merely
intended:

- Before ESPHome starts, GPIO18 sits at its hardware reset default — high-impedance
  input, pulls disabled — so the supply stays off through the boot window.
- ESPHome writes the off state *before* it enables the open-drain output, so the
  ESP32's `GPIO_OUT` register (which resets to 0) can't briefly pull `PS_ON` low and
  kick the supply on at boot.

The 2020 firmware reached the same two states a different way, by switching `pinMode()`
between `INPUT` (floating) and `OUTPUT` + `LOW` (grounded). Open-drain gets there without
reconfiguring the pin, and keeps the input buffer disabled — so the voltage on `PS_ON`
isn't applied to it. See the caveat below for why that matters.

### One caveat worth knowing

The ATX specification has the supply pull `PS_ON` up to `+5VSB`, and **ESP32 GPIOs are
not 5 V tolerant**. In the off state, GPIO18 sits at whatever voltage the supply holds
that line at. The original build ran this way for years without damage — the supply's
series pull-up limits current into the pin's protection diode to a few hundred
microamps — but "it survived" is not the same as "it's in spec."

> **Measurement status: not yet taken.** The `PS_ON` idle voltage on this specific supply
> has not been measured. This section will be updated with the real figure rather than an
> assumed one. See [`docs/old-build-archaeology.md`](docs/old-build-archaeology.md) §4.

If you want the in-spec version, either add a series resistor (1–10 kΩ) between GPIO18
and `PS_ON` to bound that current explicitly, or use a small-signal N-channel MOSFET as a
level-safe open-drain buffer — gate from the GPIO, drain to `PS_ON`, source to ground.
The MOSFET keeps the same fail-safe behaviour: gate low at reset means off.

## Quick start

1. **Install ESPHome** — the [Home Assistant add-on](https://esphome.io/guides/getting_started_hassio/)
   or the CLI (`pip install esphome`).

2. **Create your secrets file:**

   ```bash
   cp secrets.yaml.example secrets.yaml
   ```

   Fill in your WiFi credentials, a fallback AP password, an OTA password, and an API
   encryption key (`openssl rand -base64 32`). `secrets.yaml` is gitignored — keep it
   that way.

3. **Check the config:**

   ```bash
   esphome config smart-atx.yaml
   ```

4. **Flash over USB** (the first flash must be wired; OTA works after that):

   ```bash
   esphome run smart-atx.yaml
   ```

### Flashing without a local toolchain

`esphome run` compiles and flashes in one step, which is the simplest path if you have
ESPHome on the machine the USB cable is plugged into. If you don't, you have options —
but note that **none of them let you skip compiling somewhere**. There is no cloud build
service for ESPHome; every route ends with a binary that something had to build.

**<https://web.esphome.io/>** flashes over Web Serial in the browser. It can:

- install a factory `.bin` you built elsewhere,
- install a generic adoptable ESPHome firmware ("Prepare for first use") and set up
  Wi-Fi, so an ESPHome Device Builder can adopt the device over the network,
- show live serial logs, and reconfigure Wi-Fi.

It needs Chrome or Edge on desktop — Web Serial doesn't exist in Firefox or Safari, and
not on iOS at all.

So if your Home Assistant box is headless in another room: compile with the ESPHome
Device Builder add-on there, use **Download firmware binary** to get the factory `.bin`,
carry it to the machine at the bench, and flash it at web.esphome.io.

> **Why not flash straight from the Home Assistant add-on?** Its "plug into this
> computer" option also uses Web Serial, and browsers only expose Web Serial in a
> *secure context*. A Home Assistant instance reached over plain `http://` isn't one, so
> the option is unavailable — nothing is broken, the browser is refusing by design.
> web.esphome.io is served over HTTPS, which is exactly why it works where the add-on
> doesn't.

After the first flash everything is over-the-air anyway, so this choice only matters once.

5. **Adopt it in Home Assistant.** The device announces itself; Settings → Devices &
   Services should offer it under the ESPHome integration. You'll be asked for the API
   encryption key from step 2.

6. **Verify the fail-safe before connecting a real load.** Confirm the supply is off at
   boot, off after pressing reset, and off when WiFi is unavailable. Only then wire up
   something you care about.

## In Home Assistant

The device exposes:

| Entity | Type | Notes |
|---|---|---|
| **Power** | switch | The supply itself |
| Uptime | sensor | diagnostic |
| WiFi Signal | sensor | diagnostic |
| Heap Free | sensor | diagnostic |
| Reset Reason | text sensor | diagnostic — useful for diagnosing unexpected reboots |
| Restart | button | diagnostic |

The onboard LED mirrors the supply's state, so you can see what it's doing at the bench
without opening an app.

**Voice control** comes from Home Assistant's own Alexa or Google integrations — expose
the `Power` switch through whichever you use. Nothing on the ESP32 talks to a voice
assistant directly, which is precisely why this version won't rot the way the last one
did: when Amazon changes how discovery works, Home Assistant absorbs it.

## Adapting to another board

Three values in `smart-atx.yaml`, all commented in place:

- `board:` — the PlatformIO board ID
- `status_led_pin` — wherever your board's onboard LED lives
- `ps_on_pin` — the control pin

If you change the control pin, check the ESP32 strapping-pin table first. It must boot
high-impedance, or the supply may kick on during the boot window before ESPHome takes
over. GPIO18 is safe; GPIO0, 2, 5, 12 and 15 are not.

> The board this repo targets (`esp32doit-devkit-v1`) is **inferred**, not confirmed —
> the 2020 project never recorded it. The reasoning and its confidence level are in
> [`docs/old-build-archaeology.md`](docs/old-build-archaeology.md).

## Alternatives not taken

- **Matter-native** — the ESP32 speaks Matter directly to Alexa, Google, or Apple, with
  no Home Assistant in the middle. The cleanest modern path if you don't already run HA.
  Not tested here.
- **SinricPro** — a cloud service that's robust across firmware churn, at the cost of an
  account and an external dependency.
- **A patched FauxmoESP fork** — only worth it if you specifically want Alexa-direct
  control with no hub. It's the approach that died; patching it means signing up to
  re-patch it whenever discovery changes again.

## Repository layout

```
smart-atx.yaml          the ESPHome configuration
secrets.yaml.example    template — copy to secrets.yaml and fill in
docs/                   how the board was identified, and what came from the old build
openspec/               specs and change proposals driving this project
```

## References

- Original writeup (2020): <https://miswired.io/esp32/2020/05/25/Smart-ATX/>
- Original firmware, now superseded: <https://github.com/miswired/SmartATX>
- ESPHome: <https://esphome.io/>
- ESPHome pin schema (`open_drain`): <https://esphome.io/guides/configuration-types/>
