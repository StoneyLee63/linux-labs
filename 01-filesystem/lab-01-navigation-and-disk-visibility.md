# Lab 01 – Navigation & Disk Visibility

## Objective
Practice navigating the filesystem and inspecting disk usage.

---

## Commands Practiced

pwd
ls -lah
cd
du -sh *
df -h

---

## What I Did

- Verified current working directory with `pwd`.
- Listed all files (including hidden) using `ls -lah`.
- Navigated between directories using `cd`.
- Measured directory sizes with `du -sh *`.
- Checked mounted disk usage with `df -h`.

---

## Observations

- `du` shows actual file usage inside directories.
- `df` shows total mounted filesystem usage.
- Hidden files (.) are critical in user home directories.

---

## Operator Notes

Filesystem visibility is foundational.
Before securing or troubleshooting anything, I must know:
- Where I am
- What exists
- How much space is available
