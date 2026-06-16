# direwatch
*by Craig Lamparter KM6LYW — MIT License*

Display [Direwolf](https://github.com/wb2osz/direwolf) APRS/packet radio information on a small TFT display (and/or save to a PNG file).

![Ygate with Direwatch](http://craiger.org/direwatch.png)

> **Newer demonstration (with symbols):** https://www.youtube.com/watch?v=NJ_IJNU7NA0&t=7s

---

## Table of Contents

- [Overview](#overview)
- [Supported Displays](#supported-displays)
- [Shopping List](#shopping-list)
- [Quick Start](#quick-start)
- [Usage](#usage)
- [Additional Scripts](#additional-scripts)
- [Documentation](#documentation)
- [Credits](#credits)

---

## Overview

`direwatch.py` tails a Direwolf log file and renders received APRS station callsigns and symbols on a small SPI TFT display attached to a Raspberry Pi. It can also save screen updates to a PNG file for use in web dashboards or remote viewing.

Features:
- Live APRS callsign and symbol display
- Single-station full-screen mode (`-o`) or scrolling list mode
- Distance and bearing to stations (when your coordinates are provided)
- Red/Green PTT and DCD status indicators (requires Direwolf `-d o` flag)
- Bluetooth connection indicator (blue icon)
- Physical GPIO LED support (GPIO 5 = blue, GPIO 26 = red)
- Optional PNG screenshot output

---

## Supported Displays

Three SPI TFT display types are supported:

| Driver   | Resolution | Product Link |
|----------|-----------|--------------|
| `st7789`  | 240×240   | [Adafruit Mini PiTFT 1.3"](https://www.adafruit.com/product/4484) |
| `ili9341` | 320×240   | [Adafruit 2.8" TFT](https://www.adafruit.com/product/2423) |
| `ili9486` | 480×320   | [Waveshare SpotPear Rev2.0 (B)](https://www.amazon.com/dp/B07V9WW96D) |

> **Note:** The default display type is `st7789`. Specify a different type with the `-d` option.  
> For ILI9486, only the Waveshare SpotPear Rev2.0 (B) is recommended — other ILI9486 screens have poor viewing angles.

Do **not** install the kernel module or framebuffer driver for these displays.

---

## Shopping List

Minimal parts for a PiZero 2W SDR APRS receive-only igate:

| Price | Item | Link |
|-------|------|------|
| $18 | Raspberry Pi Zero 2W with headers | [Adafruit](https://www.adafruit.com/product/6008) |
| $4  | USB OTG adapter cable | [Amazon](https://amazon.com/UGREEN-Adapter-Samsung-Controller-Android/dp/B00N9S9Z0G) |
| $31 | RTL-SDR dongle | [Amazon](https://amazon.com/RTL-SDR-Blog-RTL2832U-Software-Defined/dp/B0CD745394) |
| $32 | VHF antenna | [eBay](https://www.ebay.com/itm/321819895073) |
| $14 | ST7789 240×240 display | [Amazon](https://www.amazon.com/DIYmall-Display-240x240-Raspberry-ST7789/dp/B08F9VD2GZ) |

---

## Quick Start

### 1. Install dependencies (Raspberry Pi OS Bookworm)

```bash
sudo apt-get update
sudo apt-get install direwolf rtl-sdr git python3-pip fonts-dejavu \
    python3-pil python3-pyinotify python3-numpy python3-libgpiod python3-lgpio

sudo pip3 install --break-system-packages adafruit-circuitpython-rgb-display Adafruit-Blinka aprslib
```

### 2. Clone the repository

```bash
git clone https://github.com/craigerl/direwatch.git
cd direwatch
```

### 3. Enable SPI

```bash
sudo nano /boot/firmware/config.txt
```

Uncomment or add:
```
dtparam=spi=on
```

Then reboot.

### 4. Configure Direwolf

Create or edit `direwolf.conf` (example for an RTL-SDR receive-only igate):

```
MYCALL NOCALL
IGSERVER noam.aprs2.net
IGLOGIN NOCALL 12345
PBEACON sendto=IG compress=1 delay=00:15 every=30:00 symbol="igate" overlay=X lat=40.911 long=-122.935 comment="Direwatch Rx-only igate"
AGWPORT 8000
KISSPORT 8001
ADEVICE null
```

### 5. Run

```bash
# Start Direwolf piped from rtl_fm, logging to file
rtl_fm -s 22050 -g 49 -f 144.39M 2>/dev/null | direwolf -d o -t 0 -r 22050 - > direwolf.log &

# Start direwatch (single-station mode, ST7789 display)
./direwatch.py -o -l direwolf.log -t "APRS"
```

---

## Usage

```
usage: direwatch.py [-h] -l LOG [-f FONTSIZE] [-t TITLE_TEXT] [-o] [-y LAT] [-x LON] [-s SAVEFILE] [-d DISPLAY]

options:
  -h, --help                    Show this help message and exit
  -l LOG, --log LOG             Direwolf log file path (required)
  -f FONTSIZE, --fontsize FONTSIZE
                                Font size for callsigns (default: 30)
  -t TITLE_TEXT, --title_text TITLE_TEXT
                                Text displayed in the title bar (default: "Direwatch")
  -o, --one                     Show one station at a time full screen
  -y LAT, --lat LAT             Your latitude, e.g. 40.911
  -x LON, --lon LON             Your longitude, e.g. -122.935
  -s SAVEFILE, --savefile SAVEFILE
                                Save each screen update to this PNG file
  -d DISPLAY, --display DISPLAY
                                Display type: st7789 (default), ili9341, or ili9486
```

**Examples:**

```bash
# List mode, custom title, save PNG
direwatch.py -l /root/direwolf.log -t "APRS Digi" -f 20 -s /run/direwatch.png

# Single-station mode with distance/bearing, ILI9341 display
direwatch.py -o -l /root/direwolf.log -y 40.911 -x -122.935 -d ili9341

# ILI9486 large display
direwatch.py -o -l /root/direwolf.log -d ili9486 -t "KM6LYW"
```

**Red/Green status indicators** require the Direwolf `-d o` flag so that PTT/DCD events are written to the log.

---

## Additional Scripts

| Script | Description |
|--------|-------------|
| [`digibanner.py`](docs/scripts.md#digibannerpy) | Display a customizable splash screen on the TFT |
| [`digibuttons.py`](docs/scripts.md#digibuttonspy) | Monitor GPIO buttons to start/stop TNC and digipeater services |
| [`ILI9486.py`](docs/scripts.md#ili9486py) | Python SPI driver library for the ILI9486 display |

---

## Documentation

Detailed documentation is in the [`docs/`](docs/) folder:

- [Installation Guide](docs/installation.md)
- [Hardware & Wiring](docs/hardware.md)
- [Configuration Reference](docs/configuration.md)
- [Script Reference](docs/scripts.md)

---

## Credits

- Craig Lamparter KM6LYW — author
- [ladyada / Adafruit](https://learn.adafruit.com/adafruit-mini-pitft-135x240-color-tft-add-on-for-raspberry-pi/python-setup) — display driver examples
- [SirLefti/Python_ILI9486](https://github.com/SirLefti/Python_ILI9486) — ILI9486 Python library
- [hessu/aprs-symbols](https://github.com/hessu/aprs-symbols) — SVG APRS symbol set
