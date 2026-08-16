# Lab 06 — sudo and visudo: Controlled Privilege Escalation

---

## Objective

- Built command-line fluency granting scoped, command-specific sudo privileges to individual users via `/etc/sudoers.d/` drop-in files
- Practiced `NOPASSWD` configuration for a non-interactive service account, scoped to a narrow command pattern
- Configured and verified group-based sudo grants (`%groupname`) and confirmed rule stacking against an individual user
- Deliberately triggered and observed visudo's syntax-checking safety mechanism
- Discovered a real security gap: `visudo -f` does not auto-enforce required file permissions on newly created drop-in files
- Directly applicable to RHCSA exam tasks requiring precise, least-privilege sudo configuration

---

## Environment

- **Distribution:** Ubuntu (WSL2)
- **Platform:** WSL2 on Windows, host DESKTOP-N8MS0U8
- **User Context:** Standard user (ltksol) with sudo privileges
- **Identities used:** `wateruser01`, `svc_water`, `watercrew` (created in Lab 10)

---

## Scenario

Three identities need exactly the privilege escalation they require and nothing more: an operator account that can restart and check one specific service, a service account that needs non-interactive status checks without a password prompt, and a team group that needs full administrative access. This lab built all three using `visudo`-protected drop-in files, then deliberately broke a sudoers file to confirm the safety mechanism actually catches malformed syntax — which led directly to discovering a second, less obvious gap during the closing verification pass.

---

## Technical Concepts Covered

- sudoers rule anatomy: who, host scope, target identity, and allowed command(s)
- `visudo -f <path>` for syntax-checked, lock-protected editing of any sudoers file, including new `/etc/sudoers.d/` drop-ins
- `%groupname` syntax for group-based sudo grants
- `NOPASSWD:` tagging for scoped, non-interactive command access
- Rule stacking — individual and group-level sudo grants apply simultaneously, not exclusively
- `visudo`'s syntax-error recovery prompt and its "What now?" e/x/Q options
- `visudo -c` as a full-configuration audit across the main file and every `/etc/sudoers.d/` entry, checking both syntax and file permissions
- The `0440` permission requirement for sudoers files as a tampering-prevention control

---

## Commands Used

```bash
sudo -l -U wateruser01

sudo visudo -f /etc/sudoers.d/wateruser01-nginx
# wateruser01 ALL=(ALL) /usr/bin/systemctl restart nginx, /usr/bin/systemctl status nginx

sudo -l -U wateruser01

sudo visudo -f /etc/sudoers.d/svc_water-status
# svc_water ALL=(ALL) NOPASSWD: /usr/bin/systemctl status *

sudo -l -U svc_water

sudo visudo -f /etc/sudoers.d/watercrew-fullaccess
# %watercrew ALL=(ALL) ALL

sudo -l -U wateruser01

sudo visudo -f /etc/sudoers.d/broken-test
# wateruser01 ALL=(ALL ALL    (deliberately malformed)

sudo rm /etc/sudoers.d/broken-test
sudo visudo -c

sudo -l -U wateruser01
sudo -l -U svc_water
```

---

## Procedure

1. Confirmed baseline: `wateruser01` had no sudo access at all before any rule existed.
2. Created a scoped individual rule via `visudo -f /etc/sudoers.d/wateruser01-nginx`, granting exactly two systemctl actions on the nginx service — verified with `sudo -l -U wateruser01`, which showed only those two commands.
3. Created a `NOPASSWD` rule for `svc_water`, scoped to status checks on any unit — verified the rule appeared correctly tagged, distinct from password-required entries.
4. Created a group-wide rule (`%watercrew ALL=(ALL) ALL`) — re-checked `wateruser01` (a `watercrew` member) and confirmed both the broad group grant and the narrow individual grant were active simultaneously.
5. Deliberately wrote a malformed rule (`wateruser01 ALL=(ALL ALL`, missing a closing parenthesis) into a test drop-in file via `visudo -f` — visudo caught it immediately, reporting the exact line/column of the syntax error and entering its recovery prompt.
6. Exited the recovery prompt via Ctrl-C (interrupt signal) rather than selecting one of the offered e/x/Q options, then manually removed the broken test file.
7. Ran `sudo visudo -c` as a closing audit across the entire sudoers configuration — this is where the real finding surfaced.

---

## Results

- All three sudo grants (individual command-scoped, NOPASSWD service account, group-wide) applied and verified exactly as configured.
- Rule stacking confirmed directly: `sudo -l -U wateruser01` showed both the inherited group rule and the individual rule active at the same time — sudo does not deduplicate or override between sources, it accumulates.
- visudo's syntax protection worked exactly as designed — the malformed test rule was caught before it could ever be committed to a live sudoers file.
- **The real finding:** `sudo visudo -c` flagged all three legitimately created drop-in files with "bad permissions, should be mode 0440." Creating a new file through `visudo -f` validates syntax and writes atomically, but does not automatically lock the file's permissions to the secure `0440` standard — they were left at the default umask-driven mode instead, which is broader than sudo's security model expects.
- The rules remained functionally active despite the looser permissions in this run, but the gap is real and exam-relevant: a sudoers file readable beyond root and the sudo group defeats part of the purpose of restricting *who can even view* privilege configuration.

