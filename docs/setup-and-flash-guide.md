# Setting up ESPHome and flashing the Smart ATX — a step-by-step log

A working record of getting from "nothing installed" to "firmware on the board," with
the real commands, the real output, and the two things that went wrong on the way.

Every command below was actually run. Timings and version numbers are from that run, not
estimates. Raw transcripts are in [`logs/`](logs/).

**Environment this was done on:**

| | |
|---|---|
| OS | EndeavourOS (Arch-based) |
| Python | 3.14.7 (system) |
| ESPHome | 2026.8.2 |
| Board | ESP32 DOIT DevKit V1 (inferred — see [archaeology](old-build-archaeology.md)) |

---

## First: why not just use the Home Assistant add-on?

If you already run Home Assistant, the ESPHome Device Builder add-on is the obvious home
for this — it compiles, stores secrets, does OTA, and shows logs. It's a good tool.

The snag is the **first** flash, which has to happen over USB.

The add-on's "plug this into your computer" option flashes via **Web Serial**, a browser
API that is only exposed in a *secure context* — HTTPS, or `localhost`. A Home Assistant
instance reached over plain `http://homeassistant.local:8123` is neither, so the browser
refuses to hand over the serial port and the option is unavailable.

Nothing is broken. The browser is behaving exactly as specified. But it leaves you three
ways out:

1. **Put ESPHome on the machine with the USB cable.** One command compiles and flashes.
   This is what the rest of this guide does.
2. **Compile in the add-on, flash in the browser.** Use the add-on's *Download firmware
   binary* to get a factory `.bin`, then flash it at <https://web.esphome.io/>, which
   *is* served over HTTPS. Needs Chrome or Edge on desktop.
3. **Give Home Assistant a real HTTPS certificate.** Correct, and more work than this
   project justifies on its own.

Route 1 also keeps the YAML in this git repo rather than inside Home Assistant's config
directory, which matters when the repo is the thing you're publishing.

---

## Step 1 — Check your prerequisites

### Python

ESPHome 2026.8.2 declares `requires-python = ">=3.12,<3.15"`. Check what you have:

```console
$ python3 --version
Python 3.14.7
```

This version range matters more than it looks — see the pitfall in step 2.

### Serial port permissions

On Linux, your user needs permission to open the USB serial device. The group name
differs by distro:

| Distro | Group |
|---|---|
| Arch / EndeavourOS | `uucp` |
| Debian / Ubuntu / Raspberry Pi OS | `dialout` |

Check which groups you're in:

```console
$ id -nG
miswired sys tty docker rfkill uucp wheel plugdev dialout
```

Both `uucp` and `dialout` are present here, so nothing to do. If yours isn't listed:

```bash
sudo usermod -aG uucp $USER     # or dialout, per the table above
```

**You must log out and back in** for that to take effect — a new terminal is not enough,
group membership is established at login. Skipping this produces a
`Permission denied: '/dev/ttyUSB0'` later, at the worst moment.

### The USB-to-serial chip

Most ESP32 devkits use a CP2102 or a CH340. Both have in-tree Linux drivers, so no driver
install is needed. You can confirm the board enumerates once it's plugged in:

```bash
ls -la /dev/ttyUSB* /dev/ttyACM*
dmesg | tail -20
```

---

## Step 2 — Install ESPHome

Three routes. They are not equivalent.

| Route | Command | Gets you |
|---|---|---|
| Distro package | `sudo pacman -S esphome` | 2026.7.4 — whatever Arch has packaged |
| pipx | `pipx install esphome` | current release, isolated |
| **uv** | `uv tool install esphome` | current release, isolated, fastest |

I used **uv**: it was already installed, it needs no root, and it finished in five
seconds. `pipx` is the better instruction to hand a stranger, since more people have it —
the two are interchangeable here.

The distro package is a version behind and pins you to whatever Arch ships. Fine if you
don't care; avoid if you want to match a config someone else validated.

```console
$ uv tool install esphome
...
Installed 1 executable: esphome
# elapsed: 5s
```

### Pitfall: check what version you actually got

```console
$ esphome version
Version: 2026.6.5
```

**That is not the current release.** It's two months old.

The cause: `uv tool install` built the tool's virtualenv on **Python 3.11.15** — an older
interpreter it had available — and 2026.6.5 is the newest ESPHome that still supports
3.11. There was no error and no warning. uv resolved correctly for the interpreter it
chose; it just chose an interpreter that quietly capped the version.

Pin the interpreter explicitly:

```console
$ uv tool install --force --python 3.14 esphome
...
Installed 1 executable: esphome

$ esphome version
Version: 2026.8.2
```

Confirm what you're running:

```console
$ ~/.local/share/uv/tools/esphome/bin/python -c "import importlib.metadata as m; print(m.version('esphome'), m.metadata('esphome').get('Requires-Python'))"
2026.8.2 <3.15,>=3.12.0
```

**The general lesson:** after installing any Python CLI into a managed environment, check
the version you got rather than the version you expected. A silently-old install is far
more annoying to debug three steps later, when a config key "doesn't exist."

### Confirm it's on your PATH

```console
$ which esphome
/home/miswired/.local/bin/esphome
```

If that comes back empty, `~/.local/bin` isn't on your `PATH`. `uv tool update-shell`
adds it, or add it to your shell rc file by hand.

---

## Step 3 — Create your secrets file

```bash
cp secrets.yaml.example secrets.yaml
```

Fill in your Wi-Fi credentials, a fallback AP password, an OTA password, and an API
encryption key. Generate the key with:

```bash
openssl rand -base64 32
```

