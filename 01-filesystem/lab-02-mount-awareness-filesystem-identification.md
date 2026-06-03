# Lab 02 — Mount Awareness & Filesystem Identification

---

## Objective

- Understand how Linux organizes storage through mounted filesystems
- Learn to distinguish between local storage, memory-based filesystems, and externally mounted systems

This skill is critical for accurate disk usage analysis, troubleshooting, and safe system navigation.

---

## Environment

- **Distribution:** Ubuntu (WSL)
- **Platform:** WSL
- **User Context:** Standard user (sudo privileges available)

---

## Scenario

Simulating a system inspection to determine where data is actually stored and how different filesystems are attached.

This includes identifying:

- Local Linux storage
- Memory-based filesystems
- External storage (Windows via WSL)

---

## Technical Concepts Covered

- Filesystem mounting
- Block device inspection
- Filesystem types (ext4, tmpfs, overlay, 9p)
- Disk usage vs filesystem boundaries
- Mount point behavior

---

## Commands Used

```bash
lsblk
df -h
df -Th
mount | less
cd /
pwd
cd /mnt
ls
cd /mnt/c
pwd
ls
df -h .
```

---

## Procedure

1. Used `lsblk` to identify block devices and their mount points.
2. Ran `df -h` to view disk usage across mounted filesystems.
3. Ran `df -Th` to identify filesystem types and understand behavior differences.
4. Used `mount | less` to inspect the full mount configuration and layering.
5. Navigated to `/` to confirm the primary Linux filesystem.
6. Navigated to `/mnt` to identify mount points.
7. Entered `/mnt/c` to access Windows storage through WSL.
8. Used `df -h .` inside `/mnt/c` to confirm the active filesystem for that directory.

---

## Results

- Successfully identified multiple mounted filesystems within the system
- Confirmed that `/` is backed by a Linux filesystem (ext4)
- Observed `tmpfs` entries representing memory-based storage
- Verified that `/mnt/c` is a separate filesystem (Windows via 9p)
- Observed `/mnt/c` at 100% usage while Linux root remained unaffected
- Encountered permission restrictions when accessing certain Windows system files

---

## Evidence

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdc       1007G  3.1G  953G   1% /
C:\             58G   58G   228M 100% /mnt/c
tmpfs           940M     0  940M   0% /run

ls: cannot access 'pagefile.sys': Permission denied
```

---

## Key Takeaways

- Linux presents a unified directory tree but operates on multiple mounted filesystems
- A directory does not guarantee local storage
- Filesystem type determines behavior and persistence
- Disk usage must be evaluated per filesystem, not per path
- Mounted systems can have independent permissions and limitations

---

## What This Demonstrates

- Linux uses mounts to attach different storage systems into one structure
- Filesystem boundaries directly impact access, performance, and storage reporting
- The same commands behave consistently across different filesystems, but results differ based on the underlying system

---

## Security / Administration Relevance

- Prevents misinterpretation of disk usage and storage availability
- Helps avoid performing operations on unintended filesystems
- Supports accurate troubleshooting of storage-related issues
- Reinforces awareness of cross-system permission differences (Linux vs Windows in WSL)

---

## Time Spent

60 minutes (Execution + validation + documentation)

---

## Conclusion

This lab validated that Linux storage is not a single continuous system but a collection of mounted filesystems. Understanding mount points and filesystem types is essential for correctly interpreting disk usage, navigating systems safely, and performing reliable administrative tasks.
