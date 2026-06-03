# Lab 03 — Umask & Default Permission Creation

---

## Objective

- Understand how Linux assigns permissions at file and directory creation time
- Learn how to predict permissions using `umask` instead of reacting after the fact
- Build control over system behavior at object creation level

Why it matters: this is foundational for system administration, security hardening, and access control design in Linux.

---

## Environment

- **Distribution:** Linux (Ubuntu/Debian-based shell)
- **Platform:** Terminal / CLI
- **User Context:** Standard user
- **Shell:** Bash

---

## Scenario

Investigating how Linux decides default permissions when new files and directories are created.

Instead of manually fixing permissions after creation, the goal was to understand the system-level rule that defines them at birth.

Focus:

- Why files and directories start with different permissions
- How `umask` silently shapes everything before you see it

---

## Technical Concepts Covered

- Default base permissions (666 for files, 777 for directories)
- `umask` as a subtractive permission filter
- File vs directory permission behavior
- Creation-time permission calculation
- No retroactive permission changes from `umask`

---

## Commands Used

```bash
umask
touch file1
mkdir dir1
ls -l
umask 077
touch file2
mkdir dir2
ls -l
```

---

## Procedure

1. Checked system default `umask` value
2. Created a file (`file1`) and directory (`dir1`)
3. Observed resulting permissions using `ls -l`
4. Changed `umask` to `077`
5. Created new file (`file2`) and directory (`dir2`)
6. Verified how new permissions differed from earlier outputs
7. Compared old vs new creation behavior

---

## Results

Default `umask` observed: `0022`

- `file1` → `-rw-r--r--` (644)
- `dir1` → `drwxr-xr-x` (755)

After changing `umask` to `077`:

- `file2` → `-rw-------` (600)
- `dir2` → `drwx------` (700)

Key confirmations:

- Existing files did not change after `umask` update
- Files never get execute permission by default
- Directories always include execute permission for traversal

---

## Evidence

```bash
umask
# 0022

ls -l
# file1 → -rw-r--r--
# dir1  → drwxr-xr-x

umask 077

ls -l
# file2 → -rw-------
# dir2  → drwx------
```

---

## Key Takeaways

- Linux does not "assign" permissions — it calculates them at creation time
- `umask` removes permissions from a base template, it does not add them
- Files and directories follow different base rules (666 vs 777)
- Permission mistakes must be understood at creation context, not fixed afterward blindly

---

## What This Demonstrates

- Linux access control begins before interaction (creation-time logic)
- System behavior is deterministic, not arbitrary
- Permission structure is a predictable formula, not guesswork
- Understanding `umask` = understanding how the system "births" objects

---

## Security / Administration Relevance

- Prevents accidental exposure of sensitive files
- Enforces baseline security posture on multi-user systems
- Critical for server hardening and deployment environments
- Helps troubleshoot permission issues at the root cause (creation layer)

---

## Time Spent

~20–30 minutes

---

## Conclusion

This lab confirmed a core system truth: permissions are not corrected after creation — they are shaped at creation.

Once `umask` is understood, file behavior stops being reactive and becomes predictable.
