# System Clock Synchronisation: Web Fetch with Manual Fallback

## Overview

A Bash script that sets the system clock by scraping current time from a public web source, validates the result, and falls back to manual input if the fetch fails or the system is offline. Updates the hardware clock on completion.

Intended for air-gapped environments, freshly provisioned VMs, and embedded systems where NTP is unavailable or has not yet synchronised.

---

## Prerequisites
```bash
sudo apt install curl html2text
```

---

## Method 1: Auto-Sync from Web Source

Fetches current time from `timeanddate.com`, validates format, sets system and hardware clocks. Falls back to manual input on validation failure.
```bash
#!/bin/bash

current_time=$(curl -sk https://www.timeanddate.com/worldclock/ireland/dublin | html2text | grep GMT | head -n1 | awk '{print $1}')
current_date=$(curl -sk https://www.timeanddate.com/worldclock/ireland/dublin | html2text | grep -A1 GMT | sed -n '2p')

if [[ "$current_time" =~ ^[0-9]{2}:[0-9]{2}:[0-9]{2}$ && "$current_date" =~ ^[A-Za-z]+,\ [0-9]+\ [A-Za-z]+\ [0-9]{4}$ ]]; then
    echo "Fetched date and time successfully:"
    echo "Date: $current_date"
    echo "Time: $current_time"

    formatted_date=$(date -d "$current_date" +%Y-%m-%d)
    datetime="${formatted_date} ${current_time}"
    echo "Setting system date and time to: $datetime"
    sudo date -s "$datetime"
else
    echo "Failed to fetch date and/or time automatically. Switching to manual mode..."

    read -p "Enter the current date (YYYY-MM-DD): " input_date
    read -p "Enter the current time hour (HH, 00-23): " input_hour
    read -p "Enter the current time minutes (MM, 00-59): " input_minute
    datetime="${input_date} ${input_hour}:${input_minute}:00"
    echo "Setting system date and time to: $datetime"
    sudo date -s "$datetime"
fi

sudo hwclock --systohc
echo "Date and time have been updated successfully."
```

---

## Method 2: Manual Input Only

For offline systems or environments where web access is unavailable.
```bash
#!/bin/bash

read -p "Enter the current date (YYYY-MM-DD): " input_date
read -p "Enter the current time hour (HH, 00-23): " input_hour
read -p "Enter the current time minutes (MM, 00-59): " input_minute

datetime="${input_date} ${input_hour}:${input_minute}:00"
echo "Setting system date and time to: $datetime"
sudo date -s "$datetime"
sudo hwclock --systohc
echo "Date and time have been updated successfully."
```

---

## NTP Ongoing Sync (Optional)

Once the clock is within an acceptable range, use NTP to prevent drift. Does not require the full `ntpd` daemon.
```bash
sudo apt install ntpdate
sudo ntpdate ntp.ubuntu.com
```

Cron job to sync hourly:
```bash
crontab -e
```
```
0 * * * * /usr/sbin/ntpdate ntp.ubuntu.com
```

---

## Known Limitations

- Web scraping method depends on the structure of `timeanddate.com`. Page layout changes may break parsing.
- `sudo date -s` requires root privileges or appropriate sudoers configuration.
- NTP hourly sync using `ntpdate` is a lightweight alternative to `ntpd` but provides less precision and no continuous adjustment.