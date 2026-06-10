# Corne Reactive RGB Implementation Plan

## Goal

Add two RGB features to this Corne ZMK configuration:

- Reactive per-key RGB: pressing a key briefly lights that key's RGB LED.
- GO60-style Magic tap battery meter: tapping Magic shows battery level using a 10-key LED meter.

This requires a small ZMK firmware patch. Stock ZMK RGB underglow can drive the LED strip, but its public keymap behavior only exposes global controls such as toggle, hue, brightness, speed, effects, and global color.

## Current Context

- Corne repo: `/home/erickueen/code/corne-zmk-config`
- GO60 reference repo: `/home/erickueen/code/go60-zmk-config`
- RGB is currently working through ZMK RGB underglow.
- Each half currently defines one WS2812-compatible LED strip on `P0.06`.
- Each half uses `chain-length = <27>`.
- Each half sets `zmk,underglow = &led_strip`.

Current Corne RGB config lives in:

- `config/corne.conf`
- `config/corne_left.overlay`
- `config/corne_right.overlay`

Current relevant config options:

```ini
CONFIG_ZMK_RGB_UNDERGLOW=y
CONFIG_ZMK_RGB_UNDERGLOW_EXT_POWER=n
CONFIG_ZMK_RGB_UNDERGLOW_ON_START=y
CONFIG_ZMK_RGB_UNDERGLOW_AUTO_OFF_USB=y
```

Current ZMK files to patch:

- `zmk/app/include/dt-bindings/zmk/rgb.h`
- `zmk/app/include/zmk/rgb_underglow.h`
- `zmk/app/src/behaviors/behavior_rgb_underglow.c`
- `zmk/app/src/rgb_underglow.c`

Current Corne keymap file:

- `config/corne.keymap`

Stock ZMK Corne shield context:

- `zmk/app/boards/shields/corne/corne.dtsi`
- It currently contains `// TODO: per-key RGB node(s)?`, so upstream Corne does not provide a complete per-key RGB abstraction.

## GO60 Reference

GO60 already has the desired keymap-level behavior shape.

Reference file:

- `/home/erickueen/code/go60-zmk-config/config/go60.keymap`

GO60 Magic tap status macro:

```dts
rgb_ug_status_macro: rgb_ug_status_macro {
    label = "RGB_UG_STATUS";
    compatible = "zmk,behavior-macro";
    #binding-cells = <0>;
    bindings
        = <&rgb_ug RGB_STATUS>;
};
```

GO60 Magic hold-tap uses the status macro on tap:

```dts
magic: magic {
    compatible = "zmk,behavior-hold-tap";
    label = "MAGIC_HOLD_TAP";
    #binding-cells = <2>;
    flavor = "tap-preferred";
    tapping-term-ms = <200>;
    bindings = <&mo>, <&rgb_ug_status_macro>;
};
```

Important finding:

- `RGB_STATUS` is not present in this repo's stock ZMK `rgb.h`.
- GO60 is therefore using a patched/generated ZMK behavior, not pure upstream stock ZMK.
- We need to recreate equivalent support in this Corne repo.

## Corne Key Position Reference

From `config/corne.keymap`, ZMK positions follow this 42-key Corne order:

```text
Top row:    0  1  2  3  4  5    6  7  8  9 10 11
Home row:  12 13 14 15 16 17   18 19 20 21 22 23
Bottom:    24 25 26 27 28 29   30 31 32 33 34 35
Thumbs:             36 37 38   39 40 41
```

Battery meter keys on the left side:

```text
Q W E R T = positions 1 2 3 4 5
A S D F G = positions 13 14 15 16 17
```

Equivalent mirrored right-side meter keys:

```text
Y U I O P = positions 6 7 8 9 10
H J K L ; = positions 18 19 20 21 22
```

## QMK LED Map Reference

Use QMK Corne's `rgb_matrix.layout` as the first LED order reference.

Reference URL:

- `https://raw.githubusercontent.com/qmk/qmk_firmware/master/keyboards/crkbd/rev1/keyboard.json`

QMK defines:

```json
"rgblight": {
    "led_count": 54,
    "split_count": [27, 27]
},
"rgb_matrix": {
    "split_count": [27, 27]
}
```

QMK left half LED indices:

- `0-5`: underglow LEDs
- `6-26`: key LEDs

QMK right half LED indices:

- `27-32`: underglow LEDs
- `33-53`: key LEDs

In ZMK, each half runs a local `0-26` strip, so right-half QMK LED indices must be normalized by subtracting `27`.

## Phase 1: Add RGB_STATUS Command

Purpose: make Corne support GO60-style `&rgb_ug RGB_STATUS`.

Files:

