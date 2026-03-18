# Edge Camera Capture System: Non-Networked Meter Monitoring

## Overview

An autonomous daily image capture system built on a Raspberry Pi Zero 2 WH. Photographs a fixed analogue display (utility meter or similar non-networked device) on a schedule, organises images into monthly folders, retries on failure, sends email alerts on repeated failure, and syncs captured images to a Windows machine via WSL.

---

## System Requirements

### Hardware

| Component | Specification |
|---|---|
| SBC | Raspberry Pi Zero 2 WH |
| Camera | Raspberry Pi Camera Module 3 |
| Ribbon cable | GeeekPi 15-pin to 22-pin |
| Storage | Micro SD card |
| Power | Micro-USB power supply |

### OS

Raspberry Pi OS Lite (32-bit). Flash via Raspberry Pi Imager with the following headless configuration:

- SSH enabled
- Hostname configured (e.g. `meterpi.local`)
- Wi-Fi SSID and password set
- Username, password, locale, and timezone set

---

## Initial Setup
```bash
ssh roy@meterpi.local

sudo apt update && sudo apt full-upgrade -y && sudo reboot
```

Verify camera:
```bash
libcamera-hello
libcamera-still -n -o test.jpg && ls -lh test.jpg
```

---

## Focus Lock

Autofocus variation between captures produces inconsistent framing over time. Lock lens position after finding optimal focus.
```bash
# Find focus
libcamera-still --autofocus-mode auto --autofocus-on-capture -o focus.jpg

# Lock position (tune value as needed)
libcamera-still --lens-position 0.0 -o locked.jpg
```

---

## Version 1: Basic Daily Capture
```bash
mkdir -p /home/roy/meter_photos
```
```bash
#!/bin/bash

DATE=$(date +"%Y-%m-%d_%H-%M")

libcamera-still \
  --lens-position 0.0 \
  --nopreview \
  --width 1920 \
  --height 1080 \
  -o /home/roy/meter_photos/meter_${DATE}.jpg
```
```bash
chmod +x /home/roy/capture_meter.sh
```

Cron schedule (daily at 09:00):
```
0 9 * * * /home/roy/capture_meter.sh
```

---

## Version 2: Resilient Capture with Retry and Email Alerting

Stores images at `/home/roy/meter_photos/<Mon>/<YYYY-MM-DD>.jpg`. Retries once after 1 hour on failure. Sends email alert on second failure.

### Install mail tooling
```bash
sudo apt install -y msmtp msmtp-mta mailutils
```

### Configure msmtp
```
defaults
auth           on
tls            on
tls_starttls   on
logfile        /home/roy/meter_photos/msmtp.log

account        gmail
host           smtp.gmail.com
port           587
from           myusername@gmail.com
user           myusername@gmail.com
password       YOUR_APP_PASSWORD

account default : gmail
```
```bash
chmod 600 /home/roy/.msmtprc
```

### Capture script
```bash
#!/bin/bash
set -euo pipefail

TO_EMAIL="myusername@gmail.com"
BASE="/home/roy/meter_photos"

MONTH="$(LC_TIME=C date +%b)"
DATE="$(date +%F)"
DIR="${BASE}/${MONTH}"
FILE="${DIR}/${DATE}.jpg"
LOG="${BASE}/capture_${DATE}.log"

mkdir -p "$DIR"

capture_once() {
  rpicam-still -n -o "$FILE" >>"$LOG" 2>&1 || return 1
  [[ -s "$FILE" ]]
}

if capture_once; then exit 0; fi

sleep 3600

if capture_once; then exit 0; fi

echo "Meter capture FAILED.
Host: $(hostname)
Time: $(date -Is)
Expected file: $FILE

Last 200 lines of log:
$(tail -n 200 "$LOG" 2>/dev/null || true)
" | mail -s "Meter capture failed (${DATE})" "$TO_EMAIL"

exit 1
```
```bash
chmod +x /home/roy/capture_meter_resilient.sh
```

Cron schedule:
```
0 9 * * * /home/roy/capture_meter_resilient.sh
```

---

## WSL Sync to Windows

Pulls the daily image from the Pi to a Windows path via WSL SCP. Retries once after 1 hour on failure. Sends email alert on second failure.

Target path: `C:\Users\Roy\Desktop\Media Xfer\Images\Meter\<Mon>\<YYYY-MM-DD>.jpg`

### Install mail tooling in WSL
```bash
sudo apt install -y msmtp msmtp-mta mailutils
chmod 600 ~/.msmtprc
```

