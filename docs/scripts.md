# Script Reference

This page documents each Python script in the repository, its purpose, dependencies, and internal behaviour.

---

## direwatch.py

**Main script.** Tails a Direwolf log file and renders received APRS station information on a TFT display.

### Dependencies

| Module | Source |
|--------|--------|
| `argparse` | stdlib |
| `time`, `subprocess`, `threading`, `signal`, `os`, `re`, `math` | stdlib |
| `PIL` (Pillow) | `python3-pil` |
| `adafruit_rgb_display.st7789` | `adafruit-circuitpython-rgb-display` |
| `adafruit_rgb_display.ili9341` | `adafruit-circuitpython-rgb-display` |
| `ILI9486` | bundled (`ILI9486.py`) |
| `pyinotify` | `python3-pyinotify` |
| `gpiod` | `python3-libgpiod` |
| `aprslib` | `aprslib` |
| `numpy` | `python3-numpy` |

### Startup sequence

1. Parse command-line arguments.
2. Initialise the selected TFT display driver.
3. Set up GPIO outputs: blue LED (GPIO 5) and red LED (GPIO 26).
4. Load DejaVu fonts from `/usr/share/fonts/truetype/dejavu/` (or current directory as fallback).
5. Load APRS symbol sprite sheets (`aprs-symbols-128-0.png`, `aprs-symbols-128-1.png`).
6. Draw a splash screen showing `title_text` for ~1 second.
7. Draw the initial header bar with title, Bluetooth icon, and dim PTT/DCD circles.
8. Start background threads:
   - **`btwatch`** — polls `hcitool con` every 2 seconds for Bluetooth connections.
   - **`rgwatch`** — tails the log file for `DCD`/`PTT` lines to update status indicators.
9. Enter the main display loop (`single_loop` or `list_loop`).

### Display threads

#### `bluetooth_connection_poll_thread` (`btwatch`)

Runs every 2 seconds. Executes `hcitool con | wc -l` to count active Bluetooth connections. Updates the Bluetooth icon in the title bar and toggles the blue LED (GPIO 5) accordingly.

#### `redgreen_thread` (`rgwatch`)

Tails the log file (`tail -F`) and watches for lines matching `DCD 0 = 1|0` or `PTT 0 = 1|0`. Updates the red/green circles in the title bar and the red LED (GPIO 26). Requires Direwolf to be run with `-d o`.

### Main loops

#### `single_loop` (enabled by `-o`)

Reads one APRS packet at a time from the log. For each packet:
1. Parses it with `aprslib.parse()`. Falls back to a regex callsign extraction for unsupported packet types.
2. Extracts the APRS symbol character and table identifier.
3. Crops the matching symbol from the 128-pixel-per-symbol sprite sheet.
4. Resizes the symbol to fit the display (up to 180×180 px on larger screens).
5. Draws:
   - Symbol (upper-left, below title bar)
   - Distance + bearing (if coordinates provided)
   - Temperature in °F (weather packets)
   - Comment/status text
   - Callsign (large, pinned to the bottom of the screen)
6. Pushes the image to the display (and saves PNG if `-s` was given).
7. Sleeps 1 second before the next packet.

#### `list_loop` (default)

Reads packets from the log and maintains a scrolling grid:
- Callsigns + symbols fill left-to-right, top-to-bottom.
- Duplicate callsign: the existing entry blinks (blanked for 0.5 s, then redrawn).
- When a column fills, the next column starts.
- When all columns fill, the screen clears after a 2-second pause and restarts.

### APRS symbol rendering

APRS symbols are stored in two 16×6 sprite sheets (`aprs-symbols-128-0.png` and `aprs-symbols-128-1.png`), each cell being 128×128 pixels.

Symbol selection:
- `symbol_table == '/'` → primary table (`aprs-symbols-128-0.png`)
- `symbol_table != '/'` → alternate/overlay table (`aprs-symbols-128-1.png`)

The ASCII code of the symbol character minus 33 gives the index into the 16-column sprite sheet:

```python
offset = ord(symbol) - 33
row = offset // 16
col = offset % 16
```

### Distance and bearing calculation

`get_distance(origin, destination)` uses the Haversine formula to compute great-circle distance in **miles** (radius = 3959 miles).