`secrets.yaml` is listed in `.gitignore`. Confirm that's actually working before you go
any further — this is cheap insurance against publishing your Wi-Fi password:

```console
$ git check-ignore -v secrets.yaml
.gitignore:2:secrets.yaml	secrets.yaml
```

No output from that command means the file is **not** ignored. Stop and fix it.

---

## Step 4 — Validate the config before you build it

```bash
esphome config smart-atx.yaml
```

This resolves every substitution and every default, then prints the fully expanded
configuration. It catches typos and invalid keys in about a second, without waiting on a
compile.

It's also the way to check that a pin means what you think it means. The control pin
should come out like this:

```yaml
switch:
  - platform: gpio
    id: atx_psu
    name: Power
    pin:
      number: 18
      mode:
        output: true
        open_drain: true
        input: false
        pullup: false
        pulldown: false
      inverted: true
```

`open_drain: true` with `pullup: false` is the safety-critical part: the pin can pull
`PS_ON` to ground or let go of it, and can never drive it high. Worth confirming with
your own eyes rather than trusting that the YAML meant what you intended.

The last line should read:

```
INFO Configuration is valid!
```

---

## Step 5 — Compile

```bash
esphome compile smart-atx.yaml
```

**The first compile is slow and large.** ESPHome 2026.8 uses its own ESP-IDF integration
rather than PlatformIO (the `platformio` toolchain is deprecated and slated for removal in
2027.2.0), so the first run downloads and builds ESP-IDF 5.5.5 into `~/.cache/esphome/`.

Measured on this machine:

| | |
|---|---|
| First compile, cold | **146 s** |
| Incremental rebuild, no changes | **4 s** |
| `~/.cache/esphome` after | **1.7 GB** |
| `.esphome/` in the project | **236 MB** |

Budget the disk space before you start — 2 GB in total, most of it a shared toolchain
cache that later projects reuse.

> If you have an old `~/.platformio` from previous ESP work, current ESPHome no longer
> touches it. On this machine it was 11 GB of reclaimable space.

The build ends with a size report:

```
Total image size: 883559 bytes (.bin may be padded larger)
RAM:   [===       ]  25.3% (used 45744 bytes from 180736 bytes)
Flash: [=====     ]  48.2% (used 883559 bytes from 1835008 bytes)
INFO Creating factory.bin...
INFO Created: .esphome/build/smart-atx/build/firmware.factory.bin
INFO Created: .esphome/build/smart-atx/build/firmware.ota.bin
INFO Created: .esphome/build/smart-atx/build/firmware.elf
INFO Successfully compiled program.
```

Roughly half the flash for a config this small is normal — most of it is the Wi-Fi stack
and the ESPHome API, not your YAML. Adding a few more sensors barely moves it.

### Which binary is which

The build produces three, and picking the wrong one is a common way to waste an evening:

| File | Size | Use it for |
|---|---|---|
| `firmware.factory.bin` | 928 KB | **First flash.** Bootloader + partition table + app, written at offset 0. This is the one web.esphome.io wants. |
| `firmware.ota.bin` | 864 KB | Over-the-air updates to a device already running ESPHome. App only. |
| `firmware.elf` | — | Debug symbols. For decoding a stack trace, not for flashing. |

If you're taking the browser route from step 0, `firmware.factory.bin` is your file. The
add-on's **Download firmware binary** button hands you the same thing.


---

## Step 6 — Flash over USB

> ### Before you plug anything in
>
> **Unplug the ATX supply from the wall.** On most ESP32 devkits, `VIN` and USB `VBUS`
> meet at the regulator input with no isolation diode. With the PSU live, connecting USB
> ties your computer's 5 V rail to the supply's `+5VSB` rail. Unplug the supply, or lift
> the `Vin` lead, every time.

```bash
esphome run smart-atx.yaml
```

`run` compiles, offers a list of serial ports, flashes, and then drops straight into the
serial log viewer. On a device that's already on the network it offers OTA instead of
USB — the first flash has to be USB because there's no firmware to update yet.

*(Pending — requires the hardware. See tasks 3.1–3.5.)*

---

## Step 7 — Adopt it in Home Assistant

ESPHome devices announce themselves over mDNS. In Home Assistant, **Settings → Devices &
Services** should show a discovered ESPHome device. Accept it and paste the API
encryption key from your `secrets.yaml`.

*(Pending — requires the hardware.)*

---

## Screenshots to capture

Terminal steps are transcribed above, which beats a screenshot for anything you might
want to copy and paste. These are the moments that genuinely need an image:

| # | Shot | Why it earns its place |
|---|---|---|
| 1 | Home Assistant add-on install page, USB option greyed out | Visual proof of the HTTP/Web Serial problem — the thing that sends people down this path |
| 2 | `esphome run` port-selection prompt | The one interactive moment in an otherwise non-interactive flow |
| 3 | Serial log after a successful boot, showing the WiFi connect and API lines | Proof it worked; also the reference for what "healthy" looks like |
| 4 | Home Assistant discovered-device card, before adoption | The payoff moment |
| 5 | The device page in HA: Power switch + the diagnostic entities | The finished result — this is the article's hero UI shot |
| 6 | web.esphome.io with the board connected | Only if you document the browser route |
| 7 | Bench: meter on the green wire reading the `PS_ON` idle voltage | Evidence for the 5 V caveat, and hard to describe in words |

---

## Raw transcripts

- [`logs/01-install-esphome.log`](logs/01-install-esphome.log)
- [`logs/02-first-compile.log`](logs/02-first-compile.log)
