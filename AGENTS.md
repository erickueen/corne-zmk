# AGENTS

This file gives coding agents the minimum context needed to work safely in this repository.

## Project

- Repository: `corne-zmk-config`
- Target: Corne split keyboard running ZMK
- Firmware stack: local `zmk/` + Zephyr workspace

## Build

Use Docker-based builds from the repo root.

Left:

```bash
docker run --rm -v "$PWD":/workspaces/zmk-config -w /workspaces/zmk-config -e GIT_CONFIG_COUNT=1 -e GIT_CONFIG_KEY_0=safe.directory -e GIT_CONFIG_VALUE_0='*' zmkfirmware/zmk-build-arm:stable west build -s zmk/app -d build/corne_left -b nice_nano/nrf52840/zmk -- -DSHIELD=corne_left -DZMK_CONFIG=/workspaces/zmk-config/config -DZMK_EXTRA_MODULES=/workspaces/zmk-config/config/corne_rgb_module -DZephyr_DIR=/workspaces/zmk-config/zephyr/share/zephyr-package/cmake
```

Right:

```bash
docker run --rm -v "$PWD":/workspaces/zmk-config -w /workspaces/zmk-config -e GIT_CONFIG_COUNT=1 -e GIT_CONFIG_KEY_0=safe.directory -e GIT_CONFIG_VALUE_0='*' zmkfirmware/zmk-build-arm:stable west build -s zmk/app -d build/corne_right -b nice_nano/nrf52840/zmk -- -DSHIELD=corne_right -DZMK_CONFIG=/workspaces/zmk-config/config -DZMK_EXTRA_MODULES=/workspaces/zmk-config/config/corne_rgb_module -DZephyr_DIR=/workspaces/zmk-config/zephyr/share/zephyr-package/cmake
```

Artifacts:

- `build/corne_left/zephyr/zmk.uf2`
- `build/corne_right/zephyr/zmk.uf2`

After building, copy the UF2 artifacts to the repo root with these names:

- `corne_left.uf2`
- `corne_right.uf2`

## Current behavior notes

- Custom RGB is provided by `config/corne_rgb_module`; include `-DZMK_EXTRA_MODULES=/workspaces/zmk-config/config/corne_rgb_module` in local builds.
- Reactive RGB is a selectable effect.
- Reactive effect keeps underglow LEDs on and lights key LEDs on keypress.
- Magic tap shows temporary status overlay (battery + USB/BT indicators).

## Key files

- Keymap: `config/corne.keymap`
- Kconfig overrides: `config/corne.conf`
- Shield overlays: `config/corne_left.overlay`, `config/corne_right.overlay`
- RGB behavior implementation: `config/corne_rgb_module/src/corne_rgb_underglow.c`
- RGB behavior bindings: `config/corne_rgb_module/src/behavior_rgb_underglow.c`
- RGB status behavior: `config/corne_rgb_module/src/behavior_corne_rgb_status.c`

## Guardrails

- Prefer preserving existing key positions unless explicitly requested.
- Rebuild both halves after firmware/keymap changes.
- Keep RGB changes battery-aware (small default brightness values).
