# omlet-esphome

ESPHome firmware for the **Omlet Smart Automatic Chicken Coop Door** (WiFi control panel, board `X107 ADO ISSB3`).

The stock control panel is an ESP32-S3 that only talks to Omlet's cloud. This project replaces the firmware with
ESPHome so the door works fully locally with Home Assistant. The stock module has no secure boot and no flash
encryption, so it can be flashed over the serial header and restored from a backup at any time.

> **Status: works on a real door, still work in progress.**
> The pin map and protocol details were recovered by disassembling the stock firmware (1.0.55). Tested on a real
> door: LCD and backlight, motor direction, end stops and calibration, crush reversal and the coop light all work.
> Run it from the 12 V adapter only: the firmware does not sleep yet, so AA batteries would go flat within days.
> Battery operation with deep sleep is planned (see below).

## Usage

Everything board-specific lives in the `omlet-door.yaml` package. Your own config only needs a name, networking
and the package line:

```yaml
esphome:
  name: coop-door
  friendly_name: Coop Door

packages:
  omlet: github://swoboda1337/omlet-esphome/omlet-door.yaml@main

logger:

api:

ota:
  - platform: esphome

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
```

See `example.yaml`. Tunables such as `invert_direction`, `crush_inverted`, `lcd_contrast` and the stall timing are
substitutions at the top of the package; set them in your own `substitutions:` block to override.

The package provides:

- `cover` for the door (`omlet_door` platform): time-of-flight position, current-sensed end stops, crush switch
  cut in an interrupt, automatic reverse and retry on obstruction, gearbox relief after each end stop, brake when
  parked, and an `omlet_door.calibrate` action.
- `button` "Calibrate Door" that runs calibration (open to stop, close to stop, open to stop, learn the thresholds).
- `light` "Coop Light" on the COOP LIGHT connector.
- `sensor` for ambient light level and battery voltage, each powered up only while sampling.
- `binary_sensor` for the crush switch and 12 V mains presence.
- The three front buttons as up, down and OK, like the stock panel. At the home screen a tap on up toggles the
  coop light, a tap on down stops the door, and holding OK opens or closes it (hold up or down to pick a
  direction). A tap on OK opens a menu on the LCD showing the time, light level, battery, Wi-Fi status and a
  calibrate command; up and down scroll, OK selects, holding OK or thirty seconds of silence leaves it.
- `display` on the built-in LCD (`lcd_st7032` platform) showing door state and what the buttons do.
- `switch` "LCD Backlight". It comes on with any button press or door movement and goes off after
  `menu_timeout` seconds of inactivity, the same timeout the LCD menu uses.

## Flashing and recovery

1. Connect a 3.3 V USB-serial adapter to HDR2: GND to 0V, adapter TX to RX-I, adapter RX to TX-O. Do not connect
   VCC. Soldering a 3-pin header is the tidy way, but for a one-off flash you can just push three header pins (or
   the stripped ends of the jumper wires) into the holes and lean them to one side so they press against the
   plating; that is enough contact for the few seconds it takes.
2. Hold RESET, press RESTART (or apply power), release RESET.
3. Back up the stock flash before anything else:
   ```
   esptool --chip esp32s3 --port /dev/ttyUSB0 --baud 921600 read-flash 0 0x1000000 omlet-stock-backup.bin
   ```
   Keep this file private. It contains your Wi-Fi credentials and the device's Omlet API token.
4. Flash ESPHome as usual. To go back to stock, write the backup to offset 0 with `write-flash`.

## Hardware

