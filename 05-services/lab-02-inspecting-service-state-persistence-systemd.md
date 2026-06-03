# Lab 02 — Inspecting Service State and Persistence with systemd

---

## Objective

- Understand how Linux manages services using systemd
- Distinguish between running state and boot persistence
- Inspect how services are defined and executed

---

## Environment

- **Distribution:** Ubuntu
- **Platform:** WSL
- **User Context:** Standard user with sudo privileges

---

## Scenario

An administrator needs to determine:

- Which services are currently running
- Which services are configured to start at boot
- How a service is defined internally

This simulates a real-world situation where understanding service behavior and persistence is required for troubleshooting or system control.

---

## Technical Concepts Covered

- systemd service management
- Unit files
- Active vs enabled states
- ExecStart behavior
- multi-user.target
- Service inspection and verification

---

## Commands Used

```bash
systemctl list-unit-files --type=service
systemctl list-units --type=service
systemctl status ssh
systemctl cat ssh
systemctl is-enabled ssh
sudo systemctl start ssh
```

---

## Procedure

1. Listed all available service unit files to observe which services are enabled or disabled.
2. Listed active services to compare runtime state versus configured state.
3. Inspected the SSH service using `systemctl status ssh` to view its current state, process ID, and service details.
4. Started the SSH service using `sudo systemctl start ssh`.
5. Rechecked the service status to confirm it was now running.
6. Viewed the full unit file using `systemctl cat ssh` to understand how the service is defined and what command is executed.
7. Verified persistence using `systemctl is-enabled ssh` to determine if the service would start at boot.

---

## Results

- SSH service was initially inactive and disabled
- After starting, SSH became active (running)
- SSH remained disabled — it would not persist after reboot
- The unit file revealed the exact command used to start the service
- The service was tied to `multi-user.target`, indicating its role in system startup

---

## Evidence

```
Active: active (running)
Loaded: disabled
ExecStart=/usr/sbin/sshd -D $SSHD_OPTS
```

---

## Key Takeaways

- A service can be running without being persistent
- Enabled state determines if a service starts at boot
- systemd uses unit files as the source of truth
- Service behavior is defined, not random

---

## What This Demonstrates

Linux separates:

- **Execution** — running state (is it running right now?)
- **Configuration** — enabled state (will it start at boot?)

This lab confirms that persistence is controlled through system configuration, not runtime behavior.

---

## Security / Administration Relevance

- Helps identify unauthorized persistent services
- Supports troubleshooting services that restart unexpectedly
- Builds awareness of system startup behavior
- Enables controlled service management

---

## Time Spent

60 minutes

---

## Conclusion

This lab validated how systemd manages services through separation of runtime state and persistence. It reinforced that services are controlled through configuration, and understanding unit files is essential for system administration.
