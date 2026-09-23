# Phase 1 / Module 1.1 — Challenge 4: Custom Systemd Service and Timer

## Context & Objectives
In this challenge, I replaced a legacy cron heartbeat pattern with a native system-level systemd `oneshot` service managed by a systemd timer. The objective was to create a reliable periodic task that records an ISO-8601 timestamp into `/var/log/devops-heartbeat.log`, verify scheduling, inspect the units, analyze boot impact, and document behavior and troubleshooting steps.

---

## Architectural Choices & Design

1. **`Type=oneshot`**:
   - The script is not a long-running daemon; it performs a single task (appending a date string) and exits immediately. `Type=oneshot` informs systemd that the process finishes immediately, avoiding hanging states or unnecessary restart loops.

2. **`OnCalendar=*:0/1` and Timer Coalescing (`AccuracySec=`)**:
   - The timer targets the boundary of every minute (`*:0/1`).
   - By default, systemd timers apply an `AccuracySec=` window of 1 minute (60s). This allows the Linux kernel to coalesce timer events to reduce CPU wakeups and conserve power. Consequently, actual unit execution triggers within that accuracy window (e.g., at :08, :11, :12) rather than strictly pinning to `XX:XX:00.000`.

3. **`Persistent=true`**:
   - Unlike standard cron jobs that silently drop triggers if the machine is powered off or suspended, `Persistent=true` writes the last execution timestamp to disk (`/var/lib/systemd/timers/`). If a scheduled tick is missed during VM downtime, systemd fires the service immediately upon reboot.

4. **Decoupled Local Dependencies**:
   - The service writes locally to `/var/log/devops-heartbeat.log` and has no network requirements. The unit avoids `After=network.target` to prevent artificial ordering constraints during boot or runtime.

---

## Implementation Details

### 1. Heartbeat Script (`/usr/local/bin/devops-heartbeat.sh`)
Defensive bash flags (`-euo pipefail`) were applied, appending standard ISO-8601 timestamps using `date -Iseconds`:

```bash
#!/usr/bin/env bash
set -euo pipefail

date -Iseconds >> /var/log/devops-heartbeat.log
```

Permissions applied:
```bash
sudo chmod 755 /usr/local/bin/devops-heartbeat.sh
```

### 2. Systemd Service Unit (`/etc/systemd/system/devops-heartbeat.service`)
```ini
[Unit]
Description=DevOps Heartbeat Logger

[Service]
Type=oneshot
ExecStart=/usr/local/bin/devops-heartbeat.sh

[Install]
WantedBy=multi-user.target
```

### 3. Systemd Timer Unit (`/etc/systemd/system/devops-heartbeat.timer`)
```ini
[Unit]
Description=Run DevOps Heartbeat periodically

[Timer]
OnCalendar=*:0/1
Persistent=true
Unit=devops-heartbeat.service

[Install]
WantedBy=timers.target
```

### 4. Activation
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now devops-heartbeat.timer
```

---

## Acceptance Evidence

### 1. Log File Content & File Metadata
Verified that the log file is actively populated with valid ISO-8601 timestamps and inspected file metadata (`stat`):

```text
[josemanco@arch-lab ~]$ sudo tail -n 5 /var/log/devops-heartbeat.log
2026-09-23T16:34:12+02:00
2026-09-23T16:35:12+02:00
2026-09-23T16:36:12+02:00
2026-09-23T16:37:08+02:00
2026-09-23T16:38:11+02:00

[josemanco@arch-lab ~]$ sudo stat /var/log/devops-heartbeat.log
  File: /var/log/devops-heartbeat.log
  Size: 806             Blocks: 8          IO Block: 4096   regular file
Device: 8,2     Inode: 1049836     Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2026-09-23 16:38:11.548247530 +0200
Modify: 2026-09-23 16:38:11.510247532 +0200
Change: 2026-09-23 16:38:11.510247532 +0200
 Birth: 2026-09-23 15:43:06.748443781 +0200
```

### 2. Timer Status and Next Run Schedule
Verified that `devops-heartbeat.timer` is enabled, active (waiting), and counting down:

```text
[josemanco@arch-lab ~]$ systemctl status devops-heartbeat.timer
● devops-heartbeat.timer - Run DevOps Heartbeat periodically
     Loaded: loaded (/etc/systemd/system/devops-heartbeat.timer; enabled; preset: disabled)
     Active: active (waiting) since Wed 2026-09-23 16:10:41 CEST; 5min ago
 Invocation: a8aa83537d754b4ca5512dc4d4d1fc16
    Trigger: Wed 2026-09-23 16:16:00 CEST; 3s left
   Triggers: ● devops-heartbeat.service

Sep 23 16:10:41 arch-lab systemd[1]: Started Run DevOps Heartbeat periodically.

