### Processes, Pipes & IAM-Relevant Commands

# Goal
- Understand what a process is (PID, PPID, state, resource usage)
- Inspect running processes with ps, top, pgrep
- Control processes with kill (SIGTERM vs SIGKILL)
- Chain commands with pipes (|) and redirection (>, >>, 2>, 2>&1)
- Know the commands used on real IAM tickets: who, w, last, lastlog, ss, systemctl

# Theory

`What is a process?`
- A running instance of a program
- Every process has:
  PID   = process ID (unique number)
  PPID  = parent process ID (who started it)
  USER  = who owns it
  STAT  = state: R running, S sleeping, Z zombie, T stopped
  %CPU / %MEM = resource usage
  COMMAND = what is running

## ps — snapshot of processes

- *ps aux*  → BSD style. Columns: USER PID %CPU %MEM VSZ RSS STAT COMMAND
  Best for RESOURCE questions ("what's using CPU?")
- *ps -ef*  → SysV style. Columns: UID PID PPID C STIME TTY TIME CMD
  Best for PARENT-CHILD questions ("who started this?")
- Both show all processes — the difference is the columns, not the capability

## top — live process monitor

- *top*   live view, updates every ~3s. Press q to quit
  Shows: load average, tasks, CPU/MEM summary, process list (sorted by CPU)
  Keys: q=quit, k=kill, M=sort by memory, P=sort by CPU, 1=per-core CPU
- First command to run on a "slow server" ticket

## kill — send a signal

- *kill <PID>*     default = SIGTERM (15) — polite "please stop"
- *kill -15 <PID>*  SIGTERM — same as default
- *kill -9 <PID>*   SIGKILL — force kill, CANNOT be caught, skips cleanup
- *kill -1 <PID>*   SIGHUP — reload config (many services)

  Signal | Number | Meaning          | When to use
  -------|--------|------------------|------------------------
  HUP    | 1      | hangup / reload  | reload config
  TERM   | 15     | polite terminate | ALWAYS try first
  KILL   | 9      | force kill       | last resort only

- *pgrep -a bash*   find PID + command of processes matching "bash"
- *pgrep -u itachi* all processes owned by itachi

## Pipe | — chain commands

- *cmd1 | cmd2*   stdout of cmd1 becomes stdin of cmd2
- Both commands run at the SAME TIME, connected by a data stream
- NOT sequential execution — it is DATA FLOW

  ps aux ──stdout──► grep bash ──stdout──► terminal

## Redirection — > >> 2> 2>&1

- *cmd > file*     stdout to file, OVERWRITE
- *cmd >> file*    stdout to file, APPEND
- *cmd 2> file*    stderr to file
- *cmd 2>&1*       stderr to wherever stdout goes
- *cmd &> file*    both stdout and stderr (bash shortcut)
- *cmd &*          run in background, prompt returns immediately
- *jobs*           list background jobs of this shell
- *fg %1*          bring job 1 to foreground

## IAM-relevant commands (ticket tools)

- *who*             who is logged in right now (simple)
- *w*               who is logged in + what they're doing + load
- *last | head*     recent login history (from /var/log/wtmp)
- *lastlog*         last login per user (from /var/log/lastlog)
- *ss -tlnp*        listening TCP ports + owning process
                    -t TCP, -l listening, -n numeric, -p process
- *systemctl status ssh*    service state (active/inactive/failed) + Main PID
- *systemctl --failed*      list failed services

# The `< >` placeholder rule

When docs say:
  kill <PID>
  chmod <octal> <file>

The `< >` are PLACEHOLDERS — replace with actual values. Do NOT type the brackets.

  ❌ kill <12345>        → bash tries to open file "12345" as stdin → syntax error
  ✅ kill 12345          → correct

Convention in man pages:
  <required>   = must supply
  [optional]   = may omit
  ...          = repeatable

Examples:
  man kill:   kill [OPTIONS] <PID>
  man chmod:  chmod [OPTIONS] <MODE> <FILE>

# Lab A — Process Inspection (results)

- ps aux → all processes, columns USER PID %CPU %MEM VSZ RSS STAT COMMAND
- ps -ef → PPID column visible (kernel threads have PPID=2, systemd PPID=0)
- pgrep -a bash → found two bash shells (9866, 12045)
- sleep 600 & → background job [1] 12351
- kill 12351 + 12460 → both terminated cleanly
- [1]-  Terminated  /  [2]+  Terminated  (job numbers: + = most recent)

# Lab B — Pipes & Filters (results)

- ps aux | grep itachi | grep -v grep → all itachi processes
  Noticed: PID 9802 STAT Z <defunct> = zombie (parent hasn't reaped)
- ps aux | awk '{print $1}' | sort | uniq -c | sort -rn
  → counts per user, but includes "1 USER" (header row)
  Fix: awk 'NR>1 {print $1}' to skip header
- ls -l /srv/iamlab | grep '^d' → only directories
- find /srv/iamlab -type d -exec ls -ld {} \;
  → permission denied inside hr/drop/finance (correct: itachi not in those groups)
  → lesson: r = list names, x = enter/access contents — DIFFERENT bits
- cat /etc/passwd | cut -d: -f1 | tail -5 → alex, taylor (our lab users) at end

# Lab C — IAM-Relevant Commands (results)

- who → itachi on seat0 + tty2, 2026-10-31 07:58
- w   → 1 user, load 0.26 / 0.18 / 0.12, running gnome-session
- last | head → login history Oct 25-31 (every bootcamp day!)
  Note: last requires wtmpdb on Ubuntu 26.04 (install first)
  "still logged in" = migration artefact on new wtmpdb
- lastlog → deprecated; lastlog2 on fresh install needs PAM record
  ("Cannot open database" until first PAM login)
  Lesson: distro drift — same concept, different package
- ss -tlnp → 631 (CUPS), 8324/8325 (WSD printer discovery)
  No :22 (SSH not installed — normal for workstation)
  All bound to 127.0.0.1 = localhost only (good security posture)
- systemctl status ssh → "Unit ssh.service could not be found"
  = NOT INSTALLED, not "down" — different diagnosis
- systemctl --failed → 0 loaded units = healthy
- systemctl list-units --type=service --state=running → 37 services
  Notable: docker, containerd (containers available for Week 5-9)
            polkit (privilege escalation — IAM-relevant)
            systemd-logind (login session management)
            rsyslog (audit trail)
            unattended-upgrades (governance/patches)

# Checkpoint Answers

`Q1 — kill vs kill -9?`
kill (SIGTERM, 15) = polite terminate — process can catch/ignore.
kill -9 (SIGKILL) = force kill — cannot be caught, skips cleanup.
Use TERM first always. Use KILL only when TERM fails (last resort).

`Q2 — Why sort before uniq -c?`
uniq only collapses ADJACENT duplicates. Without sorting first, the same
value scattered through the list won't be counted together.
Pattern: sort | uniq -c | sort -rn.

`Q3 — "user X can't log in" — three commands?`
1. id user → does the account exist? what groups? is it locked?
2. lastlog | grep user → when did they last log in? never?
3. systemctl status systemd-logind → is the session manager running?
   (also: /var/log/auth.log for auth errors — 4th option)
