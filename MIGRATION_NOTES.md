# KLOR Repo Audit — Current State vs. Work Brief

Repo cloned from `https://github.com/BFotouh/zmk-config-klor.git` into this directory (single `master` branch, working tree clean, no CMakeLists.txt or build artifacts committed).

## 1. Correction to the brief: the repo is NOT pinned to v0.3.0

`config/west.yml` already reads `revision: main`, and `.github/workflows/build.yml` already uses `@main`:

```yaml
# config/west.yml
projects:
  - name: zmk
    remote: zmkfirmware
    revision: main
    import: app/west.yml
```

```yaml
# .github/workflows/build.yml
jobs:
  build:
    uses: zmkfirmware/zmk/.github/workflows/build-user-config.yml@main
```

`git log --follow` on both files shows they've tracked `main` since 2024 (west.yml) and since the repo's initial commit in 2022 (build.yml) — there is no commit in this fork's history that ever set `v0.3.0`. So step 2 of the brief's migration order ("unpin ZMK") is already done; **the repo has probably been silently floating on whatever `main` was at each CI run**, which is a different risk than being stuck on an old pin — it means nobody has confirmed a *specific* known-good `main` commit builds cleanly. I'd treat step 3 of the brief's order of operations (green two-part build on `main`, isolated from dongle work) as still fully necessary — just note the framing changes from "unpin" to "pin down and verify."

## 2. Repo structure

```
config/
  west.yml
  klor.conf                          # shared Kconfig: encoders, display, RGB
  klor.keymap                        # single shared keymap (custom ASCII-art layout diagrams)
  boards/shields/klor/
    Kconfig.shield                   # SHIELD_KLOR_LEFT / SHIELD_KLOR_RIGHT
    Kconfig.defconfig                # per-shield defaults (keyboard name, central role, I2C/SSD1306/LVGL)
    klor.zmk.yml                     # ZMK app-metadata file (requires pro_micro, exposes i2c_oled)
    klor.dtsi                        # shared matrix transform (12x4, bare zmk,matrix_transform)
    klor_common.dtsi                 # kscan, OLED chosen node, encoder nodes, stale RGB TODO comment
    klor_left.overlay / klor_left.conf    (conf is empty, 0 bytes)
    klor_right.overlay / klor_right.conf  (conf is empty, 0 bytes)
    boards/
      nice_nano_v2.overlay           # ACTIVE: RGB SPI/WS2812 node, D1=P0.06
      nice_nano.overlay              # legacy nice!nano v1 variant, same RGB pattern, unused by build.yaml
      nrfmicro_11.overlay, nrfmicro_11_flipped.overlay, nrfmicro_13.overlay   # legacy, unused by build.yaml
    battery_status.c/.h
    output_status.c/.h
    profile_status.c/.h
    klor_status_screen.c
    icons/*.c                        # ~19 LVGL image C arrays (battery levels, BT/USB glyphs, KLOR logo)
build.yaml                            # 2 targets: klor_left, klor_right on nice_nano_v2 (settings_reset commented out)
.github/workflows/build.yml
readme.md
docs/images/*.svg
```

No `zephyr/module.yml` anywhere in the tree — confirms brief open question #1: it does **not** match the unified-config-template layout yet. That file must be added before the dongle shield search path will resolve (§3.1 of the brief).

**No `CMakeLists.txt` exists anywhere in the repo.** This is the biggest surprise from this audit:

## 3. The custom OLED widget C code is currently dead code

`battery_status.c`, `output_status.c`, `profile_status.c`, `klor_status_screen.c`, and all 19 files under `icons/` implement a fully custom LVGL status screen (`zmk_display_status_screen()` override, custom battery/output/profile-index icon widgets, KLOR logo splash). But **without a `CMakeLists.txt` in the shield directory to add these as sources, ZMK's CMake build has no way to compile or link them.** Since the brief says OLED is "working" today, the working OLED must be ZMK's stock built-in status widgets — this custom code is orphaned, not currently in the running firmware.

