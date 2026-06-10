# Lab 03 — Mapping Service Dependencies and Relationships with systemd

---

## Objective

- Understand how Linux services are connected by analyzing dependencies, reverse dependencies, targets, and socket activation
- Determine why a service runs, not just how to control it

---

## Environment

- **Distribution:** Ubuntu
- **Platform:** WSL
- **User Context:** Standard user with sudo privileges

---

## Scenario

An administrator needs to determine:

- What a service depends on
- What depends on the service
- Where the service is anchored in the system
- What can trigger the service indirectly

This simulates real-world troubleshooting and security analysis where services may start due to indirect causes.

---

## Technical Concepts Covered

- systemd dependency chains
- `list-dependencies`
- Reverse dependencies
- Targets (`multi-user`, `graphical`)
- Socket activation
- Service relationships

---

## Commands Used

```bash
systemctl list-dependencies ssh
systemctl list-dependencies --reverse ssh
systemctl cat ssh
systemctl list-dependencies multi-user.target
systemctl list-dependencies graphical.target
systemctl status ssh.socket
systemctl cat ssh.socket
```

---

## Procedure

1. Listed dependencies of SSH:
```bash
systemctl list-dependencies ssh
```
Observed foundational system layers required for SSH to run.

2. Listed reverse dependencies:
```bash
systemctl list-dependencies --reverse ssh
```
Identified system targets that rely on SSH.

3. Inspected SSH unit file:
```bash
systemctl cat ssh
```
Found `WantedBy=multi-user.target`.

4. Explored system targets:
```bash
systemctl list-dependencies multi-user.target
systemctl list-dependencies graphical.target
```
Observed grouping of services into system states.

5. Investigated socket activation:
```bash
systemctl status ssh.socket
systemctl cat ssh.socket
```
Identified that SSH can be triggered through socket listening.

---

## Results

**SSH dependencies include:**
- `system.slice`
- `sysinit.target`

**SSH is required by:**
- `multi-user.target`
- `graphical.target`

**Target observations:**
- `multi-user.target` acts as the core runtime environment grouping services like SSH
- `graphical.target` builds on top of `multi-user.target` for full system state

**Socket activation:**
- `ssh.socket` was active and listening on port 22 even when `ssh.service` was not running
- `ssh.socket` configuration includes:
  - `ListenStream=0.0.0.0:22`
  - `Triggers=ssh.service`
  - `WantedBy=sockets.target`

---

## Key Takeaways

- Services depend on foundational system layers that must exist before they can run
- Services are grouped into targets representing system states
- Reverse dependencies show what pulls a service into execution
- Socket activation allows services to start indirectly — a port can be open without the service running
- A service can be triggered without manual execution

---

## What This Demonstrates

Linux services operate within a system of:

- **Dependencies** — what must exist for the service to run
- **Targets** — where the service belongs in the system state
- **Triggers** — what activates the service

Understanding these relationships is critical for system control, troubleshooting, and security analysis.

---

## Security / Administration Relevance

- Identifies indirect service activation paths
- Helps detect hidden persistence mechanisms
- Improves troubleshooting of unexpected service behavior
- Builds system-level awareness for defensive and offensive operations

---

## Time Spent

80 minutes

---

## Conclusion

This lab demonstrated that Linux services are not isolated processes but part of a structured system of relationships. It reinforced the importance of understanding dependencies, targets, and triggers to fully explain why a service runs.
