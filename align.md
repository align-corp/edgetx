# Custom TX: Development Spec

Handheld radio based on an EdgeTX fork, B&W `taranis` target family. Prices are approximate, at about 1,000 units.

**Constraints**
- No Chinese silicon.
- Dust-sealed case for outdoor use.
- No voice or speaker audio. A buzzer is required for alarms, battery low first.
- Firmware updates from a computer only.

---

## 1. Hardware

### 1.1 Blocks

| Block | Choice | Notes |
|---|---|---|
| MCU | STM32F407VGT6 (LQFP100, 1 MB flash, 192 KB RAM) | About 45 of the 80 I/O used |
| Storage | Winbond W25Q64JVSSIQ, 8 MB SPI NOR | Second source: Macronix MX25L6433F |
| Display | 128x64 transflective FSTN COG, ST7565/ST7567, 4-wire SPI | Readable in sunlight with the backlight off. |
| Buzzer | Magnetic transducer driven from the MCU DAC (PA4) through an AC-coupled NPN stage | Required for alarms, battery low first. Uses EdgeTX's built-in tones, no voice files |
| Keys | 6 navigation keys (UP, DOWN, LEFT, RIGHT, ENTER, EXIT) plus power | No rotary encoder (it would be a dust path) |
| USB and charging | USB-C 5 V sink, TI BQ24075 power path, 1S Li-ion | Runs and charges from any USB source, no USB PD |
| 3.3 V rail | TI TPS63001 buck-boost | Cell voltage swings from 4.2 V to 3.0 V, across 3.3 V |
| RF | Plug-in daughterboard on a 10-pin connector, CRSF over UART | mLRS or ELRS board chosen per SKU |
| Gimbals | Not selected yet | Non-Chinese, target under $50 per stick; one axis per stick is acceptable |

### 1.2 Interfaces and pins

**NOR flash**
- SPI2 on PB12 (CS), PB13 (SCK), PB14 (MISO), PB15 (MOSI): the pins the taranis radios use for their SD card.
- DMA1 Stream 3 (RX) and Stream 4 (TX), channel 0. Check against RM0090.

**LCD**
- SCK, MOSI, CS, A0, RESET, plus one PWM pin for the backlight.
- Glass can be mounted either way up (`LCD_VERTICAL_INVERT` / `LCD_HORIZONTAL_INVERT`).
- Compare the controller init sequence with the chosen module's datasheet before layout.

**Buzzer**
- DAC output PA4, sample timer TIM6, DMA1 Stream 5: the stock taranis audio pins (`AUDIO_OUTPUT_GPIO`, `AUDIO_DMA_Stream`, `AUDIO_TIMER`). No conflict with the NOR flash DMA streams.
- PA4 → series capacitor (about 1 µF) → base resistor → NPN, with the transducer and a flyback diode from VSYS or 3.3 V to the collector.
- The coupling capacitor is required. The DAC idles at mid-scale, so a DC-coupled stage would hold current through the coil whenever the radio is silent.
- Transducer resonance about 2.7–4 kHz. The battery-low alarm sweeps 1950–2550 Hz and the default beep is in the same range.
- `AUDIO_MUTE_GPIO` is optional; leave it out unless idle noise is audible.

**USB-C**
- 5.1 kΩ pull-down on each of CC1 and CC2. Without both, C-to-C cables supply no power.
- One low-capacitance TVS array on D+, D− and VBUS (USBLC6-2SC6).
- VBUS divider to a GPIO for USB detection (`USB_GPIO_VBUS`).

**Charger**
- BQ24075 CHG output (open-drain) to `USB_CHARGER_GPIO`.
- Input current limit: 500 mA (always within USB rules) or about 1 A (USB-C sources only); not decided.

**Battery**
- One 1S Li-ion cell: 18650 (Molicel INR-18650-M35A) or 21700 (Molicel INR-21700-P45B); not decided. The case and holder follow the choice.
- Protection with TI BQ29700 and a dual N-FET.
- Resistor divider to an ADC pin for battery voltage.

