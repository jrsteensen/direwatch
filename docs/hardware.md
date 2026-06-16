# Hardware & Wiring

This page describes the supported TFT displays, their GPIO wiring to the Raspberry Pi, and the GPIO pins used for status LEDs and optional physical buttons.

---

## Supported TFT Displays

### ST7789 — 240×240 (default)

The Adafruit [Mini PiTFT 1.3"](https://www.adafruit.com/product/4484) is a 240×240 pixel SPI TFT designed to plug directly onto the Raspberry Pi GPIO header.

- **Resolution:** 240×240
- **Interface:** SPI
- **Buttons:** 2 physical buttons on GPIO 23 and 24
- **Driver:** `adafruit-circuitpython-rgb-display` (included in `st7789` module)

This is the **default** display type; no `-d` flag is needed.

---

### ILI9341 — 320×240

The Adafruit [2.8" TFT Touch Shield](https://www.adafruit.com/product/2423) is a larger landscape display.

- **Resolution:** 320×240 (landscape, rendered as 240 wide × 320 tall in portrait then rotated)
- **Interface:** SPI
- **Buttons:** 2 physical buttons
- **Driver:** `adafruit-circuitpython-rgb-display` (included in `ili9341` module)
- **Flag:** `-d ili9341`

---

### ILI9486 — 480×320

The [Waveshare SpotPear Rev2.0 (B)](https://www.amazon.com/dp/B07V9WW96D) is the largest supported display.

- **Resolution:** 480×320 (landscape)
- **Interface:** SPI (via `spidev`)
- **Buttons:** None
- **Driver:** `ILI9486.py` (bundled in this repository)
- **Flag:** `-d ili9486`

> **Important:** Only the Waveshare SpotPear Rev2.0 (B) is recommended. Other ILI9486-based screens often have very poor viewing angles.

---

## GPIO Wiring

### ST7789 and ILI9341 (Adafruit displays)

These displays are designed to plug directly onto the 40-pin GPIO header of a Raspberry Pi. No manual wiring is required when using the standard Adafruit display HAT.

The display drivers in direwatch use these default SPI pins:

| Signal | BCM GPIO | Physical Pin |
|--------|----------|-------------|
| SPI MOSI | GPIO 10 | Pin 19 |
| SPI SCLK | GPIO 11 | Pin 23 |
| SPI CE0  | GPIO 8  | Pin 24 |
| DC       | GPIO 25 | Pin 22 |
| CS       | GPIO 4  | Pin 7  |
| 3.3V     | —       | Pin 17 |
| GND      | —       | Pin 20 |

---

### ILI9486 (Waveshare SpotPear via spidev)

The ILI9486 display is connected over SPI using the Linux `spidev` interface directly, with two additional GPIO control pins:

| Signal | BCM GPIO | Physical Pin | Notes |
|--------|----------|-------------|-------|
| SPI MOSI | GPIO 10 | Pin 19 | Hardware SPI |
| SPI SCLK | GPIO 11 | Pin 23 | Hardware SPI |
| SPI CE0  | GPIO 8  | Pin 24 | Chip Select via spidev(0,0) |
| DC       | GPIO 24 | Pin 18 | Data/Command select |
| RST      | GPIO 25 | Pin 22 | Reset |
| 3.3V     | —       | Pin 17 | |
| GND      | —       | Pin 20 | |

The ILI9486 SPI bus runs at up to 48 MHz (`spi.max_speed_hz = 48000000`).

---

## Status LEDs (GPIO outputs)

direwatch drives two physical LEDs via GPIO to mirror the on-screen PTT/DCD status indicators:

| Colour | BCM GPIO | Physical Pin | Function |
|--------|----------|-------------|---------|
| Blue   | GPIO 5   | Pin 29      | Bluetooth connected |
| Red    | GPIO 26  | Pin 37      | PTT active (transmitting) |

Connect an LED and current-limiting resistor (≈ 330 Ω) between each GPIO pin and ground.

---

## Optional Physical Buttons (digibuttons.py)

The [`digibuttons.py`](scripts.md#digibuttonspy) script monitors two push-buttons wired to:

| Function | BCM GPIO | Physical Pin |
|----------|----------|-------------|
| Toggle TNC service | GPIO 23 | Pin 16 |
| Toggle digipeater service | GPIO 24 | Pin 18 |

Buttons should connect the GPIO pin to GND when pressed. Internal pull-ups are enabled by the script.

> **Note:** GPIO 23 and 24 are also the button pins on the Adafruit Mini PiTFT. This is intentional — the two hardware buttons on that display are wired to those pins.

---

## Raspberry Pi GPIO Header Reference

```
         3V3  (1) (2)  5V
       GPIO2  (3) (4)  5V
       GPIO3  (5) (6)  GND
       GPIO4  (7) (8)  GPIO14
         GND  (9) (10) GPIO15
      GPIO17 (11) (12) GPIO18
      GPIO27 (13) (14) GND
      GPIO22 (15) (16) GPIO23  ← Button / digibuttons
        3V3  (17) (18) GPIO24  ← Button / digibuttons / ILI9486 DC
      GPIO10 (19) (20) GND
       GPIO9 (21) (22) GPIO25  ← ILI9486 RST / ST7789 DC
      GPIO11 (23) (24) GPIO8
         GND (25) (26) GPIO7
       GPIO0 (27) (28) GPIO1
       GPIO5 (29) (30) GND    ← Blue LED
       GPIO6 (31) (32) GPIO12
      GPIO13 (33) (34) GND
      GPIO19 (35) (36) GPIO16
      GPIO26 (37) (38) GPIO20  ← Red LED
         GND (39) (40) GPIO21
```
