# Day 3 — Lab: Processes & System

**Goal:** Confidently identify process states from the command line, deliberately create and recognize a zombie process, correctly kill an entire process tree (not just its parent), and run a background job that survives you logging out.

**Prerequisites:** A Linux environment (Ubuntu via WSL2, a VM, or a cloud instance). Install the tools used today:
```bash
sudo apt install htop lsof strace psmisc   # Debian/Ubuntu
sudo yum install htop lsof strace psmisc   # RHEL/CentOS/Amazon Linux
python3 --version                          # confirm python3 exists (used to create a zombie deterministically)
```
`psmisc` provides `pstree` and `killall`, used below.

---

### Lab 1 — Read process states with `ps` and `top`

1. Open two terminals (or two panes/tmux windows). In terminal A, start something that will sit in interruptible sleep:
   ```bash
   sleep 300 &
   ```
2. In terminal B, find it and read its state:
   ```bash
   ps -eo pid,ppid,stat,cmd | grep '[s]leep 300'
   ```
   Confirm the `STAT` column shows `S` (interruptible sleep).
3. Suspend it, then observe the state change:
   ```bash
   kill -STOP %1        # from terminal A, or use the PID from terminal B with kill -STOP <pid>
   ps -eo pid,ppid,stat,cmd | grep '[s]leep 300'   # STAT should now show T
   kill -CONT %1         # resume it
   ```
4. Launch `top`, press `1` to see per-core CPU, then `P` and `M` to toggle sort order. Note the load average in the header and compare it to `nproc` (number of cores) on your machine.

**Success criteria:** You can point at a process's `STAT` column and correctly say whether it's running, sleeping, or stopped, and you understand what your machine's load average means relative to its core count.

---

### Lab 2 — Core activity: deliberately create and identify a zombie process

Bash alone reaps background jobs unpredictably fast, so we'll use a tiny Python script to create a **deterministic** zombie for observation.

1. Create the script:
   ```bash
   cat > /tmp/make_zombie.py << 'EOF'
   import os, time

   pid = os.fork()
   if pid == 0:
       # child: exit immediately
       os._exit(0)
   else:
       # parent: deliberately does NOT call os.wait() — child becomes a zombie
       print(f"Parent PID {os.getpid()}, child PID {pid} is now a zombie for 60s")
       time.sleep(60)
   EOF
   ```
2. Run it in the background:
   ```bash
   python3 /tmp/make_zombie.py &
   ```
3. Within the 60-second window, find the zombie:
   ```bash
   ps aux | grep 'Z'          # look for STAT column showing Z or Z+, COMMAND showing <defunct>
   ps -eo pid,ppid,stat,cmd | awk '$3 ~ /Z/'
   ```
4. Try to kill it and observe nothing happens:
   ```bash
   kill -9 <zombie_pid>       # exits without error, but the entry remains — it's already dead
   ```
5. Confirm the fix is killing the **parent**, not the zombie:
   ```bash
   ps -eo pid,ppid,stat,cmd | grep '[m]ake_zombie'   # find the parent PID
   kill <parent_pid>
   ps aux | grep 'Z'          # zombie entry is now gone — reaped by init after re-parenting
   ```

**Success criteria:** You can explain, using your own terminal output as evidence, why `kill -9` on a zombie's PID does nothing, and why killing the parent (or waiting for it to exit naturally) clears the zombie.

---

### Lab 3 — Core activity: build a process tree and kill it correctly

1. Create a script that spawns a small worker tree:
   ```bash
   cat > /tmp/tree_demo.sh << 'EOF'
   #!/usr/bin/env bash
   worker() { while true; do sleep 5; done; }
   worker & worker & worker &
   wait
   EOF
   chmod +x /tmp/tree_demo.sh
   ```
2. Launch it in the background and inspect the tree:
   ```bash
   /tmp/tree_demo.sh &
   pstree -p $(jobs -p)          # visualize parent + 3 workers, with PIDs
   ps --forest -o pid,ppid,cmd -g $(ps -o pgid= -p $(jobs -p) | tr -d ' ')
   ```
3. **Do it wrong first, on purpose:** kill only the top-level script PID and observe the workers survive as orphans:
   ```bash
   kill $(jobs -p)                     # kills only tree_demo.sh
   sleep 1
   ps -ef | grep '[w]orker\|sleep 5'   # the 3 sleep loops are still running, now reparented to PID 1
   pstree -p 1 | grep -A2 sleep         # confirm they're now children of init/systemd
   ```
