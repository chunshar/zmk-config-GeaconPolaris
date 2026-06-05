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

Normal firmware targets are now unified:

| Artifact | Shields | Snippets |
| --- | --- | --- |
| Polaris_L_UNIFIED | Polaris_L_Base rgbled_adapter nice_oled | Central ModuleMux zmk-usb-logging studio-rpc-usb-uart |
| Polaris_R_UNIFIED | Polaris_R_Base rgbled_adapter | Peripheral ModuleMux zmk-usb-logging studio-rpc-usb-uart |
| settings_reset-seeeduino_xiao_ble | settings_reset | none |

Per-module snippets have been removed. build.yaml now uses the unified ModuleMux route only.

## Deferred Init Strategy

config/west.yml pins te9no/zephyr revision af6fff80212a92f56c6ca9a3a339ab4957a85334. That revision provides global zephyr,deferred-init and device_init(dev).

snippets/ModuleMux/ModuleMux.overlay defines all candidate devices as compiled but deferred. zmk,input-module-mux restores the selected profile from settings and initializes only that profile's devices.

## Shared Pin Handling

Polaris has strong pin and peripheral sharing:

- TB uses spi0.
- TPD uses a software gpio-i2c bus.
- IQS also uses a software gpio-i2c bus.

IQS originally used hardware i2c0, but that conflicts with spi0 on nRF52840 because both map to the same peripheral instance. Unified firmware therefore uses gpio-i2c for both TPD and IQS candidates.

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

- ./just.sh init config/zmk-config-GeaconPolaris completes with zmk-input-module at 7237702 and Zephyr build af6fff80212a.
- ./just.sh build all completes successfully.
- Generated artifacts:
  - firmware/zmk-config-GeaconPolaris/Polaris_L_UNIFIED.uf2
  - firmware/zmk-config-GeaconPolaris/Polaris_R_UNIFIED.uf2
  - firmware/zmk-config-GeaconPolaris/settings_reset-seeeduino_xiao_ble.uf2

## Remaining Hardware Checks

- Confirm left ENC profile reads the encoder pins correctly.
- Confirm left JOY profile reads ADC and encoder routes correctly.
- Confirm left and right TB profiles initialize PMW3610 through deferred spi0.
- Confirm left and right TPD profiles initialize Pinnacle through deferred gpio-i2c.
- Confirm right IQS profile initializes IQS9151 through deferred gpio-i2c.
- Confirm UNSPECIFIED does not configure shared module pins in a way that affects the base keyboard.
- Confirm left and right profile settings can intentionally differ and survive cold boot.
