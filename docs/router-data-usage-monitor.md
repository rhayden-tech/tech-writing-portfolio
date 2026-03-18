# Router Data Usage Monitor

## Overview

A Python script that authenticates to a Huawei router's local API, retrieves monthly and daily broadband usage statistics, and alerts when consumption crosses a defined threshold. Includes a Bash wrapper that enforces once-per-day execution.

Available in two variants: Windows (desktop notification via `plyer`) and Linux (terminal output only).

---

## Prerequisites

### Router

Huawei router with web interface accessible at `http://192.168.8.1`. Statistics endpoint:
```
http://192.168.8.1/api/monitoring/month_statistics
```

Response format: XML.

### Python dependencies

Windows:
```bash
pip install requests plyer
```

Linux:
```bash
pip install requests
```

---

## Configuration

| Parameter | Description |
|---|---|
| `base_url` | Router local IP |
| `password` | Router web interface password |
| `data_limit` | Monthly cap in bytes (default: 300 GB) |
| `threshold` | Alert threshold as a decimal (e.g. `0.9` = 90%) |

---

## Windows Version

Triggers a desktop notification via `plyer` when usage exceeds the threshold.
```python
import requests
from datetime import timedelta
from plyer import notification

base_url = "http://192.168.8.1"
login_url = f"{base_url}/html/index.html"
password = "Password123"

login_payload = {'Password': password}
headers = {'Content-Type': 'application/x-www-form-urlencoded'}

session = requests.Session()
login_response = session.post(login_url, data=login_payload, headers=headers)

if login_response.status_code == 200:
    print("Login successful")
else:
    print("Login failed!")
    exit()

stats_url = "http://192.168.8.1/api/monitoring/month_statistics"
data_limit = 300 * 1024 * 1024 * 1024
threshold = 0.9

response = session.get(stats_url)

if response.status_code == 200:
    data = response.text

    current_download = int(data[data.find("<CurrentMonthDownload>") + len("<CurrentMonthDownload>"):data.find("</CurrentMonthDownload>")])
    current_upload = int(data[data.find("<CurrentMonthUpload>") + len("<CurrentMonthUpload>"):data.find("</CurrentMonthUpload>")])
    month_duration = int(data[data.find("<MonthDuration>") + len("<MonthDuration>"):data.find("</MonthDuration>")])
    last_clear_time = data[data.find("<MonthLastClearTime>") + len("<MonthLastClearTime>"):data.find("</MonthLastClearTime>")]
    current_day_used = int(data[data.find("<CurrentDayUsed>") + len("<CurrentDayUsed>"):data.find("</CurrentDayUsed>")])
    current_day_duration = int(data[data.find("<CurrentDayDuration>") + len("<CurrentDayDuration>"):data.find("</CurrentDayDuration>")])

    total_usage = current_download + current_upload
    usage_percentage = total_usage / data_limit * 100

    print(f"Total usage: {total_usage / (1024 ** 3):.2f} GB ({usage_percentage:.2f}% of 300 GB limit)")

    if usage_percentage >= threshold * 100:
        warning_message = f"Warning: You have used {usage_percentage:.2f}% of your 300 GB data limit."
        print(warning_message)
        notification.notify(
            title="Data Usage Warning",
            message=warning_message,
            app_name="Data Usage Monitor",
            timeout=10
        )
    else:
        print("Data usage is below the warning threshold.")

    print(f"Data usage for the current day: {current_day_used / (1024 ** 3):.2f} GB")
    print(f"Total online time this month: {str(timedelta(seconds=month_duration))}")
    print(f"Total online time today: {str(timedelta(seconds=current_day_duration))}")
    print(f"Last data reset date: {last_clear_time}")
else:
    print(f"Failed to retrieve stats. Status code: {response.status_code}")
```

---

## Linux Version

