# PrintSphere-Display — Specification

> Derived from `esphome-project.yaml` (owner's updated version, replacing upstream commit `2e9c0e8`).
> This file describes what the firmware **does today**, including its quirks. Section 10 lists issues found while writing it; they are
> observations, not yet confirmed on hardware.

---

## 1. Purpose

A desk-side status display for a Bambu Lab 3D printer. It does not talk to the printer directly: it
subscribes to Home Assistant entities created by the [`ha-bambulab`](https://github.com/greghesp/ha-bambulab)
integration through the ESPHome native API, and renders them on a round AMOLED screen as:

- a full-circle **progress ring** whose colour/animation encodes the printer stage, and
- two horizontally swipeable **pages**: (1) status text, temperatures, layer and remaining time;
  (2) a circular preview of the current print's cover image.

---

## 2. Hardware

| Item | Value |
|---|---|
| Board | Waveshare ESP32-S3-Touch-AMOLED-1.43 (or equivalent) |
| SoC | ESP32-S3, ESP-IDF framework |
| PSRAM | Octal, 80 MHz (required) |
| Display | CO5300 AMOLED, 466×466, QSPI via `mipi_spi` |
| Display bus | SPI3 quad: CLK GPIO10, D0–D3 GPIO11–14, CS GPIO9, RESET GPIO21, 80 MHz, MODE0 |
| Display settings | no rotation, `offset_width: 6` / `offset_height: 0`, `brightness: 248` (fixed), `invert_colors: false` |
| Touch | FT63x6-compatible on I²C `0x38`, SDA GPIO47, SCL GPIO48, 400 kHz, polled every 30 ms, no transform (no mirroring/swap) |
| Enclosure | 3D-printable case (MakerWorld model 1909685) |

---

## 3. Software dependencies

- ESPHome **≥ 2025.9.0** (`min_version`), components: `api`, `ota`, `wifi`, `logger`, `time`,
  `http_request`, `image`, `online_image`, `spi`, `i2c`, `touchscreen`, `display` (`mipi_spi`), `lvgl`, `font`.
- LVGL as bundled by ESPHome, 16-bit colour depth. Lambdas target the **LVGL 9** API (`lv_obj_remove_flag`
  with one flag per call, `(lv_obj_t *)` cast on `lv_event_get_target`) while still using v8-compat names
  (`lv_img_set_zoom`, `lv_disp_get_default`, `lv_disp_set_theme`, `lv_scr_act`).
- Home Assistant with the `ha-bambulab` integration.
- Google Fonts `Dosis` (fetched at build time) and a local `fonts/materialdesignicons-webfont.ttf`.

---

## 4. Configuration

### 4.1 Substitutions

`esphome-project.yaml` holds generic defaults. A per-device file (gitignored `printsphere-*.yaml`,
template `example-device.yaml`) overrides them and includes the main file as a package.

| Key | Meaning | Default |
|---|---|---|
| `device_name` | ESPHome node name | `printsphere` |
| `friendly_name` | Friendly name in HA | `PrintSphere-Display` |
| `homeassistant_url` | Base URL of HA reachable from the device; prefixed to relative image URLs | `http://homeassistant.local:8123` |
| `bambulab_printer` | Entity-ID slug of the printer in HA | `your_printer_id` (placeholder) |
| `bambulab_icon` | Path to the page-1 logo image | `images/bambuicon-small.png` |

### 4.2 Secrets (`secrets.yaml`, not in repo)

`api_encrypt`, `wifi_ssid`, `wifi_password`. OTA has **no password**.

### 4.3 Device identity & network

- Node name `${device_name}`, friendly name `${friendly_name}`, no MAC suffix.
- Wi-Fi `fast_connect: true`, `power_save_mode: none`, reboot after 10 min without connection.
- HTTP client: 10 s timeout, **SSL verification disabled**.

### 4.4 Build-time files

- `${bambulab_icon}` → `img_bambu_logo` (RGB565), shown on page 1 at native size (~86×86 expected).
- `images/bambuicon.png` → `img_placeholder` (RGB565), used as the preview placeholder. (Hard-coded
  path; the comment intends a neutral black/transparent image, but it is the same Bambu icon.)

---

## 5. Inputs (Home Assistant entities)

All are `platform: homeassistant` and `internal: true` (not re-exposed to HA). `<p>` = `${bambulab_printer}`.

| ESPHome id | Type | HA entity | Used for |
|---|---|---|---|
| `print_progress` | sensor | `sensor.<p>_print_progress` | Ring value, `%` label, state logic |
| `nozzle_temp` | sensor | `sensor.<p>_nozzle_temperature` | Nozzle label |
| `bed_temp` | sensor | `sensor.<p>_bed_temperature` | Bed label |
| `remaining_time` | sensor | `sensor.<p>_remaining_time` | Remaining label (interpreted as **minutes**) |
| `current_layer` | sensor | `sensor.<p>_current_layer` | Layer label |
| `total_layer` | sensor | `sensor.<p>_total_layer_count` | Layer label |
| `current_stage` | text_sensor | `sensor.<p>_current_stage` | Primary state input |
| `print_status` | text_sensor | `sensor.<p>_print_status` | Secondary state input |
| `ha_img_url` | text_sensor | `image.<p>_pick_image`, attribute `entity_picture` | Preview image URL |
| `hms_errors` | binary_sensor | `binary_sensor.<p>_hms_errors` | Error state |
| `druckfehler` | binary_sensor | `binary_sensor.<p>_print_error` | Error state |

Notes:
- Entity IDs are the English ones. Upstream used German-localised `_titelbild` (cover image) and
  `_druckfehler`; the ESPHome id `druckfehler` was kept. The preview uses `_pick_image`, not `_cover_image`.
- Text inputs are lower-cased before matching; most matches are substring (`find`) matches.

---

## 6. Internal state (globals)

| Global | Type | Init | Purpose |
|---|---|---|---|
| `ui_built` | bool | false | One-shot guard for UI construction |
| `ui_col_now` | uint32 | `0x00FF00` | Current blended ring indicator colour |
| `ui_col_target` | uint32 | `0x00FF00` | Target colour for the blend |
| `last_prog` | int | 0 | Last known progress (fallback when sensor has no state) |
| `print_active` | bool | false | Latched "a job is in progress" flag |
| `is_scrolling` | bool | false | True while the pager is being swiped (throttles ring updates) |

None are restored across reboots.

### 6.1 Derived predicates (recomputed in several lambdas)

Let `stage` = lower(current_stage), `ps` = lower(print_status), `prog` = clamp(progress, 0, 100).

| Predicate | Definition |
|---|---|
| `filament_loading` | `stage` contains `filament_loading` |
| `filament_unloading` | `stage` contains `filament_unloading` |
| `error` | `ps == "failed"` or `ps` contains `fail`, **or** `hms_errors` on, **or** `druckfehler` on |
| `done_strict` | `ps` contains `finish` **and** `stage` contains `idle` **and** `prog == 100` |
| `idleish` | `stage` contains `idle` or `offline` |
| `preheat` | `stage` contains `heatbed_preheating`, `nozzle_preheating` or `preheat` |
| `clean` | `stage` contains `cleaning_nozzle_tip` or `clean` |
| `bed_level` | `stage` contains `auto_bed_leveling` or `bed_level` |
| `cooling` | `stage` contains `cooling_down` or `cool` |

`prog` fallback when the sensor has no state: `last_prog` in the 50 ms ring loop, target-colour and
`print_active` lambdas; **0** in the status text/colour, remaining-time and arc-value lambdas.

### 6.2 `print_active` latch (1 s)

- Cleared when `error-by-status` (`ps` failed/fail — HMS/druckfehler not considered) or `done_strict`.
- Set when `ps` is `running` or `prepare` or contains `print`, or `0 < prog < 100`.
- Otherwise unchanged.

`last_prog` is updated every second whenever `print_progress` has a state.

---

## 7. User interface

### 7.1 Structure

Built once by the 300 ms interval lambda when an LVGL display exists:

```
screen (black, theme removed)
├── pager 466×466, horizontal scroll, snap-to-center, scrollbar off, transparent
│   ├── page1 (x = 0)      status, logo, temps, layer, remaining
│   └── page2 (x = 466)    page2_note, img_mask(420 circle) → img_preview
└── lv_layer_top()
    ├── arc_main          progress ring (never scrolls, not clickable)
    └── lbl_pct           percentage
```

Starts on page 1. `backdrop` (466×466 transparent obj) is declared but not used.

### 7.2 Page 1 layout (offsets from centre, px)

| Widget | Pos | Font | Default colour | Initial text |
|---|---|---|---|---|
| `lbl_status` | (0, −86) | Dosis 32 | dynamic (§8.2) | `Waiting for Data...` |
| `img_logo_small` | (0, −7) | — | — | logo, zoom 1.0, sent to background |
| `lbl_nozzle_icon` | (−182, −10) | MDI 40 (U+F0E5B) | white | |
| `lbl_nozzle` | (−132, −10) | Dosis 32 | white | ` --°C` |
| `lbl_bed_icon` | (110, −10) | MDI 40 (U+F185B) | white | |
| `lbl_bed` | (160, −10) | Dosis 32 | white | ` --°C` |
| `lbl_layer` | (0, 70) | Dosis 32 | `0xDDDDDD` | `Layer: -- / --` |
| `row_remaining` (flex row, width 280) | (0, 170) | | | |
| ├ `lbl_remaining_icon` | | MDI 40 (U+F144E clock) | `0x87CEEB` | |
| └ `lbl_remaining` | | Dosis 40 | `0x87CEEB` | ` --m` |

### 7.3 Overlay

- `arc_main`: 466×466, 0–100, `rotation: 270` (0 % at 12 o'clock, clockwise), 360° background,
  arc width 24, rounded caps; track `0x101010`, indicator initially `0x00FF00`.
- `lbl_pct`: Dosis 40, white, at (0, −187) — just inside the top of the ring.

### 7.4 Page 2

- `page2_note`: `Preview disabled`, Dosis 20, `0x888888`, centred. Hidden after the first
  successful image download and never shown again.
- `img_mask`: 420×420, radius 210, `clip_corner` → circular clip, not scrollable/clickable.
- `img_preview`: `online_image` `ha_preview` inside the mask, centred.

### 7.5 Swipe behaviour

- Pager snaps to page centres.
- On `SCROLL_BEGIN`, `is_scrolling = true`. While scrolling, page-1 widgets are hidden when
  `scroll_x > 0.6·W` and shown when `scroll_x < 0.4·W` (hysteresis), to reduce redraw cost.
- On `SCROLL_END` / `RELEASED`, `is_scrolling = false` and page-1 widgets are visible iff `scroll_x < W/2`.
- While `is_scrolling`, the 50 ms ring loop runs at most every 300 ms.

### 7.6 Fonts

Dosis 20/32/40 contain only: `0-9 : % a-z äöü A-Z ÄÖÜ space ° - . / _`. MDI 40 contains the three icons
above. Any other character renders as nothing.

---

## 8. Behaviour

### 8.1 Ring (50 ms loop) — first matching rule wins

| # | Condition | Track | Indicator | Animation |
|---|---|---|---|---|
| 1 | filament loading | `0x101010` | `0xFFD54F` yellow | value sweeps 0→100 over 2 s, holds 0.3 s, repeats |
| 1 | filament unloading | `0x101010` | `0xFFD54F` | value sweeps 100→0 over 2 s, holds 0.3 s |
| 2 | `error` | pulse | pulse | sine pulse `0x220000`↔`0xFF3333`, 1.6 s |
| 3 | `done_strict` | `0x00FFFF` | `0x00FFFF` | solid cyan |
| 4a | `idleish` and `ps == prepare` | pulse | pulse | `0x3A1F00`↔`0xFFA500`, 1.4 s |
| 4b | `idleish` and (`print_active` or `0<prog<100`) | `0x101010` | blend | blend toward `ui_col_target` |
| 4c | `idleish` otherwise | `0x404040` | `0x404040` | solid grey |
| 5 | `preheat` | pulse | pulse | `0x3A1F00`↔`0xFFA500`, 1.4 s |
| 5 | `clean` | pulse | pulse | `0x404040`↔`0xFFFFFF`, 1.2 s |
| 5 | `bed_level` | pulse | pulse | `0x220033`↔`0xAA66FF`, 1.4 s |
| 6 | anything else (printing, cooling, …) | `0x101010` | blend | blend toward `ui_col_target` |

"Pulse" colours both track and indicator, so the ring appears as a full pulsing circle.
"Blend" moves `ui_col_now` toward `ui_col_target` by up to 25 per RGB channel per tick; `ui_col_now`
only advances while a blend rule is active.

**Arc value** (1 s): `prog` (raw progress clamped, else 0), except while filament loading/unloading,
when the animated value is left alone.

### 8.2 Blend target colour (1 s)

| Condition (first match) | `ui_col_target` |
|---|---|
| `done_strict` | `0x00BFFF` |
| stage contains `heatbed` / `preheat` / `nozzle_preheating` | `0xFF3333` |
| stage contains `clean` | `0xFFFFFF` |
| `bed_level` | `0xFFA500` |
| stage contains `cool` | `0x3399FF` |
| `idleish` | `0x00FF00` if `0<prog<100` or `ps ∈ {running, prepare}`, else `0x666666` |
| default | `0x00FF00` green |

Because rules 2–5 of §8.1 return before blending, in practice the visible blend colours are: green
(printing / unknown stages), blue `0x3399FF` (cooling), and green/grey for idle-ish-but-active.

### 8.3 Status text (1 s) — first match wins

1. filament loading → `loading`; unloading → `unloading`
2. `error` → `failed`
3. `done_strict` → `done`
4. `idleish`: `ps == prepare` → `preparing`; `prog > 0` → `printing`; else `idle`
5. Stage mapping: contains `printing` → `printing`; bed level → `bed level`; preheat → `preheating`;
   clean → `clean nozzle`; cool → `cooling`
6. Status mapping: `prepare` → `preparing`; `running` → `printing`; `finish`/`finished` → `printing`;
   `failed` → `failed`; contains `pause` → `paused`; contains `cool` → `cooling`
7. Fallback: stage title-cased (`_`/`-` → space), truncated to 18 chars with `...`; else `ps` likewise;
   else `waiting...`

### 8.4 Status text colour (1 s)

| Condition (first match) | Colour |
|---|---|
| filament | `0xFFD54F` yellow |
| `error` | `0xFF3333` red |
| `done_strict` | `0x00BFFF` blue |
| `idleish` + prepare | `0xFFA500` orange |
| `idleish` + `prog > 0` | `0xFFFFFF` white |
| `idleish` otherwise | `0xAAAAAA` grey |
| preheat / clean / bed level | `0xFFA500` orange |
| cooling | `0x87CEEB` sky blue |
| otherwise | white |

### 8.5 Other labels (1 s)

| Label | Rule |
|---|---|
| `lbl_pct` | `"<n>%"` from raw progress (not clamped), else `--%` |
| `lbl_remaining` | `Done` if `done_strict`; else remaining minutes as `Hh Mm` (≥ 1 h) or `Mm`; else `--m` |
| `lbl_layer` | `layer: <cur> / <tot>` (cur capped at tot), `--` for missing values |
| `lbl_nozzle`, `lbl_bed` | ` <n>°C` (truncated to int), else ` --°C` |

Text colours of these labels and the `row_remaining` width/alignment are re-applied every second.

### 8.6 Preview image

1. When `image.<p>_pick_image`'s `entity_picture` attribute changes, the URL is set on `ha_preview`;
   relative URLs (no `http://`/`https://`) are prefixed with `homeassistant_url`. A download is triggered.
2. Download: PNG only, resized to fit 420×420, RGB565, headers `Accept: image/png,...`,
   `Accept-Encoding: identity`. Placeholder `img_placeholder` until loaded.
3. On finish: widget source updated; after 50 ms, `page2_note` hidden, zoom reset, then a "cover"
   zoom computed so the image fills the 420 px circle (`zoom = max(ceil(420·256/w), ceil(420·256/h))`,
   clamped 16–1024), centred, invalidated.

No periodic refresh exists; downloads happen only on attribute change.

---

## 9. Timing summary

| Task | Period | Work |
|---|---|---|
| Touch poll | 30 ms | FT63x6 read |
| Ring loop | 50 ms (300 ms while swiping) | §8.1 |
| UI build | 300 ms | one-shot, then early-return forever |
| Data/labels | 1 s | §6.2, §8.1 arc value, §8.2–§8.5 |
| Preview | on change | §8.6 |

There is no sleep, dimming, screen-off, or idle-timeout behaviour. Brightness is fixed at 248.

---

## 10. Known issues & observations

Found by reading the code; each needs confirming on hardware before fixing.

**Build / setup**
1. ~~Substitutions empty~~ — fixed: generic defaults in the main file, personal values in a
   gitignored device file.
2. ~~Missing build assets~~ — fixed: `images/bambuicon-small.png` (86×86), `images/bambuicon.png`
   (384×384) and `fonts/materialdesignicons-webfont.ttf` (contains U+F0E5B, U+F185B, U+F144E) are in the repo.
   The placeholder is still the Bambu icon, not a neutral image, and at 384 px it isn't scaled to the
   420 px circle.
3. ~~German-localised entity IDs~~ — fixed (`_print_error`, `_pick_image`).
4. ~~`Preview deaktiviert` vs README~~ — fixed.
18. `min_version: 2025.9.0` is likely too low. The LVGL 9 calls (`lv_obj_remove_flag`, the
    `lv_event_get_target` cast) need an ESPHome release that bundles LVGL 9.
19. OTA has no password. Anyone on the LAN can flash the device.
20. `_pick_image` in ha-bambulab is (as far as I know) the plate image used for skip-object selection,
    not the slicer cover image (`_cover_image`). Confirm it looks right on page 2.

**Correctness**
5. **NaN handling:** when an HA entity becomes `unavailable`, ESPHome sensors publish `NAN` and
   `has_state()` remains true. `(int)NAN` is undefined, so `%`, temperature, layer and remaining labels
   can show garbage such as `-2147483648`. `lbl_pct` is also not clamped to 0–100.
6. `print_active` is only cleared on `failed`/`done_strict`. If a job ends any other way (e.g. status
   goes straight to `idle`), it stays latched until reboot and rule 4b (§8.1) shows a blend instead of
   the solid idle grey.
7. The progress fallback is inconsistent: `last_prog` in some lambdas and 0 in others. This can make
   the ring colour and status text disagree when progress is unavailable.
8. The done colour is inconsistent: ring `0x00FFFF` (cyan), status text and target `0x00BFFF`.
9. In the idle-ish case, the status text treats `prog > 0` (including 100) as printing, while the ring
   uses `0 < prog < 100` or `print_active`. These can disagree, e.g. at 100 % without `finish`.
10. An empty `entity_picture` becomes a URL of just `homeassistant_url`. The device then tries to
    decode HA's HTML as a PNG. Once the first image has loaded, `page2_note` never reappears and the
    stale image stays.
11. Only PNG is accepted. If the cover image is served as JPEG, decoding fails.
12. HA rotates the image `access_token` in `entity_picture` periodically (about every 5 min). Each
    rotation changes the attribute and re-downloads the same image.

**Structure / efficiency (relevant to the planned rewrite and low-power work)**
13. State-priority logic is copy-pasted into five lambdas (ring, status text, status colour, remaining,
    target colour). A single derived state, computed once per tick, would remove whole classes of
    inconsistency (items 7–9).
14. The 50 ms loop sets arc style colours on every tick, even when unchanged. That likely invalidates
    and redraws the ring 20×/s even when idle. The 1 s loop also re-applies label colours and
    re-aligns `row_remaining` every second. Both cost CPU and SPI bandwidth with no visible change.
15. Wi-Fi power save is disabled and brightness is fixed. There is no screen-off or idle detection
    (roadmap item: sleep / low-power mode).
16. Unused: the `time` component (`esptime`) and the `backdrop` widget. The 300 ms build interval keeps
    firing after the UI is built (cheap, but unnecessary).
17. `verify_ssl: false` is set for all HTTP requests.
