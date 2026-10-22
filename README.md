# Deadband Labs Syncro

|||
| :---: | :---: |
| ![Syncro rev 1](images/photo.jpg) | ![Syncro rev 1, KiCad render](images/render-iso.png) |

A USB HID force-feedback interceptor between a PC and FFB devices.

Based on the MYIR MYC-YF135 SOM (STM32MP135), fully USB bus powered, with a USB 2.0 hub,
two-position topology mux for downstream device passthrough, and three USB-A ports.

The USB mux supports two USB operating modes, switched at runtime by `USB_MODE_SEL`.
 - In Mode 1, the power-on default, the PC sees the SOM as a USB gadget and downstream ports
sit behind the hub on the SOM host port. Downstream ports are hidden from the PC.
 - In Mode 2, USB-C feeds the hub uplink and the SOM becomes a peer device, connecting the
downstream ports directly to the PC for management.

This board also serves as an STM32MP13 development board, with Qwiic and mikroBUS (unpopulated)
as GPIO expansion, SWD, UART debug header, user buttons, and an RGB status LED.

## Hardware

- STM32MP13, 1GHz Cortex A7 CPU, 256MB DDR3L RAM
- Boot from on-module 256MB SPI NAND, USB DFU on SW2, or SDMMC
- USB-C device port (fixed 5.1 kOhm Rd both CC pins, no PD or CC sensing)
- FE1.1s 4-port hub, three downstream ports on horizontal USB-A
- Per-port common-mode filter + ESD (ECMF02-2AMX6)
- Input eFuse (647 mA) + reverse blocking on the 5 V rail
- microSD slot (SDMMC1 through EMIF06)
- Console UART on a 1x3 0.1" header, SWD on a 1.27 mm 10-pin
- Qwiic as J9, mikroBUS/Click footprint as J8
- Reset, DFU and case/user buttons; discrete RGB status LED on TIM1
- 4 layer, 1.6 mm, ENIG, JLC04161H-3313; single-sided SMT assembly

Default assembly fits the Qwiic connector only. The console, SWD and mikroBUS headers
are left unpopulated, as are C10 and C11.

## Opening the design

Designed in KiCad 10. Open `syncro-mainboard/syncro-mainboard.kicad_pro` from a clone
of this repository. The footprint and symbol tables use paths relative to the project
(`${KIPRJMOD}/../syncro-pcb-lib/`), so no library setup is needed.

Vendor datasheets, vendor 3D models, fabrication outputs and internal DFT tools are not included.
This will produce missing-3D-model warnings, which do not affect the schematic, layout, DRC or plots.

Part numbers as assembled are in the LCSC fields on the schematic symbols.

## Interfaces

### Qwiic (J9)

JST SH 1.0 mm, 4 pin, horizontal. `GND`, `3V3`, `SDA` (`PH7`, I2C3), `SCL` (`PH3`, I2C3).

- Shares the I2C3 bus with the mikroBUS socket, 10 kOhm pull-ups fitted. Qwiic modules
  usually also fit pull-ups, so a long chain may over-pull the bus. Prefer a module
  without pull-ups or drop bus speed to 100 kHz to compensate.

### mikroBUS / Click (J8, DNP)

- Two 1x8 sockets at 22.86 mm row spacing, matching the mikroBUS spec. All
twelve IO signals are SOM GPIO, selected to best suit the functions below:

| Pin | mikroBUS | Board net | SOM pad | SOM port | Roles |
|---:|---|---|---:|---|---|
| 1 | `AN` | `AIO1` | 48 | PA1 | ADC1/2_INP3, TIM2_CH2, TIM5_CH2, TIM15_CH1N |
| 2 | `RST` | `RST` | 92 | PA5 | ADC1_INP2, TIM2_CH1 |
| 3 | `CS` | `SPI_CS` | 80 | PA4 | SPI1, ADC1_INP14, ClickID 1-Wire |
| 4 | `SCK` | `SPI_SCK` | 79 | PC3 | SPI1, I2S1_CK, ADC1_INP13 |
| 5 | `MISO` | `SPI_MISO` | 77 | PA6 | SPI1, ADC1_INP17, TIM3_CH1, TIM13_CH1 |
| 6 | `MOSI` | `SPI_MOSI` | 78 | PA3 | SPI1, I2S1_SDO, ADC1_INP12 |
| 7 | `3V3` | `3V3` | 16 | - | keep under 100 mA, no limit |
| 8 | `GND` | `GND` | - | - | |
| 9 | `PWM` | `AIO0` | 47 | PA2 | ADC1/2_INP1, TIM2_CH3, TIM5_CH3, TIM15_CH1 |
| 10 | `INT` | `INT` | 91 | PF11 | ADC1/2_INP8 |
| 11 | `RX` | `CAN_RX` | 96 | PB5 | FDCAN2, UART5_RX, TIM3_CH2 |
| 12 | `TX` | `CAN_TX` | 95 | PB13 | FDCAN2, UART5_TX, LPTIM2_OUT |
| 13 | `SCL` | `I2C_SCL` | 93 | PH3 | I2C3 |
| 14 | `SDA` | `I2C_SDA` | 94 | PH7 | I2C3 |
| 15 | `5V` | `SYS_5V` | - | - | board 5 V rail |
| 16 | `GND` | `GND` | - | - | |

