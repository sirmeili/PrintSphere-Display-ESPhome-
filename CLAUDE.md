# CLAUDE.md

Guidance for Claude Code when working in this repository.

## What this is

PrintSphere-Display: an ESPHome firmware config for a round 466×466 AMOLED ESP32-S3 board
(Waveshare ESP32-S3-Touch-AMOLED-1.43, CO5300 panel, FT63x6-compatible touch) that shows Bambu Lab
printer status pulled from Home Assistant (via the `ha-bambulab` integration). The UI is LVGL:
a full-screen progress ring plus two swipeable pages (status/temps, and a round print-preview image).

The full behavioural spec, derived from the current code, is in [docs/spec.md](docs/spec.md).
Read it before changing status logic or UI layout.

## Repository layout

- `esphome-project.yaml` — the entire firmware. All logic lives in C++ lambdas inside this file.
- `example-device.yaml` — template device file: overrides substitutions and `!include`s the main YAML.
- `printsphere-*.yaml` — personal device files (gitignored). These are what gets flashed.
- `README.md` — upstream user-facing docs (fork of `cptkirki/PrintSphere-Display-ESPhome-`).
- `assets/` — README screenshots and upstream's 86×86 `bambuicon.png` (not used by the build).
- `docs/spec.md` — functional + technical spec.

- `images/` — `bambuicon-small.png` (86×86 page-1 logo) and `bambuicon.png` (384×384 preview placeholder).
- `fonts/materialdesignicons-webfont.ttf` — icon font (nozzle, bed, clock glyphs).

Not in the repo (must exist in the ESPHome config dir at build time): `secrets.yaml` with `api_encrypt`,
`wifi_ssid`, `wifi_password` (OTA currently has no password).

## Building / flashing

No build tooling is in the repo and the `esphome` CLI is not installed on this machine. Typical flow:

```sh
pip install esphome            # or use the HA ESPHome add-on / dashboard
esphome config printsphere-office.yaml    # validate YAML + substitutions
esphome compile printsphere-office.yaml
esphome run printsphere-office.yaml       # flash via USB or OTA
esphome logs printsphere-office.yaml
```

Declares ESPHome >= 2025.9.0, but the lambdas use the LVGL 9 API, so in practice it needs an ESPHome
release that bundles LVGL 9. Framework is ESP-IDF;
PSRAM octal @ 80 MHz is required (LVGL buffers + 420×420 RGB565 preview ≈ 350 KB).

`esphome-project.yaml` is a **generic template**: its substitutions (`device_name`, `friendly_name`,
`homeassistant_url`, `bambulab_printer`, `bambulab_icon`) hold placeholder defaults. Personal values go
in a gitignored `printsphere-*.yaml` device file (copy `example-device.yaml`), which overrides them and
pulls in the main file via `packages: !include`. Never put personal values (printer serial, real device
name) or credentials in tracked files; credentials come from `secrets.yaml` via `!secret`. This repo is a
public GitHub fork.

There are no automated tests. "Verification" means `esphome config` / `compile` succeeding, then
checking behaviour on-device against the state tables in `docs/spec.md`.

## Architecture in one screen

- **HA inputs** (`sensor`, `text_sensor`, `binary_sensor`, all `platform: homeassistant`, `internal: true`):
  progress, nozzle/bed temp, remaining time, current/total layer, current_stage, print_status,
  hms_errors, druckfehler, and the `entity_picture` attribute of `image.<printer>_titelbild`.
- **UI construction**: widgets are declared flat under `lvgl.widgets`, then a one-shot lambda in the
  **300 ms interval** (guarded by `ui_built`) creates a horizontal snap-scrolling pager with two pages
  in raw LVGL C and re-parents the declared widgets into it. Ring (`arc_main`) and `lbl_pct` are moved
  to `lv_layer_top()` so they stay fixed while swiping.
- **50 ms interval**: ring colour/animation state machine (pulses, colour blend, filament fill animation).
  Throttled to 300 ms while `is_scrolling`.
- **1 s interval**: data → labels (status text + colour, %, layer, temps, remaining time), arc value,
  `print_active` latch, and `ui_col_target` (target colour for the printing blend).
- **Preview image**: `online_image` `ha_preview` URL is set from the `entity_picture` attribute on change;
  `on_download_finished` applies a "cover" zoom into a 420 px circular clip mask.

The state-priority logic (filament > error > done > idle-ish > stage > printing) is **duplicated** across
the 50 ms ring lambda, the status-text lambda, the status-colour lambda, the remaining-time lambda and the
target-colour lambda. When changing status logic, update every copy (or, better, factor it into one
shared helper / global — see spec "Known issues").

## Conventions & gotchas

- Code comments are a mix of German and English (upstream author is German). Entity IDs are English
  (`_print_error`, `_pick_image`); the ESPHome id `druckfehler` (= print error) is a leftover name.
- All string matching on HA states is lower-cased and mostly substring (`find`) based.
- Raw LVGL calls are used via `using namespace esphome::lvgl;`, targeting **LVGL 9**: use
  `lv_obj_remove_flag` with one flag per call (OR-ing enum flags doesn't compile in C++) and cast
  `lv_event_get_target(e)` to `lv_obj_t *`. Some v8 compat names (`lv_img_set_zoom`, `lv_disp_*`,
  `lv_scr_act`) remain; prefer v9 names (`lv_image_set_scale`, `lv_display_*`, `lv_screen_active`) in new code.
- Fonts only contain the glyphs listed in each `font:` entry. Adding new label text with characters
  outside that set (e.g. `(`, `,`, `ß`, `!`) renders blank — extend `glyphs` too.
- Display is unrotated with `offset_width: 6` (panel column offset); touch has no transform. Hard-coded
  geometry assumes 466×466; the 1.75" variant lives on the upstream `Amoled-1.75` branch.

## Owner's roadmap (context for future work)

1. Verify correctness on hardware / fix bugs (see spec "Known issues").
2. Personal enhancements.
3. Possible rewrite (e.g. move lambdas into a C++ header / external component, single status model).
4. Sleep: screen off / low-power mode (no brightness control, dimming, or power management exists yet;
   Wi-Fi `power_save_mode: none`).