[josemanco@arch-lab ~]$ systemctl list-timers devops-heartbeat.timer
NEXT                         LEFT LAST                          PASSED UNIT                   ACTIVATES
Wed 2026-09-23 16:16:00 CEST   3s Wed 2026-09-23 16:15:12 CEST 44s ago devops-heartbeat.timer devops-heartbeat.service

1 timers listed.
Pass --all to see loaded but inactive timers, too.
```

### 3. Unit Inspection (`systemctl cat`)
```text
[josemanco@arch-lab ~]$ systemctl cat devops-heartbeat.service devops-heartbeat.timer
# /etc/systemd/system/devops-heartbeat.service
[Unit]
Description=DevOps Heartbeat Logger

[Service]
Type=oneshot
ExecStart=/usr/local/bin/devops-heartbeat.sh

[Install]
WantedBy=multi-user.target

# /etc/systemd/system/devops-heartbeat.timer
[Unit]
Description=Run DevOps Heartbeat periodically

[Timer]
OnCalendar=*:0/1
Persistent=true
Unit=devops-heartbeat.service

[Install]
WantedBy=timers.target
```

### 4. Journal Logs & Boot Overhead
```text
[josemanco@arch-lab ~]$ journalctl -u devops-heartbeat.service -n 20 --no-pager
Sep 23 16:09:34 arch-lab systemd[1]: devops-heartbeat.service: Deactivated successfully.
Sep 23 16:09:34 arch-lab systemd[1]: Finished DevOps Heartbeat Logger.
Sep 23 16:11:12 arch-lab systemd[1]: Starting DevOps Heartbeat Logger...
Sep 23 16:11:12 arch-lab systemd[1]: devops-heartbeat.service: Deactivated successfully.
Sep 23 16:11:12 arch-lab systemd[1]: Finished DevOps Heartbeat Logger.
Sep 23 16:12:12 arch-lab systemd[1]: Starting DevOps Heartbeat Logger...
Sep 23 16:12:12 arch-lab systemd[1]: devops-heartbeat.service: Deactivated successfully.
Sep 23 16:12:12 arch-lab systemd[1]: Finished DevOps Heartbeat Logger.
Sep 23 16:13:12 arch-lab systemd[1]: Starting DevOps Heartbeat Logger...
Sep 23 16:13:12 arch-lab systemd[1]: devops-heartbeat.service: Deactivated successfully.
Sep 23 16:13:12 arch-lab systemd[1]: Finished DevOps Heartbeat Logger.
Sep 23 16:14:06 arch-lab systemd[1]: Starting DevOps Heartbeat Logger...
Sep 23 16:14:06 arch-lab systemd[1]: devops-heartbeat.service: Deactivated successfully.
Sep 23 16:14:06 arch-lab systemd[1]: Finished DevOps Heartbeat Logger.
Sep 23 16:15:12 arch-lab systemd[1]: Starting DevOps Heartbeat Logger...
Sep 23 16:15:12 arch-lab systemd[1]: devops-heartbeat.service: Deactivated successfully.
Sep 23 16:15:12 arch-lab systemd[1]: Finished DevOps Heartbeat Logger.
Sep 23 16:16:06 arch-lab systemd[1]: Starting DevOps Heartbeat Logger...
Sep 23 16:16:06 arch-lab systemd[1]: devops-heartbeat.service: Deactivated successfully.
Sep 23 16:16:07 arch-lab systemd[1]: Finished DevOps Heartbeat Logger.

[josemanco@arch-lab ~]$ systemd-analyze blame | grep devops-heartbeat || echo "Executed sub-millisecond (not listed in blame)"
   99ms devops-heartbeat.service
```

---

## Failures, Debugging & Fixes

1. **Permission Denied on Direct Shell Execution**:
   - *Symptom*: Running `./devops-heartbeat.sh` as regular user `josemanco` failed with `line 4: /var/log/devops-heartbeat.log: Permission denied`.
   - *Root Cause*: `/var/log` is owned by `root:root` (`755`), preventing unprivileged writes.
   - *Resolution*: Direct interactive tests must run under `sudo`, while the system-level systemd service handles writes automatically since unit processes execute as root by default.

2. **Removal of Redundant Network Dependency**:
   - *Issue*: `devops-heartbeat.service` initially had `After=network.target`.
   - *Fix*: Writing a local timestamp to disk does not depend on the network stack. The directive was removed to avoid creating arbitrary ordering dependencies during boot.

3. **Install Target Typo**:
   - *Issue*: The original service draft had `WantedBy=multy-user.target`.
   - *Fix*: Corrected to `multi-user.target`.

---

## Conclusion & Key Takeaways
- Systemd timers provide cleaner observability than cron; `systemctl list-timers` exposes exact scheduling windows without syslog scraping.
- Understanding `AccuracySec=` is critical when evaluating execution timestamps in real-world environments.
- Decoupling schedule logic (`.timer`) from task logic (`.service`) enables isolated testing via standard `systemctl` verbs.