- `AN` and `PWM` are `AIO1` (pad 48, PA1) and `AIO0` (pad 47, PA2) with a 330 Ohm series resistor.
`RX`/`TX` are routed so a CAN transceiver Click lands on FDCAN2 directly or used as a complete UART5 pair.

- Remaining GPIO use a 33 Ohm series resistor and share two 0.3 pF multi-channel ESD arrays.

### Console (J2) and SWD (J3)

| J2 pin | Signal |
|---:|---|
| 1 | `GND` |
| 2 | `UART4_TX` |
| 3 | `UART4_RX` |

- J3 is a standard ARM Cortex Debug 10-pin, 1.27 mm, pin 1 VTref on `VDD_3V3`,
SWDIO/SWCLK plus reset.

### USB

- J1, USB-C: device port, `CC1`/`CC2` each through 5.1 kOhm to GND. In mode 1 it
  reaches the SOM (gadget); in mode 2 it lands on hub port 1.
- J5-J7, USB-A: hub ports 2-4, shells to GND. The three ports share one VBUS current
  limit of about 250 mA (200-300 mA over tolerance), set by a TPS25200 with a 390 kOhm
  ILIM resistor.

### microSD (J4)

- SDMMC1 via EMIF06-MSD02N16 (45 ohm series, 20 pF, ESD, card-detect). Card detect
uses SOM pad 110 (`PD12`). Normally empty, meant for development use and as a possible
alternate boot target using a cut trace on the bottom of the board.

### Power budget

- A TPS25200 input eFuse with a 150 kOhm ILIM resistor limits the whole board to 647 mA nominal
(590-710 mA over tolerance). The USB-A ports are further limited to use up to 250 mA, leaving
about 400 mA for the SOM (measued 128 mA average, ~160 mA peak), the hub IC (30-80 mA) and
everything else. A PC USB 2.0 port only guarantees 500 mA. Downstream devices should be
self-powered for best reliability.

- Power for GPIO peripherals comes from the SOM's 3V3 output, shared with the rest of the board.
There is no limiting efuse on these, pins. To avoid brownout, keep the total under 100 mA.

## Software

The board's OS (TF-A, OP-TEE, U-Boot, the Linux kernel and the rootfs) is built with
the Deadband Labs Buildroot SDK: <https://github.com/deadbandlabs/dbl-buildroot>.
This builds a complete image from upstream sources that boots the MYIR SOM, with
the console, USB gadget and host, SPI NAND and SD card enabled in the device tree.

dbl-buildroot is designed to be extended by a parent project. A parent device tree
include is applied on top of the base tree, and a `BR2_EXTERNAL` tree adds packages,
overlays and defconfigs, see its
[Parent Integration](https://github.com/deadbandlabs/dbl-buildroot/wiki/7.-Parent-Integration)
page. Carrier board peripherals (USB hub control, Qwiic and I2C, mikroBUS and so on) are
implemented using this integration. The Syncro userspace and its board layer are pending
final publication and will be linked here when complete.

## Licence

Design files: CERN-OHL-W-2.0, see `LICENSE` and `NOTICE`.

The interfaces a third party uses to reach this board (the USB ports, Qwiic,
mikroBUS/Click, the console serial header and SWD) are documented above. A design
that uses the board only through them is External Material under section 1.8 of
the licence and does not need to be released. Modified design files must be
published under the same licence.

`syncro-pcb-lib/` is generated from the project's parts library (see
`syncro-pcb-lib/GENERATED.md`). Part of it derives from the KiCad libraries,
which stay CC-BY-SA-4.0 with the KiCad libraries exception. See `NOTICE`.
