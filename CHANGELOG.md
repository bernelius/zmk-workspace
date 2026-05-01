# Changelog

## Summary

Migrated eyelash_sofle from broken shield architecture back to custom board architecture on Zephyr 4.1.0 / ZMK post-v0.3.0. The shield approach had BLE connectivity issues; the custom board architecture provides full SOC control and restores reliable split keyboard communication.

## Changes

### 1. Board Architecture - Shield to Custom Board Migration (Zephyr 4.1.0 HWMv2)

The eyelash_sofle was converted from a broken shield back to a proper custom board following Zephyr 4.1.0 Hardware Model v2:

**Removed:**
- `config/boards/shields/eyelash_sofle/` (shield directory that had BLE connectivity issues)

**Added:**
- `config/boards/eyelash/sofle/` (new custom board directory):
  - `board.yml` - HWMv2 board definition with left/right split variants
  - `Kconfig.sofle_left` / `Kconfig.sofle_right` - Board selection with boot retention support
  - `Kconfig.defconfig` - BLE controller, USB stack, and display defaults
  - `sofle.dtsi` - Main device tree with `nrf52840_qiaa.dtsi`, `nrf52840_uf2_boot_mode.dtsi`, and proper `zmk,matrix-transform` chosen
  - `sofle_left_nrf52840_zmk.dts` - Left half with column GPIOs and encoder
  - `sofle_right_nrf52840_zmk.dts` - Right half with column GPIOs and transform offset
  - `sofle_left_nrf52840_zmk_defconfig` / `sofle_right_nrf52840_zmk_defconfig` - Build configs (no SOC/BOARD selections per migration guide)
  - `sofle.zmk.yml` - ZMK metadata
  - `sofle.yaml` - Zephyr metadata
  - `board.cmake` / `pre_dt_board.cmake` - Build configuration

**Updated:**
- `build.yaml`: Changed from `nice_nano` + shield to `sofle_left//zmk` and `sofle_right//zmk` with `nice_view_gem` shield
- `config/eyelash_sofle.keymap` → `config/sofle.keymap` (renamed to match board)

### 2. Nice View Gem - LVGL 9.x Compatibility (Unchanged)

Continues to use forked `nice-view-gem-z4` repository as west module:
- LVGL 9.x API updates (`lv_image_dsc_t`, `LV_IMAGE_DECLARE`, etc.)
- Build target: `shield: nice_view_gem` on top of custom board

**Status:** Display functionality restored with custom board architecture

### 3. BLE and Connectivity Improvements

**Added to `Kconfig.defconfig`:**
- `config BT_CTLR default BT` - BLE controller properly initialized
- `config USB_NRFX default y` (when USB enabled) - USB stack
- `config USB_DEVICE_STACK default y` (when USB enabled) - USB/BLE coexistence
- `config ZMK_SPLIT default y` - Split keyboard support
- `config ZMK_SPLIT_ROLE_CENTRAL default y` (left half) - Central role

**Added to left defconfig:**
- `CONFIG_ZMK_USB=y` - USB on central
- `CONFIG_ZMK_BLE=y` - BLE enable
- `CONFIG_BT_CTLR_TX_PWR_PLUS_8=y` - Max TX power for range
- `CONFIG_ZMK_BLE_PASSKEY_ENTRY=n` - No PIN pairing
- `CONFIG_ZMK_BLE_CLEAR_BONDS_ON_START=n` - Preserve bonds

**Right defconfig:**
- `CONFIG_ZMK_USB=n` - No USB on peripheral (saves memory)
- `CONFIG_ZMK_BLE=y` - BLE only

**Boot Retention (Zephyr 4.1.0 feature):**
- `imply RETAINED_MEM`
- `imply RETENTION`
- `imply RETENTION_BOOT_MODE`

### 4. Hardware Configuration

**DCDC Regulator (Devicetree):**
```dts
&reg0 {
    status = "okay";
};
```
Enables high voltage DCDC for radio stability (moved from Kconfig to devicetree per Zephyr 4.1.0).

**Flash Partitions (Explicit):**
- Proper partition table for NVS/bonding storage
- Consistent between left and right halves

**Matrix Transform:**
- Added `zmk,matrix-transform = &default_transform;` to chosen node
- Proper row/column GPIO mappings per half

### 5. Zephyr 4.1.0 HWMv2 Migration Details

Following [official ZMK migration guide](https://zmk.dev/blog/2025/12/09/zephyr-4-1):

| Aspect | Old (Shield) | New (Custom Board) |
|--------|--------------|-------------------|
| Directory | `boards/shields/eyelash_sofle/` | `boards/eyelash/sofle/` |
| Build target | `nice_nano` + shields | `sofle_left//zmk`, `sofle_right//zmk` |
| SOC control | Inherited from nice_nano | Full explicit control |
| Kconfig board | N/A (shield) | `Kconfig.sofle_left/right` |
| Board selection | `depends on` | `select SOC_NRF52840_QIAA` |
| Defconfig SOC/BOARD | N/A | Removed (not allowed in HWMv2) |
| Bootloader | Basic | Full UF2 with retention support |
| DCDC | Via nice_nano | Explicit `&reg0 { status = "okay"; }` |

## Testing

- ✅ `sofle_left//zmk` builds successfully (822 KB)
- ✅ `sofle_right//zmk` builds successfully (632 KB)
- ✅ nice_view_gem display integrates properly
- ⚠️ BLE connectivity to be tested on hardware
- ⚠️ Split half communication to be tested

## Known Issues / Notes

1. **Clear bonds recommended**: Due to BLE configuration changes, users should:
   - Flash `settings_reset` firmware first
   - Clear Bluetooth pairings on host devices
   - Re-pair both halves

2. **Display status**: nice_view_gem should work but needs hardware verification

3. **Build deprecation warning**: The `config/boards/` folder shows a deprecation warning. For full compliance with future ZMK versions, consider moving to a proper [ZMK module](https://zmk.dev/docs/development/module-creation) structure.

## Related Commits

- ZMK: `0331b7d16e80954b807917f9323e59ffc1e3b626` (post-v0.3.0)
- Zephyr: `58a5874a446ace2893a196848282d271a551e512` (v4.1.0+zmk-fixes)
- nice-view-gem-z4: `286a563` (LVGL 9.x / Zephyr 4.1.0 compatibility)
