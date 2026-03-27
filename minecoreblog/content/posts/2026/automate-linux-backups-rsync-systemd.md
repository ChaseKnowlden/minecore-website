+++
date = '2026-03-27T17:09:07-04:00'
title = 'How to Automate Linux Backups with rsync and systemd (Including Silent Failure Detection)'
+++

Backups are one of those things everyone knows they should have — and yet, "I'll set that up later" has wiped out more data than any hardware failure. In a recent hands-on lab video from Learn Linux TV, the host walks through building a real, production-ready backup system using `rsync`, a custom Bash script, and `systemd` timers. By the end, you have automated incremental backups that run daily and alert you if something quietly breaks.

This post covers the entire process: writing the script, deploying it with `systemd`, and wiring up failure notifications with [healthchecks.io](https://healthchecks.io).

---

## What You'll Need

The setup is intentionally minimal:

- A Linux system (physical, VM, or cloud — distribution doesn't matter)
- `rsync` installed
- A **source directory** — the files you want to back up
- A **destination directory** — where backups will be stored

Install `rsync` with your package manager if it isn't already present:

```bash
# Debian/Ubuntu
sudo apt install rsync

# Fedora/RHEL
sudo dnf install rsync
```

For the destination, a local directory works fine for testing. In production, you'll want backups going somewhere separate from your source — an external drive, NFS share, or remote server. The video uses an NFS share mounted at `/mnt/backup`, but nothing in the script requires that specifically.

If you want NFS support on Debian/Ubuntu:

```bash
sudo apt install nfs-common
```

Then mount your share:

```bash
sudo mount <server-ip>:/path/to/share /mnt/backup
```

---

## Writing the Backup Script

Create the script in your home directory while building it:

```bash
nano ~/backup.sh
```

Here's the complete script, which we'll walk through section by section:

```bash
#!/bin/bash

# Paths
BACKUP_MOUNT="/mnt/backup"
BACKUP_SOURCE="/home/user/my-files"
BACKUP_DEST="${BACKUP_MOUNT}/current"
CURRENT_DATE=$(date +"%Y-%m-%d")
LOG_PATH="${BACKUP_MOUNT}/logs"
PREVIOUS_FILES="${BACKUP_MOUNT}/files-previous/${CURRENT_DATE}/"
HEALTH_CHECK_URL="https://hc-ping.com/your-uuid-here"

# Safety check: abort if the backup mount isn't mounted
if ! mountpoint -q "${BACKUP_MOUNT}"; then
    echo "Backup mount not found. Aborting." >&2
    exit 1
fi

# Create required directories
mkdir -p "${LOG_PATH}"
mkdir -p "${PREVIOUS_FILES}"

# Run the backup
rsync -av \
    --delete \
    --backup \
    --backup-dir="${PREVIOUS_FILES}" \
    "${BACKUP_SOURCE}" \
    "${BACKUP_DEST}" \
    > "${LOG_PATH}/backup-${CURRENT_DATE}.log" \
    2> "${LOG_PATH}/backup-errors-${CURRENT_DATE}.log"

# Ping healthchecks.io on success
if [ $? -eq 0 ]; then
    curl -fsS --retry 3 "${HEALTH_CHECK_URL}" > /dev/null
fi
```

### Variables

Pulling all paths into variables at the top means you only have to change them in one place. The `CURRENT_DATE` variable uses a subshell — `$(date +"%Y-%m-%d")` — to capture the date at runtime, which gets used to name both the previous-files folder and the log file.

### The Safety Check

This block is essential if your destination is a network mount or removable drive:

```bash
if ! mountpoint -q "${BACKUP_MOUNT}"; then
    echo "Backup mount not found. Aborting." >&2
    exit 1
fi
```

Without this, if the NFS share drops or the drive is disconnected, `rsync` will happily write backups *directly to your local filesystem* at `/mnt/backup`. You'll see success messages, but nothing is actually being backed up remotely. The `mountpoint -q` command checks silently whether the path is a mounted filesystem. The `!` inverts it — "if it's *not* mounted, abort."

`exit 1` is intentional: a non-zero exit code signals failure, which becomes important later when we check whether to send a health ping.

### The rsync Command

```bash
rsync -av \
    --delete \
    --backup \
    --backup-dir="${PREVIOUS_FILES}" \
    "${BACKUP_SOURCE}" \
    "${BACKUP_DEST}"
```

| Option | Effect |
|--------|--------|
| `-a` | Archive mode — preserves permissions, ownership, timestamps, symlinks |
| `-v` | Verbose output — logs every file transferred |
| `--delete` | Removes files at the destination that no longer exist at the source |
| `--backup` | Moves overwritten files aside rather than deleting them |
| `--backup-dir` | Where to send those moved files |

The `-a` and `--backup` combination gives you the best of both worlds: the destination always mirrors the source (`--delete`), and nothing is ever truly lost (`--backup`). If you need to restore an older version of a file, it's in the dated `files-previous/` directory.

### Dry Run Mode

Before committing to a real backup, test with `--dry-run`:

```bash
rsync -av --dry-run \
    --delete \
    --backup \
    --backup-dir="${PREVIOUS_FILES}" \
    "${BACKUP_SOURCE}" \
    "${BACKUP_DEST}"
```

This prints everything `rsync` *would* do without actually touching any files. The log output will include `dry run` at the bottom as a reminder. Always verify the file list looks correct before removing this flag.

### Output Redirection

```bash
> "${LOG_PATH}/backup-${CURRENT_DATE}.log" \
2> "${LOG_PATH}/backup-errors-${CURRENT_DATE}.log"
```

Standard output (the list of transferred files) goes to the main log. Standard error (anything that went wrong) goes to a separate error log. Both are date-stamped, so you build up a history automatically.

---

## How Incremental Backups Work

The `current/` directory always holds the latest version of every file. When a file changes and rsync runs again, the *previous* copy is moved to `files-previous/YYYY-MM-DD/` before the new version overwrites it.

```
/mnt/backup/
├── current/
│   └── my-files/
│       ├── report.csv        ← latest version
│       └── script.py         ← latest version
├── files-previous/
│   └── 2026-03-26/
│       └── my-files/
│           └── report.csv    ← yesterday's version
└── logs/
    ├── backup-2026-03-27.log
    └── backup-errors-2026-03-27.log
```

Need to restore a file from three days ago? It's in the corresponding date folder under `files-previous/`.

---

## Deploying the Script

Once you're satisfied with a dry run, move the script to a system-wide location and lock down its ownership:

```bash
sudo mv ~/backup.sh /usr/local/bin/backup.sh
sudo chown root:root /usr/local/bin/backup.sh
sudo chmod 755 /usr/local/bin/backup.sh
```

Keeping it under `/usr/local/bin/` and owned by root prevents accidental modification. A backup script that gets edited by mistake is a backup script you can't trust.

---

## Automating with systemd

Remembering to run the script manually defeats the purpose. `systemd` has two components that work together here: a **service** that runs the script, and a **timer** that triggers the service on a schedule.

### The Service Unit

Create `/etc/systemd/system/backup.service`:

```bash
sudo nano /etc/systemd/system/backup.service
```

```ini
[Unit]
Description=Run a daily backup
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh

[Install]
WantedBy=multi-user.target
```

`After` and `Wants` ensure the service waits for network connectivity before running — critical if you're backing up to a network destination. `Type=oneshot` tells systemd the service runs to completion and exits, rather than staying alive in the background.

### The Timer Unit

Create `/etc/systemd/system/backup.timer`:

```bash
sudo nano /etc/systemd/system/backup.timer
```

```ini
[Unit]
Description=Daily backup timer

[Timer]
OnCalendar=midnight
Persistent=true

[Install]
WantedBy=timers.target
```

`OnCalendar=midnight` runs the backup at midnight each day. Adjust this to any time that suits your schedule — `OnCalendar=*-*-* 02:30:00` for 2:30 AM, for example.

`Persistent=true` is the important one: if the machine was off at midnight, systemd will run the missed job the next time the system boots. Without this, a server that's down for maintenance at midnight simply skips that day's backup with no warning.

> The timer and service files must share the same base name (`backup.timer` triggers `backup.service`). systemd matches them automatically.

### Enable and Start the Timer

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now backup.timer
```

Verify it's active:

```bash
systemctl status backup.timer
```

The output will show the timer as *active (waiting)* and tell you exactly when the next run is scheduled.

To test the full pipeline right now without waiting for midnight, trigger the service directly:

```bash
sudo systemctl start backup.service
systemctl status backup.service
```

A status of `active (exited)` with result `success` means everything worked.

---

## Preventing Silent Failures with healthchecks.io

The `systemd` timer will run the service — but what if the backup silently fails? Maybe the NFS server was unreachable, maybe `rsync` encountered an error, maybe the mount check aborted the script. Without monitoring, you'd only discover the problem when you actually *need* to restore something.

[healthchecks.io](https://healthchecks.io) is a dead man's switch for scheduled jobs. You configure a check with an expected interval, and if it doesn't receive a ping within that window, it sends you an alert. The free tier covers most personal and small-team use cases.

### Setup

1. Create an account at healthchecks.io
2. Create a new project (e.g., *rsync backup*)
3. Add a check — set the period to **1 day** and the grace period to **1 hour**
4. Copy the ping URL (it looks like `https://hc-ping.com/some-uuid`)

### Wiring It Into the Script

The relevant section at the bottom of the script:

```bash
if [ $? -eq 0 ]; then
    curl -fsS --retry 3 "${HEALTH_CHECK_URL}" > /dev/null
fi
```

`$?` holds the exit code of the most recently run command — in this case, `rsync`. If rsync succeeded (exit code `0`), curl pings the health check URL. If rsync failed, or if the mount check earlier caused `exit 1`, the ping never happens. Healthchecks.io notices the silence and sends an alert after the grace period expires.

The curl flags used here:
- `-f` — fail silently on HTTP errors
- `-s` — suppress progress output
- `-S` — still show errors if `-s` is active
- `--retry 3` — retry up to 3 times on transient network issues

Output is sent to `/dev/null` since we only care that the request was made, not what the server returned.

---

## Key Takeaways

- **`rsync -av --delete --backup`** gives you a mirror that never loses old versions — current files in `current/`, replaced files in dated subdirectories
- **Always test with `--dry-run` first** — verify the file list before any real data moves
- **The mount check is not optional in production** — without it, a dropped network share means backups silently write to your local disk
- **`systemd` timers beat cron** for this use case: better logging, dependency ordering, and `Persistent=true` handles missed runs
- **Silent failure is the real enemy** — a monitoring tool like healthchecks.io closes the loop and ensures you actually know when backups stop working

A backup system you never check is a backup system you can't trust. This setup gives you confidence: files are versioned, the timer is persistent, and you'll get an email if anything breaks.

---

*This post is based on the Learn Linux TV video [How to Automate Linux Backups Using rsync and systemd](https://www.youtube.com/@LearnLinuxTV). Sample files for testing the script are available in the video's blog post, linked in the video description.*