**Power**
- The power button wakes the regulator and the MCU holds it on through a GPIO.
- USB VBUS must also enable the 3.3 V regulator. In the ROM DFU bootloader the MCU pins are inputs, so without this the radio would switch off during an update.

**Firmware recovery**
- BOOT0 test pad to enter the STM32 ROM DFU bootloader.

### 1.3 RF daughterboard connector (2x5, 1.27 mm, Harwin M50)

| Pin | Signal | EdgeTX macro | Note |
|---|---|---|---|
| 1, 2 | VSYS (3.5 to 4.4 V, switched) | `INTMODULE_PWR_GPIO` drives a TPS22918 load switch | Budget 1.5 A; module regulates locally |
| 3, 4 | GND | | |
| 5 | UART TX, MCU to module | `INTMODULE_USART` TX | 3.3 V logic |
| 6 | UART RX, module to MCU | `INTMODULE_USART` RX | 3.3 V logic |
| 7 | BOOT | `INTMODULE_BOOTCMD_GPIO` | ESP32 GPIO0 on ELRS; STM32 BOOT0 on mLRS |
| 8 | NRST | Spare GPIO | Lets the handset reset the module for flashing |
| 9 | +3V3 (100 mA) | | Logic reference only |
| 10 | MODULE_ID | ADC pin, optional | Resistor to GND identifies the fitted module |

- The antenna connector and all RF matching stay on the daughterboard. Each module board needs its own radio approval, but the main board doesn't.
- Battery voltage on VSYS lets a 1 W module draw peak current without a boost converter on the main board.

| Variant | Key parts | Board cost |
|---|---|---|
| mLRS 2.4 GHz | STM32G431/G491, Semtech SX1281, TCXO, PA/LNA | $12–15 |
| mLRS 868/915 MHz | STM32WLE5CC (radio on-chip), TCXO, RF switch | $6–8 |
| ELRS 2.4 GHz | ESP32 (Espressif, China), SX1281, TCXO, PA/LNA | $10–13 |

The ELRS variant breaks the no-Chinese-silicon rule. Use it only for SKUs where that's acceptable.

### 1.4 Sealing

- **Keys:** one moulded silicone keymat over C&K KSC IP67 tact switches, clamped between the case and the PCB. The power button sits under the same mat.
- **Buzzer:** sound port covered by an acoustic membrane vent (e.g. Gore acoustic vent, US) so the case stays dust-sealed.
- **USB-C:** either an IP67 receptacle with a panel gasket, or a standard receptacle behind a tethered silicone plug.

### 1.5 Main board BOM

| # | Function | Part number | Maker (origin) | Qty | Unit (USD) |
|---|---|---|---|---|---|
| 1 | MCU | STM32F407VGT6 | ST (EU) | 1 | 6.50 |
| 2 | Storage, 8 MB SPI NOR | W25Q64JVSSIQ | Winbond (Taiwan) | 1 | 0.50 |
| 3 | LCD 128x64 FSTN, ST7567, with backlight | Catalogue COG module | Glass China, controller Taiwan | 1 | 2.50 |
| 4 | USB-C receptacle, 16-pin USB 2.0 | USB4105-GF-A | GCT (UK) | 1 | 0.70 |
| 5 | Charger with power path | BQ24075RGTR | TI (US) | 1 | 1.60 |
| 6 | 3.3 V buck-boost | TPS63001DRCR | TI (US) | 1 | 1.50 |
| 7 | Inductor 2.2 µH for item 6 | Shielded, 1.5 A or more | TDK or Murata (Japan) | 1 | 0.25 |
| 8 | USB ESD protection | USBLC6-2SC6 | ST (EU) | 1 | 0.20 |
| 9 | RF module load switch, 2 A | TPS22918DBVR | TI (US) | 1 | 0.35 |
| 10 | Sealed tact switch, IP67 | KSC series, e.g. KSC241J | C&K (US) | 7 | 0.45 |
| 11 | Crystals: HSE and 32.768 kHz | To match the cloned target | Abracon or Epson | 2 | 0.20 |
| 12 | RF module connector, 2x5, 1.27 mm, mated pair | M50 series | Harwin (UK) | 1 | 1.20 |
| 13 | Li-ion cell, 18650 or 21700 | INR-18650-M35A or INR-21700-P45B | Molicel (Taiwan) | 1 | 4.00–5.00 |
| 14 | Cell holder, PCB mount, to match the cell | 1042 (18650) or 21700 equivalent | Keystone (US) | 1 | 2.50 |
| 15 | Cell protection IC and FETs | BQ29700 + dual N-FET | TI (US) | 1 | 0.50 |
| 16 | Silicone keymat, 7 keys | Custom moulding | To source | 1 | 0.80 |
| 17 | Buzzer, SMD magnetic transducer, about 4 kHz | CMT-8504-100-SMT or similar | CUI Devices (US) | 1 | 0.60 |
| 18 | Acoustic membrane vent for the buzzer port | Gore acoustic vent | Gore (US) | 1 | 0.40 |
| 19 | Passives, buzzer NPN and coupling capacitor, backlight FET, CC resistors, dividers | | | 1 set | 2.00 |
| 20 | PCB, 4-layer, about 100 x 60 mm | | | 1 | 2.50 |

