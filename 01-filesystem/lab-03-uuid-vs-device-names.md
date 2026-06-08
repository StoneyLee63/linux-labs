# Lab 03 — Filesystem Identity Using UUID vs Device Names

---

## Objective

- Understand how Linux identifies filesystems using UUIDs instead of device names
- Learn why this distinction is critical for system stability and persistent mounting

---

## Environment

- **Distribution:** Ubuntu (WSL)
- **Platform:** WSL
- **User Context:** Standard user with sudo privileges

---

## Scenario

Simulating a system inspection to determine how Linux reliably identifies storage devices across reboots and system changes.

This includes:

- Comparing device names to UUIDs
- Verifying filesystem identity
- Mapping UUIDs to active devices

---

## Technical Concepts Covered

- UUID (filesystem identity)
- Dynamic device naming (`/dev/sdX`)
- Filesystem metadata
- Symbolic linking (`/dev/disk/by-uuid/`)
- Persistent storage identification

---

## Commands Used

```bash
lsblk -f
sudo blkid
df -h
ls -l /dev/disk/by-uuid/
readlink -f /dev/disk/by-uuid/
```

---

## Procedure

1. Ran `lsblk -f` to view block devices along with filesystem type and UUID.
2. Used `sudo blkid` to extract UUIDs directly from filesystem metadata.
3. Ran `df -h` to observe mounted filesystems using device names.
4. Listed `/dev/disk/by-uuid/` to view how UUIDs map to device paths.
5. Used `readlink -f` on a UUID entry to trace it back to the actual device.

---

## Results

- Confirmed that each filesystem has a unique UUID
- Observed that `/dev/sdc` is associated with a consistent UUID
- Verified that UUID links point to the correct active device
- Identified that swap (`/dev/sdb`) has its own UUID
- Confirmed that device names and UUIDs represent the same filesystem through different layers

---

## Evidence

```
/dev/sdc: UUID="xxxx-xxxx" TYPE="ext4"

lrwxrwxrwx 1 root root 10 Mar 24 UUID -> ../../sdc

/dev/sdc
```

---

## Key Takeaways

- Device names (`/dev/sdX`) are not stable identifiers
- UUIDs are fixed and tied to filesystem identity
- Linux uses symbolic links to map UUIDs to active devices
- Reliable mounting depends on UUID, not device names

---

## What This Demonstrates

- Linux separates identity from labeling
- Filesystems maintain a consistent identity independent of device naming
- System reliability depends on referencing stable identifiers

---

## Security / Administration Relevance

- Prevents mount failures after reboot
- Ensures consistent storage configuration
- Reduces risk of attaching incorrect devices
- Supports reliable system administration and automation

---

## Time Spent

60 minutes (Execution + validation + documentation)

---

## Conclusion

This lab demonstrated that device names are temporary labels, while UUIDs provide stable identity for filesystems. Understanding this distinction is essential for configuring persistent mounts and maintaining system reliability across reboots and hardware changes.
