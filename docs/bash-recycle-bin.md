# Bash Recycle Bin with Automated Cleanup

## Overview

A soft-delete system for Bash that moves files to a recycle directory instead of permanently deleting them. Files are automatically purged after 30 days of inactivity. All operations are logged with timestamps.

---

## Components

| Component | Description |
|---|---|
| `bin` | Shell function that moves files to the recycle directory |
| `hoover_rubbish.sh` | Cleanup script that purges files inactive for 30+ days |

---

## Prerequisites

- Bash
- `find`, `stat`, `printf` (standard on all Linux distributions)
- `cron` (optional, for scheduled execution)

---

## Setup

### 1. Create the recycle directory
```bash
mkdir -p ~/30_day_recycle_bin
```

### 2. Add the `bin` function to `~/.bashrc`
```bash
bin() {
  if [ $# -eq 0 ]; then
    echo "Usage: bin <file1> [file2 ...]"
    return 1
  fi
  mv "$@" ~/30_day_recycle_bin/
}
```
```bash
source ~/.bashrc
```

---

## Usage
```bash
bin file1.txt file2.txt
```

Files are moved to `~/30_day_recycle_bin/`. The source directory is unaffected.

---

## Cleanup Script

`hoover_rubbish.sh` identifies and removes files that have not been accessed or modified in 30+ days. Both conditions must be true for deletion to occur.
```bash
#!/bin/bash

RECYCLE_DIR="/home/roy/30_day_recycle_bin"

if [[ ! -d "$RECYCLE_DIR" ]]; then
    echo "Error: Directory $RECYCLE_DIR does not exist."
    exit 1
fi

LOG_FILE="$RECYCLE_DIR/hoover_rubbish.log"
CURRENT_DATE=$(date "+%Y-%m-%d %H:%M")

echo "" >> "$LOG_FILE"
echo "==============================" >> "$LOG_FILE"
echo "$CURRENT_DATE: Script executed." >> "$LOG_FILE"

ALL_FILES=$(find "$RECYCLE_DIR" -type f)

if [[ -z "$ALL_FILES" ]]; then
    echo "$CURRENT_DATE: Checked $RECYCLE_DIR and found no files." >> "$LOG_FILE"
else
    echo "$CURRENT_DATE: Checked $RECYCLE_DIR and found the following files:" >> "$LOG_FILE"
    printf "%-60s %-25s %-25s\n" "File" "atime" "mtime" >> "$LOG_FILE"
    printf "%-60s %-25s %-25s\n" "----" "-----" "-----" >> "$LOG_FILE"

    while IFS= read -r FILE; do
        ATIME=$(stat --format='%x' "$FILE" | cut -d'.' -f1)
        MTIME=$(stat --format='%y' "$FILE" | cut -d'.' -f1)
        printf "%-60s %-25s %-25s\n" "$FILE" "$ATIME" "$MTIME" >> "$LOG_FILE"
    done <<< "$ALL_FILES"

    FILES_TO_DELETE=$(find "$RECYCLE_DIR" -type f -atime +30 -mtime +30)

    if [[ -z "$FILES_TO_DELETE" ]]; then
        echo "$CURRENT_DATE: No files to delete in $RECYCLE_DIR." >> "$LOG_FILE"
    else
        echo "$CURRENT_DATE: Deleting the following files:" >> "$LOG_FILE"
        while IFS= read -r FILE; do
            echo "$FILE" >> "$LOG_FILE"
            rm -f "$FILE"
        done <<< "$FILES_TO_DELETE"
        echo "$CURRENT_DATE: Successfully deleted files older than 1 month from $RECYCLE_DIR." >> "$LOG_FILE"
    fi
fi
```
```bash
chmod +x ~/scripts/hoover_rubbish.sh
```

---

## Scheduling

Run daily at 03:00 via cron:
```bash
crontab -e
```
```
0 3 * * * /home/roy/scripts/hoover_rubbish.sh
```

---

## Log Output

Normal run (no deletions):
```
==============================
2025-01-06 12:00: Script executed.
2025-01-06 12:00: Checked /home/roy/30_day_recycle_bin and found the following files:
File                                                         atime                     mtime
----                                                         -----                     -----
/home/roy/30_day_recycle_bin/teleport_7.3.26_amd64.deb       2024-12-18 14:29:37       2024-11-27 12:10:55
/home/roy/30_day_recycle_bin/hoover_rubbish.log              2025-01-06 09:00:51       2025-01-06 12:00:01
2025-01-06 12:00: No files to delete in /home/roy/30_day_recycle_bin.
```

Deletion run:
```
==============================
2025-01-20 12:00: Script executed.
2025-01-20 12:00: Deleting the following files:
/home/roy/30_day_recycle_bin/teleport_7.3.26_amd64.deb
2025-01-20 12:00: Successfully deleted files older than 1 month from /home/roy/30_day_recycle_bin.
```

---

## Deletion Criteria

A file is deleted only when both of the following are true:

- Not accessed (`atime`) in 30+ days
- Not modified (`mtime`) in 30+ days

---

## Known Limitations

- Systems mounted with `relatime` do not update `atime` on every read. Access time tracking may be imprecise on such systems.
- The `bin` function does not handle filename collisions in the recycle directory. Files with identical names will overwrite each other silently.