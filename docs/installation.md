# Installation Guide

This guide covers installing direwatch on a Raspberry Pi running **Raspberry Pi OS Bookworm** (or later).

---

## Prerequisites

- Raspberry Pi (any model with SPI; Zero 2W recommended for size)
- Supported TFT display (see [Hardware & Wiring](hardware.md))
- Raspberry Pi OS Bookworm (Debian 12) or later
- Internet connection for package installation

---

## 1. Update the system

```bash
sudo apt-get update && sudo apt-get upgrade -y
```

---

## 2. Install system packages

```bash
sudo apt-get install -y \
    direwolf \
    rtl-sdr \
    git \
    python3-pip \
    fonts-dejavu \
    python3-pil \
    python3-pyinotify \
    python3-numpy \
    python3-libgpiod \
    python3-lgpio
```

| Package | Purpose |
|---------|---------|
| `direwolf` | APRS/TNC software that produces the log file direwatch reads |
| `rtl-sdr` | Utilities for RTL-SDR USB dongles (optional, for SDR receive) |
| `fonts-dejavu` | DejaVu fonts used by direwatch for display rendering |
| `python3-pil` | Pillow image library for screen drawing |
| `python3-pyinotify` | File system event monitoring |
| `python3-numpy` | Numerical operations (used by ILI9486 driver) |
| `python3-libgpiod` | GPIO access via libgpiod (replaces RPi.GPIO) |
| `python3-lgpio` | Alternative GPIO library (used by Adafruit Blinka) |

---

## 3. Install Python packages

```bash
sudo pip3 install --break-system-packages \
    adafruit-circuitpython-rgb-display \
    Adafruit-Blinka \
    aprslib
```

| Package | Purpose |
|---------|---------|
| `adafruit-circuitpython-rgb-display` | Adafruit drivers for ST7789 and ILI9341 displays |
| `Adafruit-Blinka` | CircuitPython compatibility layer for Raspberry Pi |
| `aprslib` | APRS packet parser used to extract callsigns and positions |

> **Note:** The `--break-system-packages` flag is required on Bookworm because the system Python environment is managed by the OS. These packages do not conflict with system packages.

---

## 4. Clone the repository

```bash
git clone https://github.com/craigerl/direwatch.git
cd direwatch
```

---

## 5. Enable the SPI interface

The TFT displays communicate over SPI, which must be enabled on the Raspberry Pi.

**Option A — edit config.txt manually:**

```bash
sudo nano /boot/firmware/config.txt
```

Find and uncomment (or add) the following line:

```
dtparam=spi=on
```

**Option B — use raspi-config:**

```bash
sudo raspi-config
```

Navigate to **Interface Options → SPI → Enable**.

**Reboot after enabling SPI:**

```bash
sudo reboot
```

---

## 6. Verify SPI is active

After rebooting, check that the SPI device nodes exist:

```bash
ls /dev/spidev*
# Expected output: /dev/spidev0.0  /dev/spidev0.1
```

---

## 7. Configure Direwolf

Create or edit your Direwolf configuration file. The following is a minimal example for an RTL-SDR receive-only igate:

```
MYCALL NOCALL
IGSERVER noam.aprs2.net
IGLOGIN NOCALL 12345
PBEACON sendto=IG compress=1 delay=00:15 every=30:00 symbol="igate" overlay=X lat=40.911 long=-122.935 comment="Direwatch Rx-only igate"
AGWPORT 8000
KISSPORT 8001
ADEVICE null
```

Replace `NOCALL` with your callsign, update the `IGLOGIN` password, and set your `lat`/`long` in the `PBEACON` line.

> **Important:** Add `-d o` to the Direwolf command line to enable PTT/DCD log output, which drives the red/green status indicators in direwatch.

---

## 8. Run Direwolf and direwatch

```bash
# Pipe RTL-SDR audio into Direwolf, logging to a file
rtl_fm -s 22050 -g 49 -f 144.39M 2>/dev/null | direwolf -d o -t 0 -r 22050 - > direwolf.log &

# Start direwatch in single-station mode
./direwatch.py -o -l direwolf.log -t "APRS"
```

See the [Configuration Reference](configuration.md) for all available options.

---

## Running as a systemd service

To start direwatch automatically at boot, create a service unit:

```bash
sudo nano /etc/systemd/system/direwatch.service
```

```ini
[Unit]
Description=Direwatch APRS TFT display
After=network.target

[Service]
Type=simple
User=pi
WorkingDirectory=/home/pi/direwatch
ExecStart=/home/pi/direwatch/direwatch.py -o -l /run/direwolf.log -t "APRS"
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable direwatch
sudo systemctl start direwatch
```

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `No module named 'board'` | Adafruit-Blinka not installed | `sudo pip3 install --break-system-packages Adafruit-Blinka` |
| `No module named 'adafruit_rgb_display'` | Missing Adafruit display driver | `sudo pip3 install --break-system-packages adafruit-circuitpython-rgb-display` |
| Blank screen | SPI not enabled | Enable SPI and reboot (step 5) |
| `Couldn't find font DejaVuSans.ttf` | Missing font package | `sudo apt-get install fonts-dejavu` |
| No callsigns appear | Direwolf not logging / wrong log path | Check `-l` path; verify Direwolf is running and writing to the file |
| Red/green indicators always off | Direwolf missing `-d o` flag | Restart Direwolf with `-d o` |
| GPIO permission error | User not in `gpio` group | `sudo usermod -aG gpio $USER` then log out and back in |
