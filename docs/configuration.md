# Configuration Reference

This page documents all command-line options for `direwatch.py`, display type behaviour, font sizing guidelines, and the Direwolf configuration required for full functionality.

---

## direwatch.py — Command-line Options

```
usage: direwatch.py [-h] -l LOG [-f FONTSIZE] [-t TITLE_TEXT] [-o]
                    [-y LAT] [-x LON] [-s SAVEFILE] [-d DISPLAY]
```

### Options

#### `-l LOG` / `--log LOG` *(required)*

Path to the Direwolf log file that direwatch will tail.

```bash
direwatch.py -l /root/direwolf.log
direwatch.py -l /run/direwolf.log
```

Direwolf must be running and writing to this file. direwatch uses `tail -F` internally, so it will wait for the file and follow it across log rotations.

---

#### `-f FONTSIZE` / `--fontsize FONTSIZE`

Font size (in points) used for callsigns in **list mode**. Default is `30`.

| Value | Approximate lines visible (240px height) |
|-------|-----------------------------------------|
| 20    | ~8 lines |
| 30    | ~6 lines (default) |
| 33    | ~5 lines (maximum readable width) |

This option has no effect in single-station mode (`-o`), which uses fixed font sizes.

```bash
direwatch.py -l direwolf.log -f 20
```

---

#### `-t TITLE_TEXT` / `--title_text TITLE_TEXT`

Text to display in the title bar at the top of the screen. Default is `"Direwatch"`.

```bash
direwatch.py -l direwolf.log -t "KM6LYW igate"
```

---

#### `-o` / `--one`

Enable **single-station mode**. Instead of a scrolling list of callsigns, the display shows one station at a time with its APRS symbol, callsign, distance/bearing (if coordinates are set), and comment text.

The display updates each time a new packet is received.

```bash
direwatch.py -o -l direwolf.log
```

---

#### `-y LAT` / `--lat LAT`

Your station latitude as a decimal number (negative for South).

```bash
direwatch.py -o -l direwolf.log -y 40.911 -x -122.935
```

When both `-y` and `-x` are provided, direwatch calculates the distance (in miles) and bearing from your position to each received station and displays this information in single-station mode.

---

#### `-x LON` / `--lon LON`

Your station longitude as a decimal number (negative for West).

---

#### `-s SAVEFILE` / `--savefile SAVEFILE`

Path to a PNG file where each screen update is saved. Useful for displaying live status on a web page or remote monitor.

```bash
direwatch.py -l direwolf.log -s /var/www/html/direwatch.png
```

The file is written with `compress_level=1` for speed.

---

#### `-d DISPLAY` / `--display DISPLAY`

Select the TFT display driver. Must be one of:

| Value | Display | Resolution |
|-------|---------|-----------|
| `st7789` | Adafruit Mini PiTFT 1.3" | 240×240 (default) |
| `ili9341` | Adafruit 2.8" TFT | 320×240 |
| `ili9486` | Waveshare SpotPear Rev2.0 (B) | 480×320 |

```bash
direwatch.py -l direwolf.log -d ili9341
```

If `-d` is not specified, `st7789` is used.

---

## Display Modes

### List Mode (default)

Shows a scrolling grid of received station callsigns with their APRS symbols. New stations are appended; duplicates blink briefly. When the screen fills, it pauses for 2 seconds then clears and starts over.

### Single-Station Mode (`-o`)

Shows one station at a time, full screen, with:
- APRS symbol (large, upper-left area)
- Callsign (large text, bottom of screen)
- Distance and bearing from your position (if `-y`/`-x` provided)
- Comment or status text from the packet

Supported packet formats: `mic-e`, `compressed`, `uncompressed`, `object`, `weather`, `status`.

---

## Status Indicators

### On-screen indicators (title bar)

| Indicator | Colour | Meaning |
|-----------|--------|---------|
| Right circle | Green (bright) | DCD active (signal detected) |
| Right circle | Green (dim) | DCD inactive |
| Middle circle | Red (bright) | PTT active (transmitting) |
| Middle circle | Red (dim) | PTT inactive |
| Bluetooth icon | Visible | Bluetooth device connected |

These indicators require Direwolf to be started with `-d o` so that `DCD` and `PTT` events appear in the log file.

### Physical LEDs

| Colour | GPIO | Condition |
|--------|------|-----------|
| Blue | GPIO 5 | Bluetooth device connected |
| Red | GPIO 26 | PTT active |

See [Hardware & Wiring](hardware.md) for wiring details.

---

## Direwolf Configuration

For full direwatch functionality, Direwolf must:

1. **Write a log file** — pipe or redirect stdout to a file:
   ```bash
   direwolf ... > /run/direwolf.log
   ```

2. **Enable PTT/DCD logging** with the `-d o` flag:
   ```bash
   direwolf -d o -t 0 ...
   ```
   Without this flag, the red/green indicators will never change state.

3. **Set PTT/DCD GPIO pins** in `direwolf.conf` if you want Direwolf to control a transmitter:
   ```
   PTT GPIO 12
   DCD GPIO 16
   ```
   direwatch reads the PTT/DCD state from the log file lines, not directly from GPIO.

### Example direwolf.conf (receive-only igate)

```
MYCALL N0CALL-10
IGSERVER noam.aprs2.net
IGLOGIN N0CALL-10 12345
PBEACON sendto=IG compress=1 delay=00:15 every=30:00 symbol="igate" overlay=R lat=40.911 long=-122.935 comment="Direwatch SDR igate"
AGWPORT 8000
KISSPORT 8001
ADEVICE null
```

### Example direwolf.conf (digipeater with TNC)

```
MYCALL N0CALL-1
ADEVICE plughw:1,0
ACHANNEL 0
MODEM 1200
PTT GPIO 12
DIGIPEAT 0 0 ^WIDE[3-7]-[1-7]$ ^WIDE[12]-[12]$
AGWPORT 8000
KISSPORT 8001
```

---

## digibanner.py — Command-line Options

```
usage: digibanner.py [-h] [-f FONTSIZE] [-b BIG] [-s SMALL] [-t TINY] [-g GRAPHIC] [-d DISPLAY]
```

| Option | Description | Default |
|--------|-------------|---------|
| `-f FONTSIZE` | Font size | 30 |
| `-b BIG` | Large text (upper area) | `"DigiPi"` |
| `-s SMALL` | Medium text (below big) | `""` |
| `-t TINY` | Small text (bottom) | `""` |
| `-g GRAPHIC` | Path to a PNG image to display (right side) | None |
| `-d DISPLAY` | Display type: `st7789`, `ili9341`, `ili9486` | `st7789` |

Example:
```bash
digibanner.py -b "KM6LYW" -s "Standby" -t "192.168.1.50" -d st7789
```

The rendered image is also saved to `/run/direwatch.png`.

---

## digibuttons.py — Environment Variables

`digibuttons.py` reads one environment variable from `/home/pi/localize.env` (loaded via `python-dotenv`):

| Variable | Description |
|----------|-------------|
| `NEWDISPLAYTYPE` | Display type string (`st7789`, `ili9341`, or `ili9486`) passed to `digibanner.py` when a service is stopped |

Example `/home/pi/localize.env`:

```
NEWDISPLAYTYPE=st7789
```