Terminal output only. `notify-send` and `plyer` calls are commented out for headless and WSL compatibility. Threshold set to 80%.
```python
import requests
from datetime import timedelta
import subprocess

base_url = "http://192.168.8.1"
login_url = f"{base_url}/html/index.html"
password = "Password123"

login_payload = {'Password': password}
headers = {'Content-Type': 'application/x-www-form-urlencoded'}

session = requests.Session()
login_response = session.post(login_url, data=login_payload, headers=headers)

if login_response.status_code == 200:
    print("Login successful")
else:
    print("Login failed!")
    exit()

stats_url = "http://192.168.8.1/api/monitoring/month_statistics"
data_limit = 300 * 1024 * 1024 * 1024
threshold = 0.8

response = session.get(stats_url)

if response.status_code == 200:
    data = response.text

    current_download = int(data[data.find("<CurrentMonthDownload>") + len("<CurrentMonthDownload>"):data.find("</CurrentMonthDownload>")])
    current_upload = int(data[data.find("<CurrentMonthUpload>") + len("<CurrentMonthUpload>"):data.find("</CurrentMonthUpload>")])
    month_duration = int(data[data.find("<MonthDuration>") + len("<MonthDuration>"):data.find("</MonthDuration>")])
    last_clear_time = data[data.find("<MonthLastClearTime>") + len("<MonthLastClearTime>"):data.find("</MonthLastClearTime>")]
    current_day_used = int(data[data.find("<CurrentDayUsed>") + len("<CurrentDayUsed>"):data.find("</CurrentDayUsed>")])
    current_day_duration = int(data[data.find("<CurrentDayDuration>") + len("<CurrentDayDuration>"):data.find("</CurrentDayDuration>")])

    total_usage = current_download + current_upload
    usage_percentage = total_usage / data_limit * 100

    print(f"Total WiFi data usage: {total_usage / (1024 ** 3):.2f} GB ({usage_percentage:.2f}% of 300 GB monthly limit)")

    if usage_percentage >= threshold * 100:
        warning_message = f"*** Warning: You have used {usage_percentage:.2f}% of your 300 GB monthly data limit ***"
        print(warning_message)
        # subprocess.run(['notify-send', 'Data Usage Warning', warning_message])
    else:
        print("WiFi data usage for the month is below the warning threshold of 80%, all good ✓")

    print(f"Data usage for the current day: {current_day_used / (1024 ** 3):.2f} GB")
    print(f"Total online time this month: {str(timedelta(seconds=month_duration))}")
    print(f"Total online time today: {str(timedelta(seconds=current_day_duration))}")
    print(f"Last data reset date: {last_clear_time}")
else:
    print(f"Failed to retrieve data usage statistics. Status code: {response.status_code}")
```

---

## Once-Per-Day Execution Wrapper

Prevents the script from running more than once per day. Triggered from `~/.bashrc` on shell open.

Add to `~/.bashrc`:
```bash
bash ~/scripts/WIFI.sh
```

`WIFI.sh`:
```bash
#!/bin/bash

LOG_FILE="/tmp/wifi_last_run_time.log"
TODAY=$(date +%F)

if [[ ! -f $LOG_FILE ]]; then
    touch $LOG_FILE
fi

LAST_RUN_DATE=$(cat $LOG_FILE)

if [[ "$LAST_RUN_DATE" == "$TODAY" ]]; then
    echo "WiFi usage for the month was already determined to be fine today ✓"
else
    rm /tmp/pythonoutfile2.log 2>/dev/null

    python3 /home/rhayden/scripts/wifi_data_usage_checker.py > /tmp/pythonoutfile2.log ; echo
    cat /tmp/pythonoutfile2.log | grep 'Warning\|reset\|usage' | grep -v current\ day ; echo
    cal ; echo

    echo "$TODAY" > $LOG_FILE
fi
```

---

## Sample Output
```
Total WiFi data usage: 269.12 GB (89.71% of 300 GB monthly limit)
*** Warning: You have used 89.71% of your 300 GB monthly data limit ***
Data usage for the current day: 2.11 GB
Total online time this month: 210:03:12
Total online time today: 03:42:18
Last data reset date: 2025-07-01
```

---

## Known Limitations

- API endpoint and XML tag names are Huawei-specific. Other router models require modification.
- Credentials stored in plaintext. Do not expose the script in a public repository without redacting.
- XML parsed via string slicing, not a proper XML parser. Malformed responses will cause index errors.