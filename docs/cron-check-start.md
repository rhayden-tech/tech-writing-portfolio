# cron Status Check and Start

## Overview

Two Bash approaches for verifying that the `cron` daemon is running and starting it if not. Includes a comparison of methods and a recommendation.

---

## Method 1: Exit-Code Check (Recommended)

Checks `cron` status via exit code rather than output parsing. Portable across distributions. Verbose output on all outcomes.
```bash
#!/bin/bash

if service cron status >/dev/null 2>&1; then
    echo "Cron is detected to be running ✓"
else
    echo "Cron not running, starting..."
    if sudo service cron start >/dev/null 2>&1; then
        echo "Cron has been started ✓"
    else
        echo "Failed to start Cron"
        exit 1
    fi
fi
echo
```

**Properties:**

- Does not depend on the exact wording of `service cron status` output
- Will not break across distribution updates or locale changes
- Uses interactive `sudo`, no credentials in the script

---

## Method 2: Output Parsing (Not Recommended)
```bash
#!/bin/bash

cronstatus=$(service cron status | cut -c 4-)
if [ "$cronstatus" == "cron is not running" ]; then
    echo password123 | sudo -S service cron start
fi
```

**Problems:**

- Depends on exact output string from `service`. Breaks if wording changes between distributions or updates.
- Hardcodes `sudo` password in plaintext. This is a security risk and defeats the purpose of using `sudo`.
- No output on success or failure. Silent behaviour makes debugging harder.

---

## Comparison

| | Method 1 | Method 2 |
|---|---|---|
| Failure detection | Exit code | String match |
| Credential handling | Interactive sudo | Hardcoded plaintext |
| Portability | Distro-agnostic | Output-dependent |
| User feedback | Verbose | Silent |
| Recommended | Yes | No |

---

## Autostart on Boot

To ensure `cron` is running after every reboot, call Method 1 from a `@reboot` crontab entry:
```
@reboot /home/roy/scripts/check_cron.sh
```

Or enable via systemd:
```bash
sudo systemctl enable --now cron
```

For WSL, use:
```bash
sudo service cron start
```