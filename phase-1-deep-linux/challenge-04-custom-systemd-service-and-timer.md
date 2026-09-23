# Phase 1 / Module 1.1 — Challenge 4: Custom Systemd Service and Timer

## Context & Objectives
In this challenge, I replaced a legacy cron heartbeat pattern with a native system-level systemd `oneshot` service managed by a systemd timer. The objective was to create a reliable periodic task that records an ISO-8601 timestamp into `/var/log/devops-heartbeat.log`, verify scheduling, inspect the units, analyze boot impact, and document the behavior and troubleshooting steps.

---

## Architectural Choices & Design

When building out this solution, I made several deliberate design decisions:

1. **`Type=oneshot` vs `Type=simple`**:
   - The script is not a persistent daemon; it performs a single task (appending one line of date output) and exits immediately. Setting `Type=oneshot` informs systemd that the process is expected to finish before considering the execution complete, avoiding hanging states or unnecessary restart loops.

2. **`OnCalendar=*:0/1`**:
   - I used calendar scheduling rather than monotonic elapsed time (`OnUnitActiveSec=`). `OnCalendar=*:0/1` synchronizes the heartbeat directly to wall-clock minute boundaries (`:00`, `:01`, `:02`), providing clean, predictable timestamp intervals in the log.

3. **`Persistent=true`**:
   - Standard cron jobs silently drop triggers if the machine is powered down or suspended. Setting `Persistent=true` tells systemd to write the last trigger time to disk (`/var/lib/systemd/timers/`); if a scheduled run is missed during VM downtime, systemd fires the service immediately upon boot.

4. **Script Location & Security**:
   - The script lives in `/usr/local/bin/devops-heartbeat.sh` with permissions `755`. This adheres to FHS guidelines for local system executables, ensuring standard users can inspect or run it if permitted, while preserving root-level execution when invoked by systemd.

---

## Step-by-Step Implementation

### 1. Heartbeat Script (`/usr/local/bin/devops-heartbeat.sh`)
I created the script with standard defensive bash flags (`-euo pipefail`) and formatted the output using `date -Iseconds`:

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
After=network.target

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

### 4. Daemon Reload and Activation
After creating the unit definitions, I reloaded the manager configuration and enabled the timer immediately:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now devops-heartbeat.timer
```

---

## Acceptance Evidence

### 1. Timer Active Status & Scheduled Run Verification
I verified that `devops-heartbeat.timer` was enabled, active (waiting), and correctly armed for the next trigger:

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

### 2. Unit File Inspection (`systemctl cat`)
Inspecting both unit configurations registered with systemd:

```text
[josemanco@arch-lab ~]$ systemctl cat devops-heartbeat.service devops-heartbeat.timer
# /etc/systemd/system/devops-heartbeat.service
[Unit]
Description=DevOps Heartbeat Logger
After=network.target

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

### 3. Journalctl Service Execution Logs
Reviewing the execution journal confirmed that the service repeatedly triggers every minute, transitions through `Starting` -> `Deactivated successfully` -> `Finished`, and cleans up as a proper `oneshot`:

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
```

### 4. Boot Overhead Analysis (`systemd-analyze blame`)
Analyzing the startup impact showed that the service introduces minimal boot footprint (around 99ms):

```text
[josemanco@arch-lab ~]$ systemd-analyze blame | grep devops-heartbeat || echo "Executed sub-millisecond (not listed in blame)"
   99ms devops-heartbeat.service
```

---

## Failures, Debugging & Fixes

During the initial build and test phase, I encountered two specific issues:

1. **Permission Denied Writing to `/var/log`**:
   - **Symptom**: When testing `./devops-heartbeat.sh` directly in the shell as user `josemanco`, it failed with:
     ```text
     ./devops-heartbeat.sh: line 4: /var/log/devops-heartbeat.log: Permission denied
     ```
   - **Root Cause**: On Arch Linux, `/var/log` is owned by `root:root` with mode `755`. Unprivileged users cannot create or append files there directly.
   - **Resolution**: Verified that executing the script via `sudo` or allowing systemd to invoke it works seamlessly, because system-level systemd units execute with root privileges by default.

2. **Typo in Service Install Target**:
   - **Symptom**: During initial authoring of `devops-heartbeat.service`, I noticed a typo in the `[Install]` section: `WantedBy=multy-user.target`.
   - **Impact & Fix**: While the timer unit (`devops-heartbeat.timer`) directly triggers the service and works regardless of the service's `[Install]` section, having an invalid target name would cause failures if someone ever attempted to enable the service directly via `systemctl enable devops-heartbeat.service`. I corrected the directive to `WantedBy=multi-user.target` and ran `sudo systemctl daemon-reload`.

---

## Conclusion & Key Takeaways
- Replacing cron with systemd timers provides superior observability: `systemctl list-timers` displays precise countdowns and execution history without needing to parse syslog.
- Systemd timers cleanly separate **scheduling logic** (`.timer`) from **execution logic** (`.service`), making unit testing and manual ad-hoc triggering (`systemctl start devops-heartbeat.service`) trivial.
- The `Persistent=true` directive provides robust recovery guarantees for transient environments, VMs, or laptops that experience frequent reboots or sleep cycles.