- ESP32-S3-WROOM-1 N16R8 (16 MB flash, 8 MB PSRAM), 40 MHz crystal, 32.768 kHz crystal on GPIO15/16
- Power: 12 V barrel jack or 4×AA backup, two buck regulators
- Serial header `HDR2` (through-hole, next to the module): `0V / RX-I / TX-O`, 115200 8N1, 3.3 V logic
- Download mode: hold the **RESET** button (SW4, GPIO0) while pressing **RESTART** (SW5, EN) or applying power
- I2C character LCD (chip-on-glass, ST7032-class) 2×16 at address 0x3E
- Discrete H-bridge for the door motor built from two dual-MOSFET SOIC-8s with low-side shunts for current sensing
- 4-pin Molex "DOOR" (same as a 4-pin ATX plug): black pair = motor, yellow pair = crush switch
- 2-pin Molex "COOP LIGHT" for the Omlet coop light accessory
- 5-pin "I2C COMS" header `HDR4`: `V+ / NT / SCL / SDA / 0V` (shares the LCD I2C bus), see below
- 2-pin "Prod Test" header `HDR5` on GPIO42, see below

## Pin map

Recovered from the stock firmware's pin table and its call sites.

| GPIO | Function | Notes |
|------|----------|-------|
| 0  | RESET button (SW4) | Also the boot strap; hold at power-up for download mode |
| 1  | ADC1_CH0 ambient light sensor | Enabled by GPIO2; stock firmware reads it from the ULP during deep sleep |
| 2  | Light sensor enable | Active low; pulled low only while sampling |
| 3  | ADC1_CH2 battery voltage | Divider enabled by GPIO46 |
| 4  | I2C SCL | LCD and the I2C COMS header `HDR4` |
| 5  | I2C SDA | |
| 6  | LCD reset | Active low, pulsed at init |
| 7  | LCD power enable | Driven low at init, high to switch the display off (probably P-FET) |
| 8  | Coop light output | Plain push-pull GPIO |
| 9  | ADC1_CH8 motor current | Shunt amplifier, enabled by GPIO45 |
| 10 | ADC1_CH9 | Fifth analogue input, no enable pin; probably coop light current |
| 11 | H-bridge high-side B | Motor |
| 12 | H-bridge low-side A | Motor |
| 13 | H-bridge high-side A | Motor |
| 14 | H-bridge low-side B | Motor |
| 15 / 16 | 32.768 kHz crystal | |
| 17 | Middle button (down) | Active high, external pull-down |
| 18 | Right button (OK) | Active high, external pull-down |
| 21 | Left button (up) | Active high, external pull-down |
| 40 | Crush switch input | Yellow pair on the DOOR connector, interrupt driven |
| 42 | Prod Test header `HDR5` input | Low at boot enters production test mode in stock firmware; unused by this package |
| 43 / 44 | UART0 TX / RX | Serial header HDR2 |
| 45 | Motor current amplifier enable | Active high |
| 46 | Battery divider enable | Active high |
| 47 | LCD backlight | Active low, confirmed on the door |
| 48 | 12 V adapter present | Input, high when mains adapter is connected |

## Motor control

Only two of the four bridge pins are ever high at once:

| State | Pins high |
|-------|-----------|
| Open (stock "reverse") | GPIO11 + GPIO12 |
| Close (stock "forward") | GPIO13 + GPIO14 |
| Stop (brake) | GPIO12 + GPIO14 |

Stop shorts the motor through both low-side FETs and is held while parked. The stock firmware drives all four pins
low before raising the new pair. For the standard sliding door the pins are plain full-on GPIO outputs; the rotary
door variant uses 5 kHz PWM at 50 % forward and 70 % reverse.

On batteries the stock firmware runs a small ULP-RISC-V program while the main core sleeps: it polls the three
buttons every few milliseconds and samples the light sensor about once a second, waking the main core on a button
change or when the averaged light level crosses the open or close threshold for about twenty seconds. ESPHome has
no ULP support yet, so battery mode here will use timer wake instead.

