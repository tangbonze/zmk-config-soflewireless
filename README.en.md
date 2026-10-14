# zmk-config-soflewireless

**English** · [简体中文](./README.md)

ZMK firmware configuration for the Sofle split keyboard, with
[DYA Studio](https://studio.dya.cormoran.works/) support.

> This branch (`dya`) adds DYA Studio on top of `main`. The `main` branch keeps
> the plain ZMK official `v0.2` configuration; the two are independent, so pick
> whichever suits you.

## About Sofle

[Sofle](https://github.com/josefadamcik/Sofle) was designed by Josef Adamčík
and named after the Slovak word "Sofle". It is a 6-key-wide split ergonomic
keyboard.

Its defining characteristics:

- **4 rows plus thumb keys, mirrored** — 5 columns × 4 rows + 3 thumb keys per half
- **One EC11 rotary encoder per half** — mappable to volume, page, scroll, etc.
- **Split wireless** — a nice!nano in each half, paired over BLE
- **Hot-swap sockets** — 3×6 MX hot-swap footprint

This configuration targets the standard Sofle build (nice!nano v2 + EC11
encoders + 0.91" OLED or nice!view).

## Firmware variants

Actions → latest build → Artifacts → `zmk-firmware` contains these `.uf2` files:

| Artifact | Description |
| --- | --- |
| `sofle_left` / `sofle_right` | **No display**, most flash headroom — recommended |
| `sofle_left_oled` / `sofle_right_oled` | SSD1306 OLED |
| `sofle_left_niceview` / `sofle_right_niceview` | nice!view adapter |
| `sofle_left_studio_unlocked` / `sofle_right_studio_unlocked` | No display + Studio always unlocked |
| `settings_reset` | Clear all persistent settings (including those saved by DYA Studio) |

Flash the matching file to each half. After `settings_reset`, flash a normal
firmware again.

**To skip the unlock step, flash `*_studio_unlocked`** — these variants carry
`-DCONFIG_ZMK_STUDIO_LOCKING=n`. If Studio still locks afterwards, that cmake
argument did not take effect (the two uf2 files should differ noticeably in
size); check the [Troubleshooting](#troubleshooting) section.

## Left / right assignment

**The central (main) half is the LEFT one** — both USB and BLE live on the left,
and the DYA Studio web UI only connects to the left half.

| | Left · central | Right · peripheral |
| --- | --- | --- |
| **Which firmware** | **`sofle_left*`** | **`sofle_right*`** |
| USB / BLE connection | yes | — |
| Studio communication | yes | — |
| Connection management, OS detection, per-OS layers | yes | — |
| Settings relay | originates | receives and applies |
| Bounce diagnostics, watchdog | aggregates both halves | detects and reports |

This matches ZMK's built-in sofle shield (`app/boards/shields/sofle/Kconfig.defconfig`
has `if SHIELD_SOFLE_LEFT → ZMK_SPLIT_ROLE_CENTRAL default y`).
**Sofle defaults to left-side central.**

> ⚠️ **Flashing them swapped leaves the keyboard with no input at all** (central
> and peripheral won't pair). Remember: **the half with USB plugged in = left =
> flash `sofle_left*`**.

## Layer design

| Layer | How to enter | Contents |
| --- | --- | --- |
| 0 DEFAULT | — | Main key area, **identical key-for-key to the original `main` branch** (except the right 2nd thumb key — see below) |
| 1 LOWER | Left 4th thumb key | F keys / digits / symbols |
| 2 RAISE | Right 2nd thumb key (tap = backspace / hold = enter layer) | Bluetooth profiles / navigation / editing |
| 3 ADJUST | LOWER + RAISE together, or `&mo 3` from RAISE | Media / brightness / Bluetooth / bootloader |

> **The only change from the original `main` branch**: the right 2nd thumb key on
> DEFAULT went from `&kp RCTRL` to `&lt 2 BACKSPACE` (tap backspace / hold RAISE),
> freeing up a layer entry and matching Corne's convention. The other 59 keys are
> untouched. Right Ctrl is not placed elsewhere in this branch; if you rely on it,
> change that key back to `&kp RCTRL` (and find RAISE another entry point).

Thumb cluster (3 keys per side):

```
Left                             Right
GUI   ALT  CTRL                  SPC   RAISE  RCTRL
LWR   ENT                       ALT   RGUI
```

- `DEFAULT → LOWER`: left `&mo 1`
- `DEFAULT → RAISE`: right 2nd key `&lt 2 BACKSPACE` (tap backspace / hold RAISE)
- `RAISE → ADJUST`: left 2nd thumb key `&mo 3`
- `LOWER + RAISE` together → ADJUST (conditional layer)
- The fallback key in LOWER / RAISE / ADJUST is `&mo 0` — hold to return to
  DEFAULT temporarily

### Encoders

One EC11 per half, mapped identically across all four layers:

| | Mapping |
| --- | --- |
| Left encoder | Volume (`C_VOL_UP` / `C_VOL_DN`) |
| Right encoder | Paging (`PG_UP` / `PG_DN`) |

These are handled by `&inc_dec_kp` in `sensor-bindings`, which **depends on
`CONFIG_EC11=y` and `CONFIG_EC11_TRIGGER_GLOBAL_THREAD=y` in
`config/sofle.conf`** — do not remove those two lines, or the sensors stay
present but stop emitting events and the encoders go completely dead.

> `config/sofle.conf` does not set `CONFIG_ZMK_DISPLAY`. The display is selected
> by the snippet in `build.yaml` (`sofle-oled` / `sofle-no-display`).

### External power

The DEFAULT layer keeps the original `main` branch binding `&ext_power EP_TOG`
(external power toggle), on the 4th row just right of the MUTE key. Sofle's
OLED / nice!view and peripherals are related to external power; if your build
has no such feature, just delete that binding.

## DYA Studio

Open <https://studio.dya.cormoran.works/> in a browser and pick a connection
method on the splash page:

| Method | Description | Supported browsers |
| --- | --- | --- |
| USB | Pick a serial port after plugging in (Web Serial) | Chrome / Edge (desktop) |
| Bluetooth | Connect over BLE after pairing (Web Bluetooth) | Chrome / Edge (desktop), Android Chrome, iOS via Bluefy |
| Demo | Try every UI without a keyboard | same as above |

Firefox and Safari are not supported.

### Available features

- **Keymap Editor** — edit keys and layers visually, with live key highlighting
- **Macros & Combos** — create macros and combos in the browser, **no reflash needed**
- **Connection Management** — rename / switch / unbind BLE profiles, set USB vs Bluetooth priority
- **Device Settings** — power management (idle / deep sleep timeouts), per-half settings
- **Troubleshooting** — battery, firmware version, uptime; per-key bounce detection; full support report

> Sofle has encoders but no trackball, so DYA Studio's Trackball Tuning and the
> related modules (PMW3610 driver) do not apply; encoders map to the Input
> Processor panel instead.

### Studio locking

**The default firmware locks Studio**: it locks immediately on disconnect and
re-locks after idle. While locked, the browser's Web Bluetooth **cannot see the
keyboard** — press an unlock key first:

```
Enter RAISE: DEFAULT layer, right 2nd thumb key — tap backspace / hold RAISE
Unlock:     left 1st thumb key on RAISE (GUI position)
Backup:     left 3rd thumb key on ADJUST
```

## Local build

You need the [Zephyr SDK](https://docs.zephyrproject.org/latest/develop/toolchains/index.html)
and [`west`](https://docs.zephyrproject.org/latest/develop/west/install.html).

**Python 3.12** is required (Zephyr 4.1 does not support 3.13/3.14), plus Zephyr
SDK 0.17.0:

```bash
brew install python@3.12
python3.12 -m venv .venv && ./.venv/bin/pip install west
export ZEPHYR_SDK_INSTALL_DIR=~/zephyr-sdk-0.17.0

# dependencies land in ./dependencies
west init -l config
# or into the west topdir (share one ZMK across several config repos)
# west init -l . --mf config/west-workspace.yml

west update --narrow
west zephyr-export
# Studio's nanopb generator needs these, or the build fails while generating protos
./.venv/bin/pip install "protobuf>=5.27" grpcio-tools

west zmk-build                    # build every variant in build.yaml
west zmk-build -a sofle_right    # build just one
west zmk-build -a sofle_left -p always   # pristine rebuild after config edits

# flashing requires a single variant (--flash errors out with multiple targets)
west zmk-build -a sofle_left --flash

# for debugging, build a firmware with USB logging (costs extra flash)
west zmk-build -a sofle_left -S zmk-usb-logging --flash
```

Firmware lands in `./build/<artifact>/zephyr/zmk.uf2`.

## Troubleshooting

A firmware that compiles is not proof that features are enabled. The classic
case: if none of the DYA Studio Kconfig is loaded, the firmware still builds,
still types after flashing, but the Studio protocol stack was never linked in —
symptom: "can't connect to Studio".

**1. Check the Kconfig warnings in the build log — this branch should have none.**
Every `assigned the value 'y' but got 'n'` means some feature did not turn on;
the warning names the symbol:

```bash
grep -iE "warning: (assigned|attempt to assign|choice symbol)" \
  build/sofle_left/stdout_and_stderr.log
```

**2. Confirm which `.conf` files were actually merged.** This is the fastest
way to find "the config never got read" — if you see
`dependencies/zmk/app/boards/shields/sofle/sofle_left.conf` (the empty file in
ZMK) instead of this repo's snippet, the config was never picked up:

```bash
grep -oE "[^ ']*\.conf" build/sofle_left/stdout_and_stderr.log | sort -u
```

A correct run shows `snippets/sofle-central/sofle-central.conf`.

**3. Verify the final `.config`.** All of the following must be `y`; any line
ending in `n` means that feature is off (the most typical one is
`ZMK_BEHAVIOR_LOCAL_ID_TYPE_CRC16`, which must match the DYA Studio web
client — otherwise bindings written by Macros / Combos / Keymap Editor don't
line up with the existing keys, showing up as "changes had no effect"):

```bash
grep -E "ZMK_STUDIO|PHYSICAL_LAYOUTS|FAST_KEYMAP|KSCAN_DIAG|WATCHDOG|OS_DETECTION|DEFAULT_LAYER|BLE_MANAGEMENT|RUNTIME_(MACRO|COMBO|INPUT)|INPUT_STREAM|DEVICE_INFO|LOCAL_ID_TYPE" \
  build/sofle_left/zephyr/.config | grep -v "=n$"
```

**4. Encoders do nothing** — confirm `grep EC11 build/sofle_left/zephyr/.config`
shows `CONFIG_EC11=y`, and that the matching encoder in the overlay has
`status = "okay"` (already the default).

**5. Capture device logs.** Daily firmware disables `CONFIG_CONSOLE` to save
flash; for debugging, build a logged variant with `-S zmk-usb-logging`.

**6. No input at all** — first make sure you didn't flash them swapped: the
half with USB plugged in must be `sofle_left*`.

## Directory layout

```
config/
  sofle.keymap           key definitions (4 layers, with ASCII diagrams)
  sofle.json             layout description for external keymap-editor /
                         keymap-drawer (not compiled; DYA Studio draws the
                          keyboard from the physical-layout built into the ZMK
                          fork, not this file)
  sofle.conf             Kconfig shared by both halves (Bluetooth, battery,
                         sleep, EC11 encoders, …)
  west-dependency.yml    dependency manifest: ZMK fork + DYA Studio module stack
  west.yml               default west entry point
  west-isolated.yml      dependencies into ./dependencies
  west-workspace.yml     dependencies into the west topdir
boards/shields/sofle/
  Kconfig.defconfig      keyboard name + split role + inherited
                         ZMK_SPLIT / display / LVGL defaults
  Kconfig.shield         shield identity
snippets/
  sofle-central/         left (central) only: Studio and all *_STUDIO_RPC,
                         OS detection, default layer, connection management,
                         extension modules
  sofle-peripheral/      right (peripheral) only: receives / applies relayed settings
  sofle-oled/            OLED display
  sofle-no-display/      disables the onboard OLED, reclaims flash
zephyr/module.yml        tells west where to find boards / snippets
build.yaml               build matrix
```

> `sofle.dtsi`, `sofle_left.overlay`, `sofle_right.overlay` and the physical
> layout (`josefadamcik_sofle_layout`) all come from the `main+dya` branch of
> [cormoran/zmk](https://github.com/cormoran/zmk); this repo does not redefine them.

### Splitting config per half: why snippets instead of `<shield>.conf`

DYA Studio's Kconfig must be isolated per half (the protobuf stub code that
`*_STUDIO_RPC` depends on is only generated for central; enabling it on the
right half fails to build for lack of `proto/zmk/custom.pb.h`). This repo does
that isolation through the `snippets:` list of each variant in `build.yaml`:

| File | Scope | Contents |
| --- | --- | --- |
| `Kconfig.defconfig` | shield | keyboard name, split role, inherited `ZMK_SPLIT` / display / LVGL |
| `snippets/sofle-central` | left only | Studio / RPC / OS detection / default layer / extension modules |
| `snippets/sofle-peripheral` | right only | receives and applies relayed settings only |
| `config/sofle.conf` | both halves | Bluetooth, battery, sleep, EC11 encoders, split event relay, thread stacks |

**The trap: putting config in `boards/shields/sofle/sofle_left.conf` does not
work.** Zephyr's `SHIELD_ROOT` only resolves to the **first** board root
containing that shield, and ZMK's `app/boards` comes first, so the build log
shows

```
dependencies/zmk/app/boards/shields/sofle/sofle_left.conf   ← the empty file in ZMK
```

instead of yours. `Kconfig.defconfig` works because Zephyr `osource`s **all**
board roots. So there are only two ways to isolate config per half:

1. `Kconfig.defconfig` + `default y` (works, but a symbol whose `depends on`
   isn't satisfied is **silently dropped with no warning** — hence the classic
   "it compiled but the feature is off");
2. **snippets** (this repo's approach, via `EXTRA_CONF_FILE`). It's an explicit
   assignment, so unmet dependencies produce a clear warning, and it isolates
   per half naturally.

## Dependency source

This branch does not use ZMK official `main` / `v0.3` but the `main+dya` branch
of [cormoran/zmk](https://github.com/cormoran/zmk) — ZMK main plus the custom
Studio RPC protocol and split relay support DYA Studio needs. **DYA Studio's
extended features depend on this fork**; reverting to official ZMK keeps only
the basic keymap editor.

> ⚠️ cormoran's ZMK fork is an experimental branch tuned for DYA keyboards and
> may contain unstable changes. Fine for personal use, but not recommended where
> high stability matters. The repo runs a weekly build to catch breakage from
> upstream changes.
>
> By contrast, the `main` branch uses official ZMK `v0.2` — more conservative
> and stable, but without DYA Studio.

## License

MIT