# Lab 04 — File Permissions: chmod, chown, and Special Bits

---

## Objective

- Built command-line fluency converting between octal and symbolic chmod notation and predicting their effect on the rwx triplet
- Practiced ownership changes (chown) as an operation fully independent of permission changes (chmod)
- Verified real-world SUID behavior on a production binary (`/usr/bin/passwd`) without needing to fabricate a test case
- Created and configured a shared directory using SGID and the sticky bit together — the exact pattern used for multi-user team folders and `/tmp`-style shared spaces
- Directly applicable to RHCSA exam tasks involving permission audits, ownership assignment, and shared-directory configuration

---

## Environment

- **Distribution:** Ubuntu (WSL2)
- **Platform:** WSL2 on Windows, host DESKTOP-N8MS0U8
- **User Context:** Standard user (ltksol) with sudo privileges

---

## Scenario

A deployment script needs locked-down permissions before it goes anywhere near production, and a team needs a shared directory where multiple users can drop files without overwriting or deleting each other's work. This lab built both from scratch: explicit permission control on a single file using octal and symbolic chmod, ownership reassignment, and a shared directory hardened with SGID (consistent group inheritance) and the sticky bit (delete protection). It also used the live `/usr/bin/passwd` binary to confirm SUID behavior on a real system file rather than a synthetic example.

---

## Technical Concepts Covered

- The rwx permission triplet and its octal equivalent (r=4, w=2, x=1)
- Octal chmod (`chmod 750`) vs. symbolic chmod (`chmod u-x,g+w`) — same mechanism, different precision
- Directory execute bit semantics — traversal (`cd`), not "run"
- Ownership (`chown user:group`) as metadata independent from permission bits
- SUID — execution under the file owner's identity regardless of caller
- SGID on a directory — automatic group inheritance for new files
- Sticky bit — delete/rename restricted to file owner even when others have directory write access
- Capital vs. lowercase special-bit notation (`s`/`S`, `t`/`T`) as a diagnostic signal for whether the underlying execute bit is present
- Verification via `stat -c` formatted output across multiple targets in one command

---

## Commands Used

```bash
mkdir ~/water_lab && cd ~/water_lab
ls
echo "deploy script placeholder" > project.sh
ls -l project.sh

chmod 750 project.sh
ls -l project.sh

chmod u-x,g+w project.sh
ls -l project.sh

sudo chown root:root project.sh
ls -l project.sh

ls -l /usr/bin/passwd

sudo mkdir /shared_water
sudo chmod 2770 /shared_water
ls -ld /shared_water

sudo chmod +t /shared_water
ls -ld /shared_water

stat -c '%A %U:%G %n' project.sh /shared_water /usr/bin/passwd
```

---

## Procedure

1. Created `~/water_lab` and a test file `project.sh` — default creation produced `-rw-r--r--`, confirming the umask default (owner read/write, group/others read-only, no execute anywhere).
2. Applied `chmod 750` — result `-rwxr-x---`. Owner gets full rwx, group gets read/execute, others get nothing.
3. Applied symbolic `chmod u-x,g+w` on top of that state — result `-rw-rwx---`. This diverged from the pre-session prediction (`rw-rw----`); the discrepancy was a math error on my part, not a system error — see Results.
4. Reassigned ownership with `sudo chown root:root` — owner/group columns flipped to `root root`, permission bits stayed completely untouched, confirming ownership and permissions are independently tracked metadata.
5. Inspected the live `/usr/bin/passwd` binary — confirmed SUID already set (`-rwsr-xr-x`, owned by root, 64152 bytes) without needing to set it manually on a system file.
6. Created `/shared_water`, applied `chmod 2770` — result `drwxrws---`, confirming SGID plus full rwx for owner/group, none for others.
7. Added the sticky bit with `chmod +t` — result `drwxrws--T`, capital `T` because "others" has no execute bit to pair the lowercase notation with.
8. Ran a single `stat -c` command against all three targets (`project.sh`, `/shared_water`, `/usr/bin/passwd`) to verify every change from steps 2–7 held in one consolidated check.

---

## Results