4. **Do it right:** find the process group and kill the whole group at once (relaunch first if you killed everything above):
   ```bash
   /tmp/tree_demo.sh &
   PGID=$(ps -o pgid= -p $! | tr -d ' ')
   kill -TERM -$PGID                    # note the leading minus — targets the whole process group
   sleep 1
   ps -ef | grep '[w]orker\|sleep 5'    # nothing left
   ```

**Success criteria:** You've reproduced the "orphaned children survive a single-PID kill" failure mode yourself, then fixed it with a process-group kill, and can explain the difference between `kill <pid>` and `kill -<pgid>`.

---

### Lab 4 — Core activity: background jobs with `&`, `nohup`, and `disown`

1. Start a plain background job and inspect it with job control:
   ```bash
   sleep 200 &
   jobs                 # shows [1]+ Running   sleep 200 &
   ```
2. Bring it to the foreground, suspend it, and resume it in the background:
   ```bash
   fg %1                # brings sleep 200 to foreground
   # press Ctrl-Z to suspend it
   jobs                 # shows [1]+ Stopped   sleep 200
   bg %1                 # resumes it in the background
   ```
3. Demonstrate the `SIGHUP`-on-logout problem and its fix. Open a fresh SSH session (or a new terminal simulating one) and run:
   ```bash
   sleep 300 &
   echo "job PID: $!"
   exit                  # close this session/terminal entirely
   ```
   Reconnect and check whether the process is still alive: `ps -ef | grep '[s]leep 300'` — on many systems it will have been killed by `SIGHUP` (behavior varies by shell config, but this is the failure mode `nohup` prevents).
4. Now do it correctly with `nohup`:
   ```bash
   nohup sleep 300 > /tmp/nohup_test.log 2>&1 &
   echo "job PID: $!"
   exit
   ```
   Reconnect and confirm it survived: `ps -ef | grep '[s]leep 300'`.
5. Demonstrate `disown` as the after-the-fact rescue for a job you forgot to `nohup`:
   ```bash
   sleep 300 &
   disown -h %1          # detach from this shell's job table without changing SIGHUP disposition mid-run
   jobs                  # job no longer listed, but the process is still running (check with ps)
   ```

**Success criteria:** You can state, from direct observation, what happens to a plain `&` background job when its shell session ends, and you've used both `nohup` (at launch) and `disown` (after the fact) to prevent it.

---

### Lab 5 — Inspect a running process with `htop`, `lsof`, and `strace`

1. Start a simple long-running server to inspect:
   ```bash
   python3 -m http.server 8899 &
   PID=$!
   ```
2. In `htop`: find it (press `/` to search for `http.server`), note its `%CPU`/`%MEM`/`STAT` columns, then try renicing it from inside `htop` with `F7`/`F8` (raise/lower niceness) — confirm the `NI` column changes.
3. Use `lsof` to see what it has open:
   ```bash
   lsof -p $PID                  # every file descriptor: cwd, executable, libraries, the listening socket
   lsof -i :8899                 # confirm this PID is the one bound to port 8899
   ```
4. Use `strace` to watch it handle a request live:
   ```bash
   strace -f -e trace=network,read,write -p $PID &
   curl -s localhost:8899 > /dev/null
   # observe the accept()/read()/write() syscalls printed for the incoming connection
   ```
5. Stop the `strace` (Ctrl-C on that job, or `kill` its PID), then shut down the server:
   ```bash
   kill $PID
   ```

**Success criteria:** You've correlated the same process across three different tools (`htop`'s live view, `lsof`'s open-file/socket view, `strace`'s syscall-level view) and can say what each one uniquely tells you that the others don't.

---

### Cleanup

```bash
# Kill any leftover lab processes
pkill -f make_zombie.py 2>/dev/null
pkill -f tree_demo.sh 2>/dev/null
pkill -f "http.server 8899" 2>/dev/null
kill %1 %2 %3 2>/dev/null   # clear any remaining shell jobs

# Remove temp files created during the lab
rm -f /tmp/make_zombie.py /tmp/tree_demo.sh /tmp/nohup_test.log nohup.out
```

### Stretch challenge

Write a one-line `ps`/`awk` pipeline that finds the single process consuming the most CPU **right now**, prints its PID and command, then answer (without running anything else): would `kill -9` on that PID also stop any of its children? Justify your answer using what you learned in Lab 3, then verify it by actually building a small CPU-hogging parent/child pair and checking.