### Pull script
```bash
#!/bin/bash
set -euo pipefail

TO_EMAIL="myusername@gmail.com"
PI_USER="roy"
PI_IP="<Pi_IP>"
PASSFILE="/home/rhayden/scripts/sshpass_for_meterpi"

MONTH="$(LC_TIME=C date +%b)"
DATE="$(date +%F)"

REMOTE="/home/roy/meter_photos/${MONTH}/${DATE}.jpg"
LOCAL_BASE="/mnt/c/Users/Roy/Desktop/Media Xfer/Images/Meter/${MONTH}"
LOCAL_FILE="${LOCAL_BASE}/${DATE}.jpg"
LOG="/home/rhayden/scripts/pull_meter_${DATE}.log"

mkdir -p "$LOCAL_BASE"
chmod 600 "$PASSFILE"

pull_once() {
  sshpass -f "$PASSFILE" scp \
    -o StrictHostKeyChecking=no \
    -o UserKnownHostsFile=/dev/null \
    "${PI_USER}@${PI_IP}:${REMOTE}" \
    "$LOCAL_FILE" >>"$LOG" 2>&1 || return 1
  [[ -s "$LOCAL_FILE" ]]
}

if pull_once; then exit 0; fi

sleep 3600

if pull_once; then exit 0; fi

echo "Meter WSL pull FAILED.
Time: $(date -Is)
Remote expected: $REMOTE
Local expected:  $LOCAL_FILE

Last 200 lines of log:
$(tail -n 200 "$LOG" 2>/dev/null || true)
" | mail -s "Meter WSL pull failed (${DATE})" "$TO_EMAIL"

exit 1
```
```bash
chmod +x /home/rhayden/scripts/pull_meter_photo.sh
```

Cron schedule (5 minutes after capture):
```
5 9 * * * /home/rhayden/scripts/pull_meter_photo.sh
```

Ensure cron is running in WSL:
```bash
sudo systemctl enable --now cron
# or
sudo service cron start
```

---

## One-Shot Command: Capture, Pull, Open

Triggers capture on the Pi, pulls the image to the local Windows path, and opens it in Windows Explorer.
```bash
#!/bin/bash
set -euo pipefail

PI_USER="roy"
PI_IP="<Pi_IP>"
PASSFILE="/home/rhayden/scripts/sshpass_for_meterpi"

MONTH="$(LC_TIME=C date +%b)"
DATE="$(date +%F)"

REMOTE_CAPTURE="/home/roy/capture_meter_resilient.sh"
REMOTE_FILE="/home/roy/meter_photos/${MONTH}/${DATE}.jpg"
LOCAL_DIR="/mnt/c/Users/Roy/Desktop/Media Xfer/Images/Meter/${MONTH}"
LOCAL_FILE="${LOCAL_DIR}/${DATE}.jpg"

mkdir -p "$LOCAL_DIR"

sshpass -f "$PASSFILE" ssh \
  -o StrictHostKeyChecking=no \
  -o UserKnownHostsFile=/dev/null \
  "${PI_USER}@${PI_IP}" \
  "bash '$REMOTE_CAPTURE'"

sshpass -f "$PASSFILE" scp \
  -o StrictHostKeyChecking=no \
  -o UserKnownHostsFile=/dev/null \
  "${PI_USER}@${PI_IP}:${REMOTE_FILE}" \
  "$LOCAL_FILE"

"/mnt/c/Windows/explorer.exe" "$(wslpath -w "$LOCAL_FILE")"
```
```bash
chmod +x /home/rhayden/scripts/meterphoto.sh
```

Optional alias:
```bash
echo "alias meterphoto='/home/rhayden/scripts/meterphoto.sh'" >> ~/.bashrc
source ~/.bashrc
```

---

## Operational Notes

- Keep the camera mount rigid. Physical movement invalidates frame consistency across captures.
- Keep lighting consistent. Avoid glare and reflections on the display face.
- Do not publish scripts containing real IPs, hostnames, usernames, passwords, or file paths.

---

## Known Limitations

- Requires `sshpass` for non-interactive SCP. Key-based authentication is the more secure alternative.
- No deduplication check. If the one-shot command is run multiple times in a day, the file is overwritten silently.
- No local image validation after pull (e.g. file integrity or corruption check).

---

## Planned Extensions

- OCR of meter reading into a daily CSV
- Daily delta and kWh extraction
- Alerting on abnormal consumption jumps
- Local dashboard for historical trend viewing