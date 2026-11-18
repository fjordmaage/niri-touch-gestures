# niri-touch-gestures

Multi-finger **touchscreen** gestures for the [niri](https://github.com/YaLTeR/niri)
compositor — the same gestures you already use on the trackpad, on the glass.

> This is the minimal, pinned version. The full gesture suite (holds,
> pinches, taps, media controls, edge faders, trackpad support) grew into
> its own project: **niri-gesture-suite**.

| Gesture (finger motion) | Action |
|---|---|
| 3-finger swipe left / right | scroll through columns |
| 3-finger swipe up / down | switch workspaces |
| 4-finger swipe up / down | open / close Overview |

In the default **native mode** these are niri's *own* continuous gestures:
the view moves 1:1 with your fingers and, on release, completes or snaps back
with your release velocity — identical physics and identical physical travel
to the trackpad. Normal 1- and 2-finger touch (taps, scrolling, pinch) keeps
working exactly as before.

## Design

Inspired by how Apple approached the touchscreen Mac: don't invent a new
gesture language for the screen — carry the trackpad's over, but anchor it to
**direct manipulation**. On a trackpad you move an invisible viewport; on a
screen your fingers are *on* the content, so the content follows them:

- Swipe **left** → the canvas slides left → you land on the column to the
  **right**. (Same muscle memory as natural-scroll trackpad swipes.)
- Swipe **down** → the workspace stack slides down → you land on the workspace
  **above**.

Rather than re-implementing niri's gesture physics, native mode feeds the
gesture back through the same pipeline the trackpad uses: the daemon mirrors
your 3/4 fingers onto a **virtual touchpad**, libinput recognizes a native
multi-finger swipe, and niri drives its continuous gesture — tracking while
fingers are down, velocity-based completion on release, rubber-banding on
partial swipes. Your niri `input.touchpad` settings (like `natural-scroll`)
apply to it the same way.

The mirror's gain is **auto-calibrated for true direct manipulation**: the
content moves exactly as far as your fingers. niri needs 300 libinput units
for a workspace switch and 1200 for a working-area width of view scroll
(`src/layout/monitor.rs`, `scrolling.rs`), and libinput's unaccelerated
touchpad deltas are `mm × (1000/25.4) × slowdown`, where the slowdown depends
on the accel profile (`0.9 × 0.2968` adaptive; `0.2968 × (1 + accel-speed)`
flat) — the daemon reads your accel profile/speed from the niri config and
solves for the gain that pins the content under your fingertips. The
4-finger Overview zoom keeps trackpad-equivalent travel instead (it is not a
drag; `scale_overview` tunes it).

There is also a **discrete mode** (`mode = "discrete"`) where the daemon
interprets swipes itself and fires configurable `niri msg action …` steps —
useful for arbitrary mappings (e.g. 4-finger horizontal). Thresholds are
physical millimeters: ~28 mm for the first step, ~36 mm per further step,
plus fast-flick detection (≥ 8 mm at ≥ 130 mm/s).

## How it works

A small Python daemon (only dependency: `python-evdev`) grabs the touchscreen
with `EVIOCGRAB` and re-emits everything through a virtual uinput clone that
niri reads instead. 1–2 finger input is forwarded verbatim with no added
latency. The moment a 3rd finger lands, the already-forwarded touches are
cancelled on the clone (so the app under your fingers never sees a stray tap
or scroll); once the fingers travel past a small deadzone, they are replayed
onto the virtual touchpad (native mode) or interpreted into IPC actions
(discrete mode). Contacts are recentered on the oversized virtual pad so they
never start in libinput's edge/palm zones. Multitouch protocol-B slot caches
are kept in sync across swallowed gestures, and screen rotation is picked up
from `niri msg outputs` automatically.

If the daemon stops, the grab is released and the touchscreen falls back to
plain niri handling — it can never leave you without touch.

## Install (Arch)

```bash
sudo pacman -S --needed python-evdev
./install.sh
```

`install.sh` installs the daemon to `~/.local/bin/niri-touch-gestures`, a
config to `~/.config/niri-touch-gestures/config.toml`, and a systemd user
service, then (the one `sudo` step) adds a udev rule that `uaccess`-tags the
touchscreen so the logged-in user can read it — scoped to that one device,
instead of joining the `input` group and gaining access to every keyboard.

## Configuration

See [config.toml](config.toml) — device selection, rotation override, all
thresholds in mm, and the direction → action map. Directions refer to the
motion of your fingers; any `niri msg action …` is accepted, `""` disables a
direction. After editing:

```bash
systemctl --user restart niri-touch-gestures
```

## Try it / debug

```bash
# watch decisions without touching niri (run while the service is stopped):
systemctl --user stop niri-touch-gestures
niri-touch-gestures --dry-run -v
# logs:
journalctl --user -u niri-touch-gestures -f
```

## Troubleshooting

- **"no touchscreen found, retrying"** — the udev rule hasn't been applied
  yet; re-run `install.sh` or `sudo udevadm trigger --subsystem-match=input`.
- **Swipes act rotated** after rotating the screen with a forced transform:
  set `transform` in the config instead of `"auto"`.
- **Steps feel too long/short** — tune `trigger_mm` / `repeat_mm`; they are
  real millimeters, measure against your trackpad habit.
- **Pen input** is a separate device and is not touched.

## Uninstall

```bash
./uninstall.sh
```