- All chmod, chown, and special-bit operations applied and verified successfully across both a regular file and a directory.
- **Symbolic chmod correction:** predicted `chmod u-x,g+w` on a `750` base would produce `rw-rw----`. Actual result was `rw-rwx---`. Root cause: group's starting state was `r-x` (from step 2), and `+w` adds onto the existing bits rather than resetting them — `r-x` + `w` = `rwx`, not `rw-`. The system behaved correctly; the prediction assumed group started from a blank `r--` instead of its actual prior state.
- `chown` confirmed fully decoupled from `chmod` — ownership moved, permission bits were untouched in the same `ls -l` line.
- `/usr/bin/passwd` confirmed SUID is a live, default-configured security mechanism on this system, not a theoretical concept — this is the exact binary that lets unprivileged users change their own password despite `/etc/shadow` being root-only.
- SGID + sticky stacked correctly on `/shared_water`, with the capital `T` confirming the no-execute-for-others state predicted by the permission model.
- `stat -c` one-liner cleanly cross-verified all three targets, closing the loop on every change made in the session.

---

## Evidence

**Permission progression on project.sh**
```
-rw-r--r-- 1 ltksol ltksol 26 Jun 16 12:20 project.sh   (default)
-rwxr-x--- 1 ltksol ltksol 26 Jun 16 12:20 project.sh   (chmod 750)
-rw-rwx--- 1 ltksol ltksol 26 Jun 16 12:20 project.sh   (chmod u-x,g+w)
-rw-rwx--- 1 root   root   26 Jun 16 12:20 project.sh   (chown root:root)
```

**Live SUID on a real system binary**
```
-rwsr-xr-x 1 root root 64152 May 30  2024 /usr/bin/passwd
```

**SGID + sticky bit progression on /shared_water**
```
drwxrws--- 2 root root 4096 Jun 16 12:40 /shared_water   (chmod 2770)
drwxrws--T 2 root root 4096 Jun 16 12:40 /shared_water   (chmod +t)
```

**Final consolidated verification**
```
$ stat -c '%A %U:%G %n' project.sh /shared_water /usr/bin/passwd
-rw-rwx--- root:root project.sh
drwxrws--T root:root /shared_water
-rwsr-xr-x root:root /usr/bin/passwd
```

---

## Key Takeaways

- Symbolic chmod operations (`+`/`-`) modify relative to current state, not an assumed blank slate — predicting the result requires knowing exactly what the bits already are, not just what the target description says.
- Ownership and permissions are two completely independent audit trails. A security review (or an RHCSA task) checking "is this file secured correctly" must verify both — neither implies the other.
- Special bits don't replace the base rwx — they stack on top of it. The capital/lowercase distinction in the display (`s` vs `S`, `t` vs `T`) is the system telling you directly whether the dependent execute bit is present.
- Real system binaries (like `/usr/bin/passwd`) are the best lab material for SUID — no need to fabricate a scenario when the actual security mechanism is sitting on every Linux system already.
- A single `stat -c` call across multiple targets is faster and more reliable than running `ls -l` repeatedly — useful as a closing verification habit on any multi-step permission task.

---

## What This Demonstrates

- **Bitmask arithmetic confirmed, not assumed:** The symbolic chmod discrepancy proved that permission changes are pure bitwise operations on existing state, not text substitutions based on a target description.
- **Independent metadata layers proven:** `chown` altered owner/group without touching the permission string in the same operation — direct confirmation that Linux tracks these as separate inode fields.
- **SUID mechanism verified on production code:** Confirmed the file-owner-execution model using the actual `passwd` binary, the same mechanism behind every privileged self-service operation on a Linux system.
- **Special bit stacking and display logic understood:** SGID and sticky bit applied together, with the capital `T` correctly predicted and explained as a direct consequence of the underlying execute bit's absence.

---

## Security / Administration Relevance

- **Least-privilege auditing:** Knowing the exact octal/symbolic mechanics means being able to lock a file down to the minimum required access without guesswork — `750` vs `770` vs `700` is a security decision, not a default.
- **SUID awareness:** Every SUID binary on a system is a potential privilege escalation vector if misconfigured. Knowing how `/usr/bin/passwd` legitimately uses it is the baseline for recognizing when a *non-standard* SUID binary is a red flag during a security audit.
- **Shared directory hardening:** SGID + sticky bit is the exact configuration pattern behind `/tmp` and every properly secured team directory — required knowledge for setting up collaborative environments without giving up access control.
- **Ownership vs. permission separation:** Real incidents often happen when an admin changes one and assumes the other followed — this rep reinforced verifying both independently every time.

---

## Time Spent

Approximately 35 minutes

---

## Conclusion

Ran the full permission/ownership/special-bit chain against real lab files and a live production binary — every operation executed and independently verified. The one real friction point, a symbolic chmod prediction error, turned into the most valuable part of the session: a direct confirmation that chmod's symbolic mode operates relative to existing state, not a fresh slate, and that verification after every change is what catches reasoning errors before they become exam point losses. The Water domain's permission layer is now functional groundwork for the identity management (Users and Groups) rep that builds on top of it next.