Total: about **$32**. Gimbals, switches, enclosure and the RF daughterboard aren't included.

One-off costs: keymat tooling, and custom LCD glass if no catalogue module fits.


---

## 2. Firmware

### 2.1 Base

- One new PCBREV in `targets/taranis` with STM32F407xG, 128x64 LCD, CROSSFIRE internal module. Screen, MCU and internal module match, so the clone is mostly pin changes.
- Internal module set up with `DEFAULT_INTERNAL_MODULE CROSSFIRE` and `INTERNAL_MODULE_SERIAL`.
- The module baud rate is set per SKU in the factory `radio.yml` (`internalModuleBaudrate`), so one firmware covers all SKUs.

### 2.2 Work items

| Item | Work | Effort |
|---|---|---|
| Board | New PCBREV block in `CMakeLists.txt`, pin blocks in `hal.h` | 1–2 days |
| Storage (SPI NOR) | See 2.3 | 3–5 days including bench tests |
| LCD | Contrast offset, invert flags, check the init sequence | 0.5 day |
| Keys | Key pin defines only. Defining `KEYS_GPIO_REG_LEFT` selects the stock `NAVIGATION_9X` mapping | 0.5 day |
| USB and charger | `USB_GPIO_VBUS`, `USB_CHARGER_GPIO`; no driver changes | hours |
| RF daughterboard | `INTMODULE_*` pins, CROSSFIRE default, serial passthrough enabled | 0.5 day |
| Buzzer | Keep the stock taranis DAC audio driver (`audio_dac_driver.cpp`); no code changes. Set factory defaults in `radio.yml`: `beepMode` (alarms only, or alarms and keys) and `vBatWarn` for a 1S cell (the stock 1S `BATTERY_WARN` is 3.5 V) | hours |
| Firmware update | See 2.4 | 0–3 days, depending on the option |

### 2.3 Storage

**What's stored:** `RADIO/radio.yml`, `MODELS/`, and Lua scripts under `SCRIPTS/`. Expected use is a few hundred KB.

**What isn't:** logs, voice/sound files (`SOUNDS/`; the buzzer uses built-in tones), firmware files, or RF module firmware.

**Integration**
1. In `hal.h`, define `STORAGE_USE_SPI_FLASH` for the new PCBREV in place of the hard-coded `STORAGE_USE_SDCARD_SPI` (line 778).
2. Add the `FLASH_SPI*` pin and DMA defines, using the PL18 `hal.h` block as a template.
3. Add `spi_flash.cpp`, `diskio_spi_flash.cpp` and `drivers/frftl.cpp` to the firmware build.
4. Add `-DSTORAGE_USE_SPI_FLASH -DSPI_FLASH`. `SPI_FLASH` lowers the minimum free space from 50 MB to 2 MB; without it an 8 MB volume always reads as full.