Position is time-of-flight: calibration runs the door to each end stop using the current sense, records the travel
time, and compensates it with the measured battery voltage. The crush switch on GPIO40 (active low, external
pull-up) is never used as an end stop: any press while closing, including after the motor has stopped at the
bottom, reverses the door. The closed end is detected by current only, and a stall is treated as the end stop once
about 90 % of the calibrated travel time has passed, or within the first half second (door already at the end);
anything in between is an obstruction and the door reverses and retries. After reaching an end stop the stock
firmware waits a second and drives the motor the other way for 100 ms to unload the gearbox. The `omlet_door`
component mirrors all of this.

## LCD

ST7032-class controller at I2C 0x3E, 100 kHz, two lines of sixteen characters. Each write is a control byte
followed by payload: `0x00` commands, `0x40` data, `0x80` single command.

Init after the reset pulse on GPIO6:

```
0x38 0x39 0x13 0x76 0x55 0x68 0x38 0x0C 0x01 0x06
```

Cursor is `0x80 | col` for line 1 and `0xC0 | col` for line 2. Custom glyphs are loaded through CGRAM
(`0x40 | (index << 3)` then eight rows).

## Prod Test header (HDR5)

`HDR5` is the 2-pin through-hole header marked "Prod Test" on the front of the board: one pin is ground, the
other is GPIO42, which reads high unless the header is shorted. As far as the disassembly shows, the stock firmware
checks it at boot and, if it is shorted, runs Omlet's factory self-test instead of the normal application: it shows "PTest Started" on the LCD, then steps through a display test, a
button test, the light sensor and coop light current, motor current, sleep current and the crush switch, prints
every GPIO's configuration over the serial header, and ends with "PTest Passed" or "PTest Failed". While it runs
the firmware skips its usual coop-light detection ("not updating equipped status"). It is only useful with the
stock firmware and may rewrite calibration data in NVS, so take a flash backup before trying it.

This package does not use GPIO42, so the header is a free spare input. With the internal pull-up enabled a
switch or reed contact between its two pins reads low when closed:

```yaml
binary_sensor:
  - platform: gpio
    name: Spare Input
    pin:
      number: GPIO42
      mode:
        input: true
        pullup: true
      inverted: true
```

## I2C COMS header (HDR4)

`HDR4` is the 5-pin through-hole header marked "I2C COMS" on the back of the board, silkscreened pin 5 to 1 as
`V+ / NT / SCL / SDA / 0V`. SCL and SDA are the same bus the LCD uses (GPIO4 and GPIO5, 100 kHz, internal
pull-ups on), so any I2C sensor can be added to your config without touching the package, as long as it does not
use the LCD's address 0x3E:

```yaml
sensor:
  - platform: sht3xd
    temperature:
      name: Coop Temperature
    humidity:
      name: Coop Humidity
    address: 0x44
```

Two things are still unknown and worth checking with a meter before connecting anything. `V+` is probably the
3.3 V rail, but it has not been measured, so confirm it before powering a sensor from it. `NT` looks like an
interrupt line, but the stock firmware's pin table has no spare input assigned to it, so it is either an unrouted
pad or a GPIO the stock firmware never used; the production test's GPIO dump (see HDR5 above) would show which.
Keep leads short: the bus runs at 100 kHz and may have nothing but the ESP32's internal pull-ups.

## Planned

- Battery operation with deep sleep: decide at boot from the mains-detect pin, timer wake every ten minutes plus
  button wake on GPIO21/17/18, never sleep while the door is moving, compute sleep length from the next scheduled
  open or close, sample the light sensor on each wake.

## Partition layout of the stock firmware

| Name | Type | Offset | Size |
|------|------|--------|------|
| nvs | data | 0x9000 | 16 KiB |
| otadata | data | 0xD000 | 8 KiB |
| phy_init | data | 0xF000 | 4 KiB |
| factory_test | app | 0x10000 | 4032 KiB |
| ota_0 | app | 0x400000 | 5 MiB |
| ota_1 | app | 0x900000 | 5 MiB |
| littlefs | data | 0xE00000 | 2 MiB |