- `zmk/app/include/dt-bindings/zmk/rgb.h`
- `zmk/app/src/behaviors/behavior_rgb_underglow.c`
- `zmk/app/include/zmk/rgb_underglow.h`
- `zmk/app/src/rgb_underglow.c`

Tasks:

- Add `RGB_STATUS_CMD` after existing RGB command values.
- Add keymap macro `RGB_STATUS`.
- Add a function declaration, likely `int zmk_rgb_underglow_show_status(void);`.
- Handle `RGB_STATUS_CMD` in `on_keymap_binding_pressed()`.
- Implement `zmk_rgb_underglow_show_status()`.

Acceptance criteria:

- `&rgb_ug RGB_STATUS` compiles.
- Existing RGB controls still compile and work.
- Magic tap can call status without toggling RGB.

## Phase 2: Add Corne Key-To-LED Mapping

Purpose: map a ZMK key position to a physical local LED index.

Left half local LED map from QMK:

```c
position 38 -> LED 6
position 29 -> LED 7
position 17 -> LED 8
position 5  -> LED 9
position 4  -> LED 10
position 16 -> LED 11
position 28 -> LED 12
position 37 -> LED 13
position 36 -> LED 14
position 27 -> LED 15
position 15 -> LED 16
position 3  -> LED 17
position 2  -> LED 18
position 14 -> LED 19
position 26 -> LED 20
position 25 -> LED 21
position 13 -> LED 22
position 1  -> LED 23
position 0  -> LED 24
position 12 -> LED 25
position 24 -> LED 26
```

Right half local LED map from QMK, normalized to `0-26`:

```c
position 39 -> LED 6
position 30 -> LED 7
position 18 -> LED 8
position 6  -> LED 9
position 7  -> LED 10
position 19 -> LED 11
position 31 -> LED 12
position 40 -> LED 13
position 41 -> LED 14
position 32 -> LED 15
position 20 -> LED 16
position 8  -> LED 17
position 9  -> LED 18
position 21 -> LED 19
position 33 -> LED 20
position 34 -> LED 21
position 22 -> LED 22
position 10 -> LED 23
position 11 -> LED 24
position 23 -> LED 25
position 35 -> LED 26
```

Implementation notes:

- Use a helper like `position_to_led_index(uint32_t position)`.
- Return invalid/no LED for positions not on the current physical half.
- Leave underglow LEDs `0-5` alone for the first version.
- Be ready to tune this map because physical LED numbering may not match the printed labels. The user already observed non-obvious chain order.

Acceptance criteria:

- The helper maps local key positions to LED indices `6-26`.
- The helper ignores opposite-side keys.
- The mapping can be tested and adjusted incrementally.

## Phase 3: Battery Meter Overlay

Purpose: reproduce GO60's Magic tap status indicator.

Behavior:

- Magic tap triggers `&rgb_ug RGB_STATUS`.
- Firmware reads local battery with `zmk_battery_state_of_charge()`.
- Display duration: `1500ms`.
- Show a 10-segment meter.
- After timeout, resume normal RGB.

Battery API reference:

- Header: `zmk/app/include/zmk/battery.h`
- Function: `uint8_t zmk_battery_state_of_charge(void);`

Recommended left meter:

```text
Q W E R T A S D F G
1 2 3 4 5 13 14 15 16 17
```

Recommended right meter:

```text
Y U I O P H J K L ;
6 7 8 9 10 18 19 20 21 22
```

Recommended number of lit keys:

- Use ceiling: `(percent + 9) / 10`
- `87%` lights 9 keys.
- `42%` lights 5 keys.
- Nonzero percentages light at least 1 key.

Recommended colors:

- `>= 50%`: green
- `20-49%`: yellow
- `< 20%`: red

Acceptance criteria:

- Magic tap shows battery meter briefly.
- Magic hold still activates Magic layer.
- Normal RGB resumes after timeout.
- OLED battery percentage remains unchanged.

## Phase 4: Reactive Per-Key Lighting

Purpose: flash the LED under a key when it is pressed.

Relevant ZMK event:

- Header: `zmk/app/include/zmk/events/position_state_changed.h`
- Event: `zmk_position_state_changed`
- Useful fields: `position`, `state`, `source`, `timestamp`

Implementation:

- Subscribe from the RGB implementation to `zmk_position_state_changed`.
- On press, map `position` to a local LED index.
- Store a temporary active state for that LED.
- During each RGB tick, render normal RGB first, then overlay reactive LEDs.
- On release, initially do nothing and let timeout expire.

Recommended first-pass defaults:

- Reactive color: white or cyan.
- Reactive duration: `200ms`.
- Fade: skip initially. Add later after mapping is correct.

Acceptance criteria:

- Pressing `Q` lights only Q's LED.
- Pressing `A` lights only A's LED.
- Pressing right-side keys lights right-side LEDs.
- Multiple simultaneous key presses can light multiple LEDs.
- Existing RGB effects continue underneath.

