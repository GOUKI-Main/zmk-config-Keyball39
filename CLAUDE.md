# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a ZMK firmware configuration for the Keyball39 keyboard - a 39-key split keyboard with an integrated PMW3360 trackball on the **left half**. The keyboard runs on nice!nano v2 controllers with RGB underglow support (24 LEDs per half).

## Build System

### Building Firmware

Builds are automated via GitHub Actions. Push or PR triggers build workflow defined in `.github/workflows/build.yml`, which uses ZMK's official build workflow.

Build configuration is in `build.yaml`:
- Left half: `nice_nano_v2` + `keyball39_left` shield
- Right half: `nice_nano_v2` + `keyball39_right` shield + `studio-rpc-usb-uart` snippet
- Settings reset: `nice_nano_v2` + `settings_reset` shield

Firmware artifacts are generated and available as GitHub Actions artifacts.

### Keymap Visualization

Keymap drawings are automatically generated via `.github/workflows/keymap_drawer.yml` when keymap files change. Outputs SVG/PNG to `keymap-drawer/` folder.

## Architecture

### West Workspace Structure (`config/west.yml`)

This is a ZMK user config repository structured as a West workspace:

- **ZMK Core**: `zmkfirmware/zmk` @ v0.2
- **Trackball Driver (Current)**: `inorichi/zmk-pmw3360-driver` @ main (PMW3360 optical sensor)
- **Trackball Driver (Future)**: `kumamuk-git/zmk-pmw3610-driver` @ main (commented out, for future migration)
- **Config Path**: `config/` (this repository's config directory)

When modifying dependencies or adding new drivers, edit `config/west.yml`.

### Shield Definition (`config/boards/shields/keyball_nano/`)

The keyboard is defined as a ZMK shield with left/right split halves:

- **`keyball39.dtsi`**: Shared device tree definitions
  - Matrix transform (4 rows × 12 cols)
  - Physical layout with key positions
  - KSCAN configuration (col2row, GPIO pins)
  - I2C setup for OLED display (SSD1306 @ 0x3c)

- **`keyball39_left.overlay`**: Left half specific
  - Column GPIO mappings (pro_micro pins 4-9)
  - **SPI1 configuration for PMW3360 trackball sensor**
  - Trackball device node with layer mappings:
    - `automouse-layer = <4>` (MOUSE layer)
    - `scroll-layers = <5>` (SCROLL layer)
    - `snipe-layers = <6>` (SNIPE layer)
  - Input listener for trackball events

- **`keyball39_right.overlay`**: Right half specific
  - Column GPIO mappings (pro_micro pins 4-9)
  - No trackball configuration (trackball is on left half)

- **`boards/nice_nano_v2.overlay`**: Board-specific configuration
  - SPI3 configuration for RGB underglow (WS2812 LEDs on P0.06)
  - 24 LEDs per half, 48 total

- **Configuration files**:
  - `keyball39.conf`: Shared config in root `config/keyball39.conf` (Bluetooth, RGB, power management)
  - `keyball39_left.conf`: Left half config with PMW3360 sensor settings
  - `keyball39_right.conf`: Right half config (empty, trackball moved to left)

### Keymap Structure (`config/keyball39.keymap`)

Seven layers defined:
0. **DEFAULT**: QWERTY base layer
1. **NUM**: Number row and arrow keys
2. **SYM**: Symbols and Bluetooth controls (BT_SEL 0-4 for 5-device switching, BT_CLR)
3. **FUN**: Function keys F1-F12 with RGB underglow controls (toggle, effects, brightness, color)
4. **MOUSE**: Mouse emulation with trackball
5. **SCROLL**: Scroll mode for trackball
6. **SNIPE**: Precision mode with bootloader access

Key behaviors configured:
- Custom mod-morph behaviors (`cmqus`, `dtsmi`) for shifted punctuation
- Layer-tap combinations extensively used
- Caps word with underscore/minus continuation
- Hiragana macro (Ctrl+Space for IME switching)

### Configuration Files

- **`config/keyball39.conf`**: Shared settings
  - Bluetooth connection parameters (optimized for low latency)
  - Display and external power enabled
  - BLE experimental features enabled
  - Split battery level fetching/proxy
  - Behavior queue size: 512
  - **RGB underglow configuration**:
    - Default: White color, 30% brightness (battery saving)
    - Auto-off when idle, stays on when USB connected
    - Solid effect by default
  - **Power management**:
    - Idle timeout: 30 seconds
    - Sleep enabled after 15 minutes
    - External power control enabled

- **`config/boards/shields/keyball_nano/keyball39_left.conf`**: Trackball settings (LEFT half)
  - PMW3360 sensor configuration (SPI, input, mouse support)
  - CPI settings: 1200 (normal), 400 (snipe mode)
  - Scroll tick: 32
  - Smart algorithm enabled
  - Polling rate: 125Hz
  - Auto-mouse timeout: 700ms
  - ZMK Studio enabled
  - **Future PMW3610 migration settings included (commented out)**

## Key Modification Guidelines

### Changing Keymap

Edit `config/keyball39.keymap`:
- Layer indices must match the `#define` at top (DEFAULT=0, NUM=1, etc.)
- Maintain 39 key positions per layer (10+10+10+9 format)
- Layer-tap keys use format: `&lt <layer> <key>`

### Trackball Behavior

Modify trackball settings in `config/boards/shields/keyball_nano/keyball39_left.conf`:
- `CONFIG_PMW3360_CPI`: Cursor speed (default 1200)
- `CONFIG_PMW3360_SNIPE_CPI`: Precision mode speed (default 400)
- `CONFIG_PMW3360_SCROLL_TICK`: Scroll sensitivity (default 32)
- `CONFIG_PMW3360_AUTOMOUSE_TIMEOUT_MS`: Auto-return to base layer delay (default 700ms)

Trackball layer assignments in `keyball39_left.overlay`:
- Change `automouse-layer`, `scroll-layers`, or `snipe-layers` properties

**Note**: The trackball is on the **left (peripheral)** half. Input events are forwarded to the right (central) half via BLE split protocol, adding ~10-20ms latency (imperceptible for trackball use).

### RGB Underglow Control

RGB underglow can be controlled via keymap or configuration:

**Keymap controls** (FUN layer):
- `RGB_TOG`: Toggle on/off
- `RGB_EFF`: Cycle through effects (solid, breathe, spectrum, swirl)
- `RGB_BRI`/`RGB_BRD`: Increase/decrease brightness
- `RGB_HUI`/`RGB_HUD`: Change color (hue)
- `RGB_SAI`/`RGB_SAD`: Change color saturation

**Configuration** (`config/keyball39.conf`):
- `CONFIG_ZMK_RGB_UNDERGLOW_BRT_START`: Default brightness (0-100)
- `CONFIG_ZMK_RGB_UNDERGLOW_HUE_START`: Default hue (0-360)
- `CONFIG_ZMK_RGB_UNDERGLOW_SAT_START`: Default saturation (0-100)
- `CONFIG_ZMK_RGB_UNDERGLOW_EFF_START`: Default effect (0=solid, 1=breathe, etc.)

**Hardware** (`boards/nice_nano_v2.overlay`):
- `chain-length`: Number of LEDs per half (currently 24)
- Must match physical LED count

### Adding Dependencies

To add new ZMK modules or drivers:
1. Add remote to `config/west.yml` under `remotes:`
2. Add project under `projects:` with name, remote, revision, and optional import
3. Commit and push to trigger rebuild

## Hardware Notes

- **Controllers**: nice!nano v2 (nRF52840-based, Bluetooth LE)
  - Modified 13-pin conthrough to 12-pin (B+/B- removed, RAW and below only)
  - USB-C bridge between left and right RAW pins for simultaneous charging
- **Display**: SSD1306 OLED 128×32px on I2C
- **Trackball**: PMW3360 optical sensor on SPI (**left half only**)
  - Will migrate to PMW3610 in the future (settings prepared in comments)
- **RGB Underglow**: 24× WS2812 LEDs per half (48 total) on P0.06 (SPI3)
  - One half may have incomplete LED installation; configuration supports both
- **Matrix**: 4 rows × 6 columns per half, col2row diodes
- **Split Communication**: Wireless (Bluetooth) between halves
  - **Left half**: BLE Central (primary, USB host connection)
  - **Right half**: BLE Peripheral (secondary)
  - USB cable should be connected to **left half**
- **Bluetooth Profiles**: Supports 5 BLE profiles + USB (6 devices total)

## Migration Guide: PMW3360 → PMW3610

When migrating from PMW3360 to PMW3610 sensor:

1. **Edit `config/west.yml`**:
   - Comment out `inorichi/zmk-pmw3360-driver`
   - Uncomment `kumamuk-git/zmk-pmw3610-driver`

2. **Edit `config/boards/shields/keyball_nano/keyball39_left.overlay`**:
   - Change `compatible = "pixart,pmw3360"` → `"pixart,pmw3610"`

3. **Edit `config/boards/shields/keyball_nano/keyball39_left.conf`**:
   - Comment out all `CONFIG_PMW3360_*` settings
   - Uncomment all `CONFIG_PMW3610_*` settings (pre-configured)

4. **Test**:
   - Commit changes and verify GitHub Actions build succeeds
   - Flash firmware and test trackball functionality

## Credits

- PCB: yangxing844
- Case: delock
- Firmware: Amos698
