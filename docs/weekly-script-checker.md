# Weekly Script Execution Control

## Overview

A Bash wrapper that enforces a minimum 7-day interval between executions of a target script, regardless of how frequently the wrapper itself is called. Intended for use with daily cron jobs where the underlying task only needs to run weekly.

---

## Use Cases

- Scripts scheduled via `@daily` cron that should only fire once per week
- Automation tasks where over-execution causes redundant API calls, unnecessary server load, or log noise

---

## How It Works

1. Reads a timestamp from a file at a defined path
2. Computes the difference between stored and current epoch time
3. Exits silently if fewer than 604800 seconds (7 days) have elapsed
4. Executes the target script and updates the timestamp if the interval has been exceeded

---

## Prerequisites

- Bash 4+
- Target script must be executable (`chmod +x`)

---

## Implementation
```bash
#!/bin/bash

TIMESTAMP_FILE="/tmp/script_last_run"
SCRIPT="/path/to/script.sh"

CURRENT_TIME=$(date +%s)

if [[ -f "$TIMESTAMP_FILE" ]]; then
    LAST_RUN_TIME=$(cat "$TIMESTAMP_FILE")
    TIME_DIFF=$((CURRENT_TIME - LAST_RUN_TIME))

    if (( TIME_DIFF < 604800 )); then
        exit 0
    fi
fi

echo "$CURRENT_TIME" > "$TIMESTAMP_FILE"
bash "$SCRIPT"
```

---

## Configuration

| Variable | Description |
|---|---|
| `TIMESTAMP_FILE` | Path to file storing last execution timestamp |
| `SCRIPT` | Full path to the target script |

---

## Cron Integration

Schedule the wrapper daily. Execution frequency of the target script is self-governed.
```
@daily /home/roy/scripts/runner.sh
```

---

## Known Limitations

- `TIMESTAMP_FILE` defaults to `/tmp`, which does not persist across reboots. Change to a stable path if continuity is required.
- No file locking. Concurrent invocations are not protected against race conditions.
- Interval is fixed at compile time. To make it configurable, replace `604800` with a variable.