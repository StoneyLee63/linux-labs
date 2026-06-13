# Lab 04 — Mount Control Awareness in WSL vs Traditional fstab Behavior

---

## Objective

- Understand how persistent mount control normally works through `/etc/fstab`
- Verify how that behavior changes in a WSL environment where mounts are managed automatically outside of traditional Linux boot control

This matters because administrators need to know which layer is actually controlling storage behavior before making configuration changes.

---

## Environment

- **Distribution:** Ubuntu (WSL)
- **Platform:** WSL
- **User Context:** Standard user with sudo privileges

---

## Scenario

Simulating a mount configuration check to determine whether `/etc/fstab` is actively controlling filesystem mounts.

This lab focused on:

- Checking fstab state
- Inspecting live mount behavior
- Comparing configured control vs actual control in WSL

---

## Technical Concepts Covered

- `/etc/fstab` purpose
- Automatic mount handling in WSL
- Active mount inspection
- Filesystem layering (`ext4`, `tmpfs`, `overlay`, `9p`)
- Control plane awareness in Linux environments

---

## Commands Used

```bash
cat /etc/fstab
mount
df -h
lsblk -f
sudo mount -a
```

---

## Procedure

1. Viewed `/etc/fstab` to determine whether persistent mount rules were configured.
2. Ran `mount` to inspect all active mounted filesystems and identify how the system was currently assembled.
3. Used `df -h` to verify mounted filesystems, usage, and mount points.
4. Ran `lsblk -f` to inspect block devices, filesystem types, UUIDs, and mount relationships.
5. Executed `sudo mount -a` as a safe validation step to confirm whether any fstab-driven mounts would be processed.

---

## Results

- `/etc/fstab` returned `UNCONFIGURED FSTAB FOR BASE SYSTEM`
- Active mounts were still fully present despite fstab being unconfigured
- The system showed multiple mounted layers including:
  - Linux root on `ext4`
  - Memory-based `tmpfs`
  - WSL `overlay`
  - Windows integration through `9p`
- `/mnt/c` remained available through WSL-managed mounting
- `sudo mount -a` produced no changes — no configured fstab entries to process

---

## Evidence

```
UNCONFIGURED FSTAB FOR BASE SYSTEM

/dev/sdc on / type ext4
drivers on /usr/lib/wsl/drivers type 9p

Filesystem      Size  Used Avail Use% Mounted on
/dev/sdc       1007G  3.1G  953G   1% /
C:\             58G   58G   147M 100% /mnt/c
```

---

## Key Takeaways

- `/etc/fstab` is the traditional source of persistent mount control in standard Linux systems
- In WSL, mounts are handled automatically by the Windows + WSL integration layer
- Active system state must be verified with `mount`, `df`, and `lsblk` — not assumed from configuration files alone
- A system can be fully mounted even when fstab is empty or unused

---

## What This Demonstrates

- Mount authority depends on environment
- WSL changes the normal Linux mount control model
- Configuration files only matter if the system actually uses them
- Effective administration starts with identifying the real control layer

---

## Security / Administration Relevance

- Prevents wasted troubleshooting against the wrong configuration layer
- Builds awareness of environment-specific behavior
- Reduces risk of making ineffective or misleading mount changes
- Reinforces the need to verify actual system control paths before administration work

---

## Time Spent

60 minutes (Execution + validation + documentation)

---

## Conclusion

This lab confirmed that while `/etc/fstab` is the standard mechanism for persistent mounts in traditional Linux systems, WSL bypasses that model and manages mounts externally. The rep strengthened environment awareness by showing that the real source of control must be identified before making administrative decisions.
