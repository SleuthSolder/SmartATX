# SmartATX

An ESP32 that turns an old ATX computer power supply into something Home Assistant can
switch on and off. I use mine to run a 3D-printed LED arch lamp.

> **Full writeup:** *(website link — TBD)*
>
> Wiring photos, the flashing walkthrough, and the whole story are over there. This repo is
> just the config.

## Why this exists

Back in 2020 I had a pile of old ATX supplies and a light that needed power. ATX supplies
are great for this — clean power, low voltage, lots of current, and built-in overcurrent
protection. The catch was controlling it.

You might be tempted to stick a smart plug on the mains side and call it a day, but ATX
supplies don't love being power cycled like that. There's a better way. The supply has a
control line that turns the outputs on when you tie it to ground, and a standby line that's
always live. That's begging for a microcontroller.

So I grabbed an ESP32 and used [FauxmoESP](https://bitbucket.org/xoseperez/fauxmoesp/src/master/)
to make it pretend to be a Philips Hue bulb, which was one of the few things Alexa could
control locally without round-tripping through someone's server. It worked really well for
years, until Alexa changed how device discovery worked and it stopped answering. Chasing
protocol changes got old and the whole thing ended up on a shelf.

These days I run Home Assistant, so this is the same hardware with
[ESPHome](https://esphome.io/) in place of the old firmware. Voice control goes through
Home Assistant now instead of anything running on the ESP32, which means the next time a
cloud vendor changes their mind it isn't my problem.

The old sketch is still in the git history, tagged [`v1-fauxmoesp`](../../tree/v1-fauxmoesp)
if you want to look at it.

## The wiring

Three wires.

| ESP32 pin | ATX wire | What it does |
|---|---|---|
| `Vin` | `+5VSB` (violet, pin 9) | The always-on standby line. 5 V, low current. It feeds the board's onboard regulator, which gives the ESP32 its 3.3 V. This is the whole trick — the supply is off, but the ESP32 is awake and ready to turn it on. |
| `GND` | `GND` (black, pin 19) | Common ground, so everything shares a return path and a reference. |
| `GPIO18` | `PS_ON` (green, pin 16) | The control line. Pull it to ground and the supply powers up. Release it and the supply's internal pullup takes it high again, and everything shuts off. |

I didn't bother with a case. The wires are soldered straight to the header pins and heat
shrunk. Everything on this side is low voltage with short protection, so there's not much
risk to you — worst case you short something and kill the board. Don't open the supply
itself unless you know your way around mains voltage.

## One electrical note

With the ESP32's pin released, `PS_ON` sits at about **3.8 V** and draws basically nothing —
under 1 mA, near zero on my meter. Pulling it down to switch the supply on draws about 1 mA.

That 3.8 V is a hair over the ESP32's input rating, but at that current it's been perfectly
happy and has run for years. If you want to do it properly, put a small N-channel MOSFET on
the line as a buffer so the ESP32 never sees the supply's voltage at all.

Worth measuring your own supply before you assume it matches. The ATX spec pulls `PS_ON` up
to `+5VSB`, and 5 V on that pin is a different conversation. Meter the green wire against
any black one with the supply plugged in and the ESP32 disconnected.

## Open drain matters

The control pin is configured **open drain**, and that's the important bit of the config.
An open-drain pin either actively pulls the line to ground or lets go of it entirely. It
never drives it high. The supply has its own pullup and we don't want to fight it — we only
want to pull the line down.

The 2020 sketch did the same thing the long way around, flipping the pin between output-low
to switch on and input to let it float. ESPHome does it with two lines of config.

The nice side effect is that it fails safe. If the ESP32 resets, crashes, gets reflashed, or
loses Wi-Fi, the pin lets go and the supply shuts off. There's no failure mode where the
firmware leaves it stuck on.

## Getting it running

```bash
# Install ESPHome
pipx install esphome            # or: uv tool install esphome

# Set up your secrets
cp secrets.yaml.example secrets.yaml
openssl rand -base64 32         # generates an api_encryption_key
$EDITOR secrets.yaml

# Check the config
esphome config smart-atx.yaml

# Flash it. First one has to be over USB; after that you can go over the air.
esphome run smart-atx.yaml
```

Then add it in Home Assistant under **Settings → Devices & Services**. It announces itself,
and you'll need the `api_encryption_key` from your `secrets.yaml`.

One gotcha: use `esphome run`, not `esphome upload`. `upload` flashes whatever binary was
built last without recompiling, which is a fun way to flash old Wi-Fi credentials onto a
board and then wonder why it won't connect.

## What you get in Home Assistant

A switch called **Power**, which is the supply itself, plus uptime, Wi-Fi signal, free heap,
reset reason, and a restart button tucked into the diagnostics section. The onboard LED
follows the switch, so you can tell what it's doing from across the bench.

Voice control is whatever Home Assistant already gives you — expose the switch to Alexa or
Google through its own integration.

## Using a different board

Built on an **ESP32 DEVKIT V1**. Three values in `smart-atx.yaml` are board-specific and
they're all commented in place: `board`, `status_led_pin`, and `ps_on_pin`.

If you move the control pin, check the strapping-pin table first. It needs to come up
high-impedance at boot or the supply will kick on for a moment before ESPHome takes over.
GPIO18 is fine. GPIO0, 2, 5, 12 and 15 are not.

## What's here

```
smart-atx.yaml          the ESPHome config
secrets.yaml.example    copy to secrets.yaml and fill in your own
openspec/               the requirements the config is built against
```

## Licence

[MIT](LICENSE).