---

## Evidence

**Baseline — no access**
```
User wateruser01 is not allowed to run sudo on DESKTOP-N8MS0U8.
```

**Individual scoped rule confirmed**
```
User wateruser01 may run the following commands on [host]:
    (ALL) /usr/bin/systemctl restart nginx, /usr/bin/systemctl status nginx
```

**NOPASSWD service account rule confirmed**
```
User svc_water may run the following commands on [host]:
    (ALL) NOPASSWD: /usr/bin/systemctl status *
```

**Rule stacking confirmed — group + individual both active**
```
User wateruser01 may run the following commands on [host]:
    (ALL) ALL
    (ALL) /usr/bin/systemctl restart nginx, /usr/bin/systemctl status nginx
```

**Syntax error caught by visudo**
```
/etc/sudoers.d/broken-test:1:22: syntax error
wateruser01 ALL=(ALL ALL
                     ^~~
What now? ^Cvisudo exiting due to signal: Interrupt
```

**Permission gap discovered during closing audit**
```
$ sudo visudo -c
/etc/sudoers: parsed OK
/etc/sudoers.d/README: parsed OK
/etc/sudoers.d/svc_water-status: bad permissions, should be mode 0440
/etc/sudoers.d/watercrew-fullaccess: bad permissions, should be mode 0440
/etc/sudoers.d/wateruser01-nginx: bad permissions, should be mode 0440
```

---

## Key Takeaways

- sudo privilege is additive, not overriding — auditing a user's real access requires checking every applicable rule source (individual and every group they belong to), not just the most recently written one.
- `visudo`'s built-in protection is syntax-only. It does not guarantee correct file permissions on newly created drop-in files — that's a separate check (`visudo -c`) that has to be run deliberately as a closing step, not assumed to be covered automatically.
- A correctly written sudo rule sitting in an incorrectly permissioned file is still a real security gap, even if it doesn't cause a visible functional failure during normal use — the danger is invisible until something or someone goes looking for it.
- When visudo's recovery prompt appears after a syntax error, use the offered options rather than force-exiting — the safe-exit path is part of the tool's design, and bypassing it removes the guarantee of a clean abandon.
- A closing verification step (`visudo -c`, `getfacl`, `stat -c`, `id` — whatever fits the task) is what catches the gap between "it works" and "it's actually secured correctly." This rep is the clearest proof yet that the two are not the same check.

---

## What This Demonstrates

- **Additive privilege model proven, not assumed:** the group + individual rule stacking on `wateruser01` was directly observed in `sudo -l` output, not inferred from documentation.
- **Syntax safety net confirmed under deliberate stress:** triggering a real parse error and watching visudo catch it before commit is stronger evidence than reading about the feature.
- **A real, previously invisible gap surfaced through disciplined verification:** the permission finding only appeared because the closing audit step was run as habit — proof that consistent verification discipline finds issues that "it's working" never will.

---

## Security / Administration Relevance

- **Least-privilege sudo design:** command-scoped rules (like the nginx-only grant) are the correct pattern for any account that needs elevated access for one specific operational task — broad `ALL` grants should be reserved deliberately, not defaulted to.
- **NOPASSWD scope discipline:** appropriate for tightly bound, non-interactive automation accounts — the danger scales directly with how broad the attached command pattern is, so `NOPASSWD: ALL` is a very different risk profile than `NOPASSWD: /usr/bin/systemctl status *`.
- **Sudoers file permission hygiene:** every new drop-in file should be checked with `visudo -c` immediately after creation — a file left at default permissions can be readable by every user on the system, exposing exactly what privilege escalation paths exist to anyone looking.
- **Closing-audit discipline as a security practice:** this finding is a direct argument for why "did it work" is an insufficient security check on its own — a deliberate audit pass after every privilege change is what actually closes the loop.

---

## Time Spent

Approximately 35 minutes

---

## Conclusion

Built scoped, additive sudo privilege across three real identities — command-restricted, NOPASSWD-tagged, and group-based — and confirmed visudo's syntax protection works exactly as designed under a deliberate stress test. The most valuable result wasn't any of the planned steps, though — it was the permission gap `visudo -c` caught during the closing audit, proving that a sudo rule can be syntactically perfect and still sit in a file that's more exposed than it should be. Fixing that gap (`chmod 0440` on each drop-in, followed by a clean `visudo -c` re-check) is the next action before this rep is fully closed. Water's access-control arc now spans permissions, identity, named exceptions, and controlled privilege escalation — four real, connected layers, each verified against live system behavior rather than assumed from documentation.

---

## Open Action Item

Permission fix not yet applied/confirmed in this session: `sudo chmod 0440` on all three drop-in files, followed by a re-run of `sudo visudo -c` to confirm a fully clean configuration. This will be appended to this lab once the real output is captured.
