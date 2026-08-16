# Lab 04 — systemd Fundamentals: Service Lifecycle, Enable vs Mask

---

## Objective

Practiced managing services through systemd — checking runtime and boot-time state, starting/stopping/restarting a real service, controlling whether a service survives a reboot, and reading a unit file to understand what it's actually configured to do. This matters because service management is one of the most directly tested skill areas on the RHCSA, and the active/enabled distinction is the single most common point of real-world misconfiguration.

---

## Environment

- Distribution: Ubuntu (WSL2, systemd enabled)
- Platform: WSL
- User Context: standard user (`ltksol`) with `sudo` for privileged operations

---

## Scenario

A service needs to be stopped for inspection, restarted, and then explicitly controlled for boot persistence — separately from whether it's currently running. Also covers the stronger control of masking a service outright, including testing whether masking works on a service name that isn't currently installed on the system, which simulates pre-hardening a system against services that might be pulled in later by a package.

---

## Technical Concepts Covered

- systemd unit file anatomy (`[Unit]`, `[Service]`, `[Install]`)
- Active state vs. enabled state as independent, separately-tracked properties
- `start`/`stop`/`restart` as managed signal sequences (same mechanism as Day 1's `kill`)
- `reload` as an opt-in, per-service capability (not universally supported)
- `disable` vs `mask` as two different strengths of "don't run"
- Drop-in configuration overrides (`systemctl edit` mechanism, observed via WSL's `wsl.conf` drop-in)
- Distro-level SysV compatibility shim behavior on Ubuntu

---

## Commands Used

```bash
systemctl status systemd-timesyncd
systemctl is-active systemd-timesyncd
systemctl is-enabled systemd-timesyncd
sudo systemctl stop cron
systemctl is-active cron
sudo systemctl start cron
systemctl status cron | grep -E "Active|Main PID"
sudo systemctl restart cron
sudo systemctl reload cron 2>&1
sudo systemctl disable cron
systemctl is-active cron
systemctl is-enabled cron
sudo systemctl enable cron
ls -l /etc/systemd/system/multi-user.target.wants/ | grep cron
systemctl cat cron
sudo systemctl mask bluetooth 2>&1
systemctl is-enabled bluetooth 2>&1
sudo systemctl unmask bluetooth 2>&1
systemctl status cron --no-pager
systemctl list-unit-files cron.service
```

---

## Procedure

1. Checked `systemd-timesyncd` full status — discovered an unscripted Drop-In override file (`wsl.conf`) modifying the vendor unit without editing it directly.
2. Confirmed active and enabled state independently using `is-active` / `is-enabled`.
3. Stopped `cron`, verified `inactive`, then started it again and confirmed a new Main PID — proving stop/start produces a brand new process.
4. Ran `restart` (succeeded, new PID) followed by `reload` (failed — not supported by this unit) to demonstrate the difference between the two.
5. Disabled `cron` and verified it stayed active while flipping to disabled — proof the two states are tracked independently. Observed Ubuntu's SysV compatibility shim messaging during this step.
6. Re-enabled `cron` and verified the actual symlink created on disk in `/etc/systemd/system/multi-user.target.wants/`.
7. Read the full real unit file with `systemctl cat` to confirm `ExecStart=`, `Restart=`, and `WantedBy=` directives.
8. Masked a service name (`bluetooth`) that isn't installed on this system at all — confirmed systemd masked it anyway by symlinking the name to `/dev/null`. Unmasked it afterward.
9. Ran a final state check confirming both active and enabled status matched intent.

---

## Results

- Confirmed `systemctl status` reports both runtime state and boot-time state in a single call, and that a service can be in any combination of the two.
- Confirmed `stop`/`start` produces a genuinely new process (new PID), not a resumed one — same logic as a recreated file getting a new inode.
- Confirmed `reload` is not universally available — it depends on the unit defining an `ExecReload=` directive, and systemd refuses the request cleanly when it isn't present.
- Confirmed `disable` only affects boot-time behavior and does not touch a currently running process.
- Confirmed `enable`/`disable` are literal, inspectable symlinks on disk — not hidden internal flags.
- Confirmed `mask` works even on service names that don't currently exist on the system, pre-emptively blocking any future installation of that service from being able to start without an explicit `unmask`.

---

## Evidence

```
Loaded: loaded (/usr/lib/systemd/system/systemd-timesyncd.service;
Drop-In: /usr/lib/systemd/system/systemd-timesyncd.service.d
          └─wsl.conf

Failed to reload cron.service: Job type reload is not applicable for unit cron.service.

Synchronizing state of cron.service with SysV service script with /usr/lib/systemd/systemd-sysv-install.
Removed "/etc/systemd/system/multi-user.target.wants/cron.service".

cron.service -> /usr/lib/systemd/system/cron.service

[Service]
ExecStart=/usr/sbin/cron -f -P $EXTRA_OPTS
Restart=on-failure
KillMode=process

Unit bluetooth.service does not exist, proceeding anyway.
Created symlink /etc/systemd/system/bluetooth.service -> /dev/null.

cron.service   enabled   enabled
```

---

## Key Takeaways

- Active and enabled are two completely independent properties — never assume one from the other, always check both when a task requires a service to survive a reboot.
- `disable` and `mask` are different strengths of control — disable blocks automatic starts, mask blocks all starts by anyone, including services that don't exist yet.
- A unit file's `[Service]` section is the literal command and restart policy systemd is running — there's no hidden behavior beyond what `systemctl cat` shows.

---

## What This Demonstrates

- systemd's start/stop mechanism is built on the same process-signal model from Day 1 — service management is process control with dependency and retry logic layered on top.
- "Enabled" is not an abstract database flag — it is a real, inspectable symlink in the filesystem, consistent with the inode-and-directory-entry model from earlier Earth domain labs.
- Distro implementation details (Ubuntu's SysV compatibility shim) can surface mid-task and need to be recognized as environment-specific, not a systemd-wide behavior.

---

## Security / Administration Relevance

The active/enabled split is responsible for a very common production incident pattern: a service is manually restarted during an incident, confirmed running, and the responder walks away — but it was never re-enabled, so it silently fails to come back after the next reboot. Knowing to check both states prevents this gap. The ability to `mask` a service before it's even installed is a real hardening technique — pre-blocking known-risky service names (bluetooth, avahi, etc.) on servers where they should never run, regardless of what a future package installation might pull in.

---

## Time Spent

75 minutes

---

## Conclusion

Validated the full systemd service lifecycle — status, start, stop, restart, reload (and its limits), enable, disable, and mask — along with direct filesystem verification of what each state actually means. The active/enabled split and the disable/mask distinction are the two ideas that carry the most real-world and exam weight from this session. This builds directly toward Day 3's systemd deep dive into target dependencies and boot diagnostics, starting from the `After=remote-fs.target` dependency already observed in cron's unit file.