This matters a lot for the migration because the code itself is written against a very old API surface:
- Old Zephyr include paths (`<kernel.h>`, `<logging/log.h>`, `<bluetooth/services/bas.h>`) instead of the `<zephyr/...>` prefix current Zephyr has required for years.
- LVGL v7-era calls (`lv_obj_create(NULL, NULL)` two-arg form, `LV_ALIGN_IN_TOP_LEFT`, `lv_style_set_text_font(&style, LV_STATE_DEFAULT, &font)` three-arg form).
- Current ZMK `main` runs **LVGL 9.3** (per ZMK's Dec 2025 Zephyr 4.1 update) — three major LVGL versions ahead, plus the ZMK display/widget framework itself has been rewritten since this code was written.

**Decision point for you, not something I'll resolve silently:** either (a) leave this code deleted/unused and rely on ZMK's current default status screen or the community modules the brief proposes in §5, or (b) treat reviving it as its own project — realistically a near-total rewrite against the current LVGL 9.3 + ZMK display API, not a "renamed CONFIG_ symbols" fix. Given the brief already plans to adopt `zmk-nice-oled` (§5a) and, post-dongle, `zmk-dongle-display` (§5c) as the OLED upgrade path, my read is this legacy code is safe to delete rather than port — but that's your call since it's not part of what's currently running.

## 4. Concrete `main`-breakage findings (beyond what the brief guessed)

These are verified against ZMK's own Dec 2025 "Zephyr 4.1 Update" migration notes and current docs — not guesses:

| File | Current value | Needed on `main` |
|---|---|---|
| `build.yaml` (all entries) | `board: nice_nano_v2` | `board: nice_nano//zmk` (nice!nano v2 is now the *default* revision under Hardware Model V2 naming — full form `nice_nano@2.0.0//zmk`, shortened form `nice_nano//zmk`). This resolves the brief's open question #2 directly. |
| `config/klor.conf:10` | `CONFIG_WS2812_STRIP=y` | Deprecated/removed symbol per the Zephyr 4.1 update notes — needs removing (the `worldsemi,ws2812-spi` compatible now pulls in what it needs via devicetree/Kconfig `select`, not this flag). Verify the build doesn't warn/fail on it and drop it. |
| `config/boards/shields/klor/Kconfig.defconfig:6` | `config ZMK_SPLIT_BLE_ROLE_CENTRAL` | Renamed to `CONFIG_ZMK_SPLIT_ROLE_CENTRAL` (the brief's own dongle-Kconfig snippet in §3.2 already uses the new name — this existing file has the old one and needs to match). |
| Keymap / bootloader flashing | Brief's §3.4 mentions `&bootloader` keymap binding as a flash-entry option | ZMK's Zephyr 4.1 notes say "the previous method to enable `&bootloader` has been disabled" in favor of boot retention — verify the current mechanism before relying on a `&bootloader` binding; the `stty ... 1200` double-reset path in the brief is unaffected and remains the reliable option. |

Not present in this repo (so not a concern): `CONFIG_NFCT_PINS_AS_GPIOS`, `BOARD_ENABLE_DCDC*` — grepped, no hits.

## 5. Physical layout / matrix transform (open question #3, resolved)

Confirmed: **bare `zmk,matrix_transform` only**, no `zmk,physical-layout` nodes anywhere.

- `klor.dtsi` defines `default_transform` (12 cols × 4 rows, `compatible = "zmk,matrix-transform"`) and sets it via `chosen { zmk,matrix_transform = &default_transform; }`.
- `klor_right.overlay` reuses it with `&default_transform { col-offset = <6>; };`.
- Note the brief's own dongle-overlay snippet writes the chosen property as `zmk,matrix-transform` (hyphen) — the actual ZMK chosen-node property name is `zmk,matrix_transform` (underscore); the hyphen form is the node's `compatible` string, not the chosen key. Use the underscore form to match what's already in `klor.dtsi`/`klor_common.dtsi`, or the dongle overlay's chosen node won't resolve.
- Per §3.2, this whole `default_transform` node (12x4, same node name) needs to be copied verbatim into the new `klor_dongle.overlay`, with `kscan = <&kscan0>` not present there since there's no kscan reference on the bare transform node to begin with (that's only a physical-layout-node concern) — so the dongle overlay is simpler than the brief's caveat about stripping `kscan` implies, since there's no physical-layout node to strip it from. Just copy the `default_transform` node and the `chosen` line.

## 6. RGB — confirms brief, nothing new

`config/boards/shields/klor/boards/nice_nano_v2.overlay` — SPI3, `NRF_PSEL(SPIM_MOSI, 0, 6)` = D1 = P0.06, `worldsemi,ws2812-spi`, `chain-length = <21>`, `chosen { zmk,underglow = &led_strip; }`. Matches the brief exactly. The `// TODO: RGB node(s)` comment in `klor_common.dtsi:46` is confirmed stale (RGB lives in the board overlay, not the common dtsi) — brief was right to flag it as noise.

## 7. Encoders

`klor_common.dtsi` defines both `left_encoder` and `right_encoder` as `alps,ec11`, both `status = "disabled"` by default; each half's overlay (`klor_left.overlay` / `klor_right.overlay`) re-enables its own local encoder only. `readme.md`'s "KNOWN ISSUES" section still says *"The encoder on the secondary side doesn't work yet"* — the brief already flagged this as stale (PR #1841 merged upstream); confirmed the repo's own README hasn't been updated to reflect that fix.

## 8. Keymap (`config/klor.keymap`)

- 4 layers (BASE/RAISE/LOWER/ADJUST), homerow-mods behavior (`hm`, tap-preferred, 250ms), 5 combos, 3 macros, encoder `sensor-bindings` on every layer.
- No `&out OUT_TOG` binding anywhere in the current keymap — so the brief's §3.5 point about `OUT_TOG` semantics changing post-dongle is forward-looking, not something to migrate away from.
- No `&bootloader` binding either — flashing currently relies entirely on the double-reset/mass-storage method.
- Syntax otherwise looks like standard current-ish devicetree keymap form (no legacy `bindings = <&kp ...>;` quirks spotted) — still worth a pass through the ZMK Keymap Upgrader per the brief, but nothing jumped out as guaranteed-broken.

## 9. Haptic / piezo devicetree

Zero existing devicetree or Kconfig scaffolding for DRV2605L or piezo anywhere in the repo — this is a from-scratch addition exactly as the brief's §6.3 describes. I²C bus is `pro_micro_i2c` (already `status = "okay"` for the SSD1306 at `0x3c`); DRV2605L at `0x5A` should coexist fine, no address collision, as the brief expected — but I can't confirm SDA/SCL pin routing or the BZ1/BZ2 PWM pin without the Altium schematic (open questions #6/#7/#8 in the brief remain genuinely open — nothing in this repo answers them, since it's firmware-config only, not a board-support-package with a full pinout).

## 10. Modules referenced in the brief — existence verified

Confirmed all reachable on GitHub right now: `badjeff/zmk-output-behavior-listener`, `badjeff/zmk-drv2605-driver`, `badjeff/zmk-split-peripheral-output-relay`, `ssbb/zmk-output-behavior-listener`, `mctechnology17/zmk-nice-oled`, `mctechnology17/zmk-oled-adapter`, `englmaxi/zmk-dongle-display`, `zmkfirmware/unified-zmk-config-template`. (Existence only — didn't audit each one's own README/Kconfig surface in depth; do that at the point each is actually added, per the brief's one-module-per-commit plan.)

## 11. Net effect on the brief's suggested commit sequence

Steps 1–2 (branch, unpin) collapse into one step since there's nothing to unpin — just cut `feat/dongle-main` from current `master` and go straight to fixing `main`-breakage (board target string, `WS2812_STRIP`, `SPLIT_BLE_ROLE_CENTRAL` rename, keymap-upgrader pass) for a green two-part build. Everything from step 4 onward (`zephyr/module.yml`, dongle shield, three-target build.yaml, modules) stands as written in the brief.

## Still genuinely open (repo can't answer these)

- Dongle OLED presence/wiring (brief open Q4) and power source (Q5) — hardware decisions.
- I²C SDA/SCL pins and DRV2605L bus sharing specifics beyond "no address collision" (Q6), and the BZ1/BZ2 PWM-capable pin (Q7) — need the Altium schematic or a continuity check (Q8).
- Whether to delete or attempt to revive the orphaned custom OLED widget C code (§3 above) — new question this audit surfaced, not in the original brief.