## Phase 5: Render Priority Rules

Use this priority order:

1. RGB off means all LEDs off.
2. Battery status overlay owns meter LEDs while active.
3. Reactive key overlay overrides normal effect on active key LEDs.
4. Normal RGB effect fills the base frame.

This avoids confusing behavior:

- Battery status stays readable.
- Reactive lighting does not permanently alter color/effect settings.
- Existing Magic layer RGB controls continue controlling base RGB.

## Phase 6: Keymap Wiring

File:

- `config/corne.keymap`

Change `rgb_ug_status_macro` back from `&none` to `&rgb_ug RGB_STATUS` after Phase 1 compiles:

```dts
rgb_ug_status_macro: rgb_ug_status_macro {
    label = "RGB_UG_STATUS";
    compatible = "zmk,behavior-macro";
    #binding-cells = <0>;
    bindings = <&rgb_ug RGB_STATUS>;
};
```

Keep Magic behavior shape:

```dts
magic: magic {
    compatible = "zmk,behavior-hold-tap";
    label = "MAGIC_HOLD_TAP";
    #binding-cells = <2>;
    flavor = "tap-preferred";
    tapping-term-ms = <200>;
    bindings = <&mo>, <&rgb_ug_status_macro>;
};
```

Acceptance criteria:

- Tapping Magic shows battery status.
- Holding Magic still opens `LAYER_Magic`.
- RGB controls on Magic layer still work.

## Phase 7: Build And Verify

Build left:

```bash
docker run --rm -v "$PWD":/workspaces/zmk-config -w /workspaces/zmk-config zmkfirmware/zmk-build-arm:stable west build -s zmk/app -d build/corne_left -b nice_nano -- -DSHIELD=corne_left -DZMK_CONFIG=/workspaces/zmk-config/config -DZephyr_DIR=/workspaces/zmk-config/zephyr/share/zephyr-package/cmake
```

Build right:

```bash
docker run --rm -v "$PWD":/workspaces/zmk-config -w /workspaces/zmk-config zmkfirmware/zmk-build-arm:stable west build -s zmk/app -d build/corne_right -b nice_nano -- -DSHIELD=corne_right -DZMK_CONFIG=/workspaces/zmk-config/config -DZephyr_DIR=/workspaces/zmk-config/zephyr/share/zephyr-package/cmake
```

Copy outputs:

```bash
cp build/corne_left/zephyr/zmk.uf2 corne_left.uf2
cp build/corne_right/zephyr/zmk.uf2 corne_right.uf2
```

Verify generated configs:

- `CONFIG_ZMK_RGB_UNDERGLOW=y`
- `CONFIG_ZMK_RGB_UNDERGLOW_ON_START=y`
- `CONFIG_ZMK_RGB_UNDERGLOW_AUTO_OFF_USB=y`
- `CONFIG_ZMK_BATTERY_NRF_VDDH=y`
- `CONFIG_ZMK_WIDGET_BATTERY_STATUS_SHOW_PERCENTAGE=y`

Verify generated DTS:

- `zmk,underglow = &led_strip`
- `chain-length = < 0x1b >`

## Phase 8: Hardware Test Checklist

Basic RGB:

- RGB still turns on over USB.
- Existing Magic layer RGB controls still work.
- `RGB_TOG`, brightness, hue, and effect controls still work.

Magic tap:

- Tap Magic once.
- Battery meter appears.
- Meter disappears after timeout.
- Normal RGB resumes.

Magic hold:

- Hold Magic.
- Magic layer activates.
- Battery meter does not trigger unless tapped.

Reactive lighting:

- Press `Q`; only Q should flash.
- Press `A`; only A should flash.
- Press right-side keys; corresponding right-side LEDs should flash.
- Press multiple keys; multiple LEDs should flash.

Split behavior:

- Right-half key presses light right-half LEDs.
- Left-half key presses light left-half LEDs.
- If one half is disconnected, the connected side still operates normally.

## Risks And Notes

- LED ordering may differ from the QMK reference due to PCB revision or assembly.
- The user already observed non-obvious LED chain order, so mapping should be validated incrementally.
- Patching `zmk/app/src/rgb_underglow.c` is a fork-level change; future ZMK updates may need conflict resolution.
- Split event locality may need adjustment depending on which events are visible on peripheral firmware versus central firmware.
- Right-half battery status may need special handling if the desired behavior is to show peripheral battery on the central side. First implementation should show local battery on local LEDs.

## Recommended Execution Order

1. Add `RGB_STATUS` command and make it compile.
2. Implement battery meter only.
3. Flash and verify Magic tap behavior.
4. Implement reactive lighting without fade.
5. Flash and verify LED mapping.
6. Tune LED mapping if needed.
7. Add fade/polish only after mapping is correct.