`get_direction(origin, destination)` computes the initial bearing using the forward azimuth formula, then maps it to one of 16 compass points (`N`, `NNE`, `NE`, … `NNW`).

### Signal handling

`SIGINT` and `SIGTERM` are caught to:
1. Draw a dark grey screen (graceful exit indicator).
2. Release GPIO lines.
3. Call `os._exit(0)`.

---

## digibanner.py

**Splash screen utility.** Renders a customisable text banner and optional graphic on a TFT display, then exits. Useful as a startup screen or status message.

### Dependencies

| Module | Source |
|--------|--------|
| `argparse`, `os`, `subprocess` | stdlib |
| `PIL` (Pillow) | `python3-pil` |
| `adafruit_rgb_display.st7789` | `adafruit-circuitpython-rgb-display` |
| `adafruit_rgb_display.ili9341` | `adafruit-circuitpython-rgb-display` |
| `ILI9486` | bundled (`ILI9486.py`) |

### Layout

```
┌──────────────────────────────────┐
│ DigiPi                           │  ← always "DigiPi" in small grey text
│                                  │
│                                  │
│  BIG TEXT          [graphic.png] │  ← -b, large font (34pt + bump)
│  Small text                      │  ← -s, medium condensed font (22pt + bump)
│                                  │
│  tiny text                       │  ← -t, small font (18pt + bump), pinned bottom
└──────────────────────────────────┘
```

`fontbump` adds extra size for larger displays: 0 for ST7789, 4 for ILI9341, 10 for ILI9486.

The rendered image is always saved to `/run/direwatch.png` before the script exits.

### Usage

```bash
digibanner.py -b "KM6LYW" -s "Standby" -t "192.168.1.50" -d st7789
digibanner.py -b "DigiPi" -s "Running" -g /home/pi/logo.png -d ili9341
```

---

## digibuttons.py

**GPIO button monitor.** Watches two push-buttons (GPIO 23 and GPIO 24) and toggles systemd services in response to button presses. Designed for use with DigiPi as part of a standalone digipeater appliance.

### Dependencies

| Module | Source |
|--------|--------|
| `gpiod` | `python3-libgpiod` |
| `select`, `subprocess`, `signal`, `os`, `threading` | stdlib |
| `dotenv` | `python-dotenv` |
| `pathlib` | stdlib |

### Button behaviour

| GPIO | Button | Rising-edge action |
|------|--------|--------------------|
| GPIO 23 | TNC button | Toggle `tnc` systemd service |
| GPIO 24 | Digipeater button | Toggle `digipeater` systemd service |

**Toggle logic (per button):**
- If the service is **inactive**: start it (`systemctl start <service>`).
- If the service is **active**: stop it (`systemctl stop <service>`), retrieve the Pi's IP address, and call `digibanner.py -b Standby -s <IP>` to show a status screen.

### Environment

Reads `/home/pi/localize.env` using `python-dotenv`. Expected variable:

```
NEWDISPLAYTYPE=st7789
```

This is passed as `-d <NEWDISPLAYTYPE>` to `digibanner.py`.

### Threading model

Two background threads run `select.poll()` loops, one per GPIO line:
- `bg_thread_23` → watches GPIO 23, calls `handle_gpio23()` on rising edge
- `bg_thread_24` → watches GPIO 24, calls `handle_gpio24()` on rising edge

A shared `done_fd` (Linux eventfd) signals both threads to exit cleanly on `SIGINT`/`SIGTERM`.

### Note on signal handling

The script registers `SIGINT` and `SIGTERM` handlers that write to `done_fd` and call `os._exit(0)`. However, `os` is referenced in the handler before the `import os` statement in `__main__`. The `os` import at module level inside the handler body means the handler only works correctly when run as the main module.

---

## ILI9486.py

