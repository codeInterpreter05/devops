# Day 7 — Lab: Systemd & Cron

**Goal:** Write a systemd service for a Python script, enable it, and prove to yourself — by killing the process and watching the journal — that systemd actually restarts it automatically. Then set up a complementary cron job and verify it ran.

**Prerequisites:** A Linux environment with systemd (Ubuntu via WSL2 with systemd enabled, a VM, or a cloud instance — plain Docker containers usually do *not* run systemd as PID 1, so use a real VM or WSL2 for this lab). `python3` installed. `sudo` access. Verify systemd is actually running: `ps -p 1 -o comm=` should print `systemd`.

---

### Lab 1 — Write a Python script that can crash on demand

1. Create a working directory and the script:
   ```bash
   sudo mkdir -p /opt/pyworker
   sudo tee /opt/pyworker/worker.py > /dev/null <<'EOF'
   import time
   import sys

   print("worker starting", flush=True)
   counter = 0
   while True:
       counter += 1
       print(f"tick {counter}", flush=True)
       time.sleep(2)
       # Simulate a crash after ~30 seconds so you can also watch it die on its own
       if counter == 15:
           print("simulated crash!", file=sys.stderr, flush=True)
           sys.exit(1)
   EOF
   ```
2. Run it manually first to confirm it works and to see it exit non-zero on its own:
   ```bash
   python3 /opt/pyworker/worker.py
   ```
   Let it run until it prints `simulated crash!` and exits (~30s), and confirm with `echo $?` that the exit code is `1`.

**Success criteria:** The script runs standalone, prints ticks every 2 seconds, and exits with code `1` after the 15th tick.

---

### Lab 2 — Write the systemd unit file

1. Create the unit file:
   ```bash
   sudo tee /etc/systemd/system/pyworker.service > /dev/null <<'EOF'
   [Unit]
   Description=Day 7 lab Python worker
   After=network.target

   [Service]
   Type=simple
   ExecStart=/usr/bin/python3 /opt/pyworker/worker.py
   Restart=on-failure
   RestartSec=5
   StartLimitIntervalSec=60
   StartLimitBurst=5
   User=root
   WorkingDirectory=/opt/pyworker

   [Install]
   WantedBy=multi-user.target
   EOF
   ```
2. Reload systemd's unit cache so it picks up the new file:
   ```bash
   sudo systemctl daemon-reload
   ```

**Success criteria:** `systemctl cat pyworker` prints the unit file back with no syntax errors reported.

---

### Lab 3 — Enable, start, and verify it's running

1. Enable (boot-persistent) and start (right now) in one command:
   ```bash
   sudo systemctl enable --now pyworker
   ```
2. Check status:
   ```bash
   systemctl status pyworker
   ```
   Confirm: `Active: active (running)`, a `Main PID`, and recent log lines shown inline.
3. Confirm it's enabled for boot separately from running:
   ```bash
   systemctl is-enabled pyworker   # expect: enabled
   systemctl is-active pyworker    # expect: active
   ```

**Success criteria:** Both `is-enabled` and `is-active` print positive results, and `status` shows a live `Main PID` with ticks flowing.

---

### Lab 4 — Verify auto-restart on the built-in crash

1. Watch the journal live in one terminal:
   ```bash
   journalctl -u pyworker -f
   ```
2. In another terminal (or just wait ~30s), let the script hit its scripted `sys.exit(1)`. Observe in the journal: the `simulated crash!` stderr line, systemd logging that the process exited with code 1, a restart being scheduled, and then `worker starting` appearing again with a **new PID**.
3. Confirm the PID actually changed:
   ```bash
   systemctl show -p MainPID pyworker
   ```
   Run this right before and right after the crash and confirm the number is different.

**Success criteria:** You can point to the exact journal lines showing exit → restart scheduled → new process started, and confirm the `MainPID` changed across the crash.

---

### Lab 5 — Verify auto-restart on a manual kill (not just the scripted crash)

1. Get the current PID and kill it forcefully, simulating an unexpected crash rather than a clean exit:
   ```bash
   PID=$(systemctl show -p MainPID --value pyworker)
   sudo kill -9 $PID
   ```
2. Within a few seconds (per `RestartSec=5`), check status again:
   ```bash
   systemctl status pyworker
   ```

**Success criteria:** Despite a `SIGKILL`, the service shows `active (running)` again with a new PID — proving `Restart=on-failure` covers signal-based deaths, not just non-zero exit codes.

---

### Lab 6 — Inspect logs with `journalctl`

1. Full history for this unit:
   ```bash
   journalctl -u pyworker --no-pager
   ```
2. Only the last 20 lines:
   ```bash
   journalctl -u pyworker -n 20
   ```
3. Only from the last 5 minutes:
   ```bash
   journalctl -u pyworker --since "5 min ago"
   ```
4. Count how many times the service has restarted so far by counting "Started" lines:
   ```bash
   journalctl -u pyworker | grep -c "Started Day 7 lab Python worker"
   ```

**Success criteria:** You can produce a filtered log slice answering "how many times has this crashed and restarted since I started it" without scrolling manually.

---

### Lab 7 — Set up a complementary cron job

Set up a periodic job that logs a heartbeat every minute, independent of the always-on worker service.

1. Create a small script:
   ```bash
   sudo tee /opt/pyworker/heartbeat.sh > /dev/null <<'EOF'
   #!/bin/bash
   echo "$(date '+%F %T') heartbeat from cron" >> /var/log/pyworker-heartbeat.log
   EOF
   sudo chmod +x /opt/pyworker/heartbeat.sh
   sudo touch /var/log/pyworker-heartbeat.log
   sudo chmod 666 /var/log/pyworker-heartbeat.log
   ```
2. Add a crontab entry (edit your own user's crontab):
   ```bash
   crontab -e
   ```
   Add this line, using the **absolute path** to the script and explicit redirection:
   ```
   * * * * * /opt/pyworker/heartbeat.sh >> /var/log/pyworker-heartbeat-cron-stderr.log 2>&1
   ```
3. Wait at least 2 minutes, then verify:
   ```bash
   tail -n 5 /var/log/pyworker-heartbeat.log
   ```

**Success criteria:** At least two timestamped heartbeat lines appear, roughly 60 seconds apart, proving cron actually invoked the script on schedule.

---

### Cleanup

```bash
# Stop and fully remove the systemd service
sudo systemctl disable --now pyworker
sudo rm /etc/systemd/system/pyworker.service
sudo systemctl daemon-reload
sudo systemctl reset-failed pyworker 2>/dev/null

# Remove the cron entry (open the editor and delete the line, or clear entirely)
crontab -e     # delete the heartbeat line manually
# or, if this crontab ONLY contained lab entries:
# crontab -r

# Remove lab files
sudo rm -rf /opt/pyworker
sudo rm -f /var/log/pyworker-heartbeat.log /var/log/pyworker-heartbeat-cron-stderr.log
```

### Stretch challenge

Convert the cron heartbeat job into an equivalent systemd timer + service pair (`pyworker-heartbeat.service` running the script `Type=oneshot`, plus `pyworker-heartbeat.timer` with `OnCalendar=*-*-* *:*:00` and `Persistent=true`). Enable the `.timer` (not the `.service` directly), confirm it appears in `systemctl list-timers`, and explain in one sentence what `Persistent=true` buys you here that the plain cron version doesn't have.