Everything above the driver layer (YAML settings and models, Lua, USB mass storage) needs no changes.

**RAM.** FrFTL allocates about 33 KB of heap for 8 MB (about 16 KB if a 4 MB W25Q32 is used). The heap must be in main SRAM, not CCM, because flash reads use DMA and DMA can't reach CCM on the F407.

**Factory defaults.** Compile each SKU's `radio.yml`, default models and Lua scripts into the firmware, and write any missing file to NOR at boot. Then there's no file-copy step at the factory, and a blank or corrupted chip repairs itself on the next boot.

### 2.4 Firmware update

There's no `FIRMWARE/` folder; updates come over USB from a computer only. The bootloader doesn't need the NOR driver.

- **Recovery path (always available):** STM32 ROM DFU through the BOOT0 button. Works with EdgeTX Companion, EdgeTX Buddy, dfu-util and STM32CubeProgrammer.
- **Normal path, option A:** ROM DFU only. Patch the classic bootloader to remove "Write Firmware" and its USB mass-storage mode.
- **Normal path, option B:** add USB DFU to the EdgeTX bootloader. The DFU class, `usbRegisterDFUMedia()`, `boot_dfu.cpp` and the F4 sector map in `stm32_flash_driver` already exist, so the new code is an internal-flash DFU backend.
  - Layout string, assuming a 32 KB bootloader: `"@Internal Flash /0x08008000/02*016Kg,01*064Kg,07*128Kg"`.
  - Ignore writes below `BOOTLOADER_SIZE`, so tools can flash the full `.bin` at 0x08000000.
  - Report a long enough poll timeout for 128 KB sector erases.
  - Optional: check the board name and a CRC before jumping, so the wrong SKU's image is refused.
  - About 2–3 days.
- **RF modules:** flashed from the PC through EdgeTX serial passthrough.

### 2.5 Product features

Self-provisioning presets (implemented through the factory defaults in 2.3), settings lock, bind gesture and telemetry UI, as planned.

Boot combos, to be confirmed:

| Function | Combo |
|---|---|
| Bootloader | LEFT + RIGHT held at power-on |
| Settings unlock | UP + DOWN held at power-on |
| Bind | ENTER held 2 s on the main view |

---

## 3. Open items

**Hardware**
- [ ] Select gimbals and toggle switches. They also fix the remaining ADC and GPIO pin count.
- [ ] Pick the catalogue 128x64 ST7567 COG module.
- [ ] Choose the cell: 18650 or 21700. It sets the holder, case size and `vBatWarn` default.
- [ ] Choose the charge input limit: 500 mA or about 1 A.
- [ ] Choose USB-C sealing: IP67 receptacle or tethered plug.
- [ ] Confirm the HSE crystal frequency from the cloned target's clock setup.
- [ ] Check that VBUS enables the 3.3 V regulator independently of the power-hold GPIO.
- [ ] Place the BOOT0 button or pad under the keymat.
- [ ] Pick the buzzer transducer and check loudness through the sealed sound port, outdoors.
- [ ] Check distributor stock and ordering codes for the C&K switches and the Harwin connector pair.

**Firmware**
- [ ] Bench-test W25Q64JVSSIQ, and then MX25L6433F, with the EdgeTX driver on the F407 before layout freeze.
- [ ] Measure FrFTL heap use; confirm the heap is in main SRAM, not CCM.
- [ ] Test blank-chip first boot and power loss during saves.
- [ ] Bench-test the battery-low alarm on the buzzer: tone, level, repeat rate and `vBatWarn` threshold for the chosen cell.
- [ ] Decide the firmware update path: option A or B.
- [ ] Confirm how mLRS enters bind from the handset, and whether it accepts EdgeTX's CRSF bind frame.
- [ ] Confirm mLRS module firmware can be updated through serial passthrough.