**ILI9486 TFT display driver.** A pure-Python SPI driver for the ILI9486 480×320 display controller, adapted from [SirLefti/Python_ILI9486](https://github.com/SirLefti/Python_ILI9486).

### Dependencies

| Module | Source |
|--------|--------|
| `spidev` | `python3-spidev` |
| `gpiod` | `python3-libgpiod` |
| `PIL` (Pillow) | `python3-pil` |
| `numpy` | `python3-numpy` |

### Classes

#### `Origin` (Enum)

Defines the display orientation by setting the MADCTL register byte. Each value encodes the MY, MX, MV, ML, BGR, MH bits.

| Constant | MADCTL byte | Description |
|----------|------------|-------------|
| `UPPER_LEFT` | `0x28` | Portrait, Pi GPIO at top |
| `LOWER_RIGHT` | `0xE8` | Landscape, used by direwatch |
| `UPPER_LEFT_MIRRORED` | `0xA8` | — |
| `LOWER_LEFT` | `0x48` | — |
| `LOWER_LEFT_MIRRORED` | `0x08` | — |
| `UPPER_RIGHT` | `0x88` | — |
| `UPPER_RIGHT_MIRRORED` | `0xC8` | — |
| `LOWER_RIGHT_MIRRORED` | `0x68` | — |

direwatch and digibanner use `Origin.LOWER_RIGHT`.

#### `ILI9486`

Main display class.

| Method | Description |
|--------|-------------|
| `__init__(spi, dc, rst=None, *, origin)` | Create driver instance. `spi` is a `SpiDev` object; `dc` and `rst` are BCM GPIO pin numbers. |
| `begin()` | Reset display and run init sequence. Must be called before use. Returns `self`. |
| `reset()` | Toggle RST pin to hardware-reset the display. No-op if `rst` is `None`. Returns `self`. |
| `image(image, x0, y0)` | Blit a Pillow `Image` to the display at position `(x0, y0)`. Image must be RGB or RGBA, same size as the display. |
| `clear(color)` | Fill the internal buffer with a solid RGB colour. Does not push to display. |
| `draw()` | Return a `PIL.ImageDraw.Draw` object for drawing on the internal buffer. |
| `invert(state=True)` | Enable or disable display colour inversion. |
| `idle(state=True)` | Enable or disable idle mode (reduced colour depth). |
| `on()` / `off()` | Turn the display on or off. |
| `sleep()` / `wake_up()` | Enter or exit sleep mode. |
| `set_window(x0, y0, x1, y1)` | Set the pixel address window for subsequent write commands. |
| `send(data, is_data, chunk_size)` | Low-level SPI write; sets DC pin accordingly. |
| `command(data)` / `data(data)` | Send a command byte or data byte(s). |
| `dimensions()` | Return `(width, height)` tuple for current orientation. |
| `is_landscape()` | Return `True` if MV bit is set (landscape mode). |
| `is_inverted()` | Return current inversion state. |
| `is_idle()` | Return current idle state. |

### Pixel format

The ILI9486 is configured for **18-bit (666 RGB)** colour (`CMD_PXLFMT = 0x3A, data = 0x66`). The `image_to_data()` helper converts a Pillow image to this format by masking the two LSBs of each 8-bit channel:

```python
pb[:, :, channel] & 0xFC   # keep top 6 bits
```

### SPI configuration

```python
spi = SpiDev(0, 0)
spi.mode = 0b10          # CPOL=1, CPHA=0
spi.max_speed_hz = 48000000
```

### Initialisation sequence

The `_init_sequence()` method sends:
1. Interface mode control (`CMD_IFMODE`)
2. Sleep out (`CMD_SLPOUT`) — 20 ms delay
3. Pixel format: 18 bpp (`CMD_PXLFMT`, `CMD_RDPXLFMT`)
4. Power control (`CMD_PWRCTLNOR`)
5. VCOM control (`CMD_VCOMCTL`)
6. Positive gamma correction (`CMD_PGAMCTL`) — 15 bytes, sent one at a time
7. Negative gamma correction (`CMD_NGAMCTL`) — 15 bytes, sent one at a time
8. Memory address control / orientation (`CMD_MADCTL`)
9. Sleep out + display on

---

## Asset Files

| File | Description |
|------|-------------|
| `aprs-symbols-128-0.png` | Primary APRS symbol sprite sheet (128 px/symbol, 16×6 grid) |
| `aprs-symbols-128-1.png` | Alternate/overlay APRS symbol sprite sheet |
| `bt.small.on.png` | Bluetooth connected icon (transparent PNG, ~28×28 px) |
| `bt.small.off.png` | Bluetooth disconnected icon (transparent PNG, ~28×28 px) |
