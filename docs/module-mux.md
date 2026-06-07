# GeaconPolaris ModuleMux

GeaconPolaris uses zmk-input-module to build one firmware per half while supporting mutually-exclusive input modules.

## Profiles

Profile IDs live in include/dt-bindings/polaris/module_select.h.

| Profile | Left half | Right half | Notes |
| --- | --- | --- | --- |
| POLARIS_MODULE_UNSPECIFIED | yes | yes | Initializes no module candidate. |
| POLARIS_MODULE_ENC | yes | no | Left EC11 encoder candidate. |
| POLARIS_MODULE_JOY | yes | no | Left analog input plus EC11 encoder candidate. |
| POLARIS_MODULE_TB | yes | yes | PMW3610 trackball candidate. |
| POLARIS_MODULE_TPD | yes | yes | Cirque Pinnacle touchpad candidate. |
| POLARIS_MODULE_IQS | no | yes | Right IQS9151 candidate. IQS is exclusive on Polaris, not an optional companion module. |

The profile value is stored per MCU under polaris/module. Because zmk-input-module uses event-source locality, pressing a profile key on the physical left half updates the left MCU, and pressing a profile key on the physical right half updates the right MCU.

## Build Targets

Normal firmware targets are unified:

| Artifact | Shields | Snippets |
| --- | --- | --- |
| Polaris_L_UNIFIED | Polaris_L_Base rgbled_adapter nice_oled | Central ModuleMux zmk-usb-logging studio-rpc-usb-uart |
| Polaris_R_UNIFIED | Polaris_R_Base rgbled_adapter | Peripheral ModuleMux |
| settings_reset-seeeduino_xiao_ble | settings_reset | none |

Per-module snippets and temporary diagnostic targets have been removed from build.yaml. The active build matrix now contains only unified firmware and normal settings_reset.

## Deferred Init Strategy

config/west.yml pins the unified firmware dependencies to dedicated branches:

- te9no/zmk: codex/polaris-zmk-input-module-mux
- te9no/zephyr: codex/polaris-zephyr-deferred-init
- te9no/zmk-driver-iqs9151-rpc: codex/polaris-iqs-unified-firmware

The Zephyr branch provides global zephyr,deferred-init and device_init(dev).

snippets/ModuleMux/ModuleMux.overlay defines all candidate devices as compiled but deferred. zmk,input-module-mux restores the selected profile from settings and initializes only that profile's devices.

This is the important rule: unselected module candidates must not initialize their bus, pinctrl state, input listener source, or sensor route at boot.

## Shared Pin Handling

Polaris has strong pin and peripheral sharing. The current unified route is:

| Candidate | Current route | Reason |
| --- | --- | --- |
| TB | hardware spi2 + PMW3610 | TPD needs hardware i2c0, so TB cannot stay on spi0 because spi0 and i2c0 share the same nRF52840 peripheral instance. |
| TPD | hardware i2c0 + Cirque Pinnacle | gpio-i2c failed on real hardware; hardware i2c0 initialized the touchpad successfully. |
| IQS | hardware i2c1 + IQS9151 | IQS is right-side-only and exclusive. Polaris IQS pins are SDA=D4/P0.04, SCL=D8/P1.13, DR=D7/P1.12. It uses deferred i2c1 so TPD can keep deferred i2c0 in the same unified candidate graph. |

TPD and TB still share physical module-area pins with other candidates. The conflict is handled by deferred init and profile-gated input proxies, not by per-module firmware images.

## Keymap UX

config/Polaris.keymap exposes module profile selection in the BT layer.

Left-side profile keys:

- ENC
- JOY
- TB
- TPD
- UNSPECIFIED

Right-side profile keys:

- TB
- TPD
- IQS
- UNSPECIFIED

A changed profile is restored on the next boot. Reboot or power-cycle the relevant half after selecting a new profile.

## Verified

Build verified after cleanup:

- Polaris_L_UNIFIED builds successfully.
- Polaris_R_UNIFIED builds successfully.
- Generated artifacts were updated under firmware/zmk-config-GeaconPolaris/.

Hardware verified:

- Polaris_L_UNIFIED boots with saved TPD profile.
- ModuleMux initializes i2c@40003000.
- Cirque Pinnacle glidepoint_L@2a initializes successfully.
- USB endpoint reaches configured state.
- Split central connects to the right peripheral.
- Polaris_R_UNIFIED boots with saved IQS profile.
- ModuleMux initializes i2c@40004000.
- IQS9151 iqs9151@56 initializes successfully after RDY retries.

The success log is stored at docs/logs/2026-06-06-polaris-l-unified-tpd-success.log.

## Removed Diagnostic Scaffolding

The following temporary files were used during bring-up and removed after the issue was isolated:

- config/PolarisDiag.keymap
- snippets/CentralDiag/
- snippets/NoInputModule/
- snippets/NoSplitDiag/
- snippets/NoStudioNoOled/
- snippets/TpdHwI2cDiag/
- snippets/UsbOnlyDiag/
- src/tpd_hw_i2c_diag.c
- Polaris_L_DIAG_* build targets
- settings_reset_LOG build target

The diagnostic history is kept in docs/boot-diagnostics.md and docs/logs/.

## Remaining Hardware Checks

- Confirm left ENC profile reads the encoder pins correctly.
- Confirm left JOY profile reads ADC and encoder routes correctly.
- Confirm left and right TB profiles initialize PMW3610 through deferred spi2.
- Confirm right TPD profile initializes Pinnacle through deferred i2c0.
- Confirm right IQS profile input behavior, gestures, sensitivity, and startup delay after deferred hardware i2c1 initialization.
- Confirm UNSPECIFIED does not configure shared module pins in a way that affects the base keyboard.
- Confirm left and right profile settings can intentionally differ and survive cold boot.
