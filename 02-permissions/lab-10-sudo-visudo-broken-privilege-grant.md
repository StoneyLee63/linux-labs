# Lab 10 — sudo & visudo: Diagnosing and Repairing a Broken Privilege Grant

---

## Objective

Practiced diagnosing why a sudo grant that "should" work was actually failing, distinguishing a malformed rule from a missing one, repairing it safely with the tool built for that exact purpose, and proving the fix was both functional and correctly scoped — not blanket root access. Also worked through real job-control and terminal friction mid-task, which turned into its own diagnostic exercise separate from the sudoers content itself.

---

## Environment

- Distribution: Rocky Linux 9
- Platform: VM (VirtualBox)
- User Context: root (switching into the standard user session via `su -` for diagnosis and verification)

---

## Scenario

A junior admin set up a sudo rule so a user (`elena`) could restart a specific service (`backup.service`) herself whenever it hung, instead of waiting on IT. She hand-edited the sudoers drop-in directly instead of going through the proper tool. Now when Elena tries to restart it, something's not working. Task: find out what's actually broken, fix it so Elena can restart that one service herself, and prove she still can't do anything beyond that.

---

## Technical Concepts Covered

- sudoers rules as all-or-nothing per line — a syntax error invalidates the entire rule, not just weakens it
- `visudo -c` — syntax validation across the main sudoers file and every file under `/etc/sudoers.d/`
- `visudo -f <file>` — safety-gated editing of one specific drop-in, refuses to save invalid syntax
- The `NOPASSWD:` tag's colon requirement and what happens to the parser when it's missing
- Shell job control (`jobs`, `kill %1`) as a real blocker independent of the sudoers content itself
- `sudo -n` — forcing non-interactive behavior to get a deterministic result when no controlling terminal is available
- `su - user -c '...'` — one-shot verification from the actual account's session, both for the grant and the scope limit

---

## Commands Used

```bash
groupadd -f backup_ops
useradd -m elena
usermod -aG backup_ops elena
# systemd dummy unit + sudoers.d drop-in staged via setup script

su - elena -c 'sudo systemctl restart backup.service'
visudo -c
cat /etc/sudoers.d/elena-backup

visudo -f /etc/sudoers.d/elena-backup
jobs
kill %1
jobs
visudo -f /etc/sudoers.d/elena-backup

cat /etc/sudoers.d/elena-backup
visudo -c

su - elena -c 'sudo systemctl restart backup.service && echo OK'
su - elena -c 'sudo systemctl restart sshd'
su - elena -c 'sudo -n systemctl restart sshd'
```

---

## Procedure

1. Attempted the exact command Elena was told she could run, from her own session, to reproduce the reported failure firsthand.
2. Ran `visudo -c` to check syntax across the entire sudoers configuration, including all drop-ins — confirmed a syntax error in `elena-backup`.
3. Read the broken line directly with `cat` and identified the cause: `NOPASSWD` missing its required trailing colon.
4. Opened the file with `visudo -f` to make the fix under the same validate-before-save protection as the main sudoers file — first attempt was interrupted with Ctrl+Z, which suspended the editor instead of saving.
5. Diagnosed the resulting "file busy" error on the next attempt by checking `jobs`, found the suspended session still holding the file lock, and cleared it with `kill %1`.
6. Reopened the file cleanly, added the missing colon, and saved properly with `:wq`.
7. Re-verified with `visudo -c` that both the main file and the drop-in parsed OK before testing any behavior.
8. Proved the grant from Elena's own session — command executed with no password prompt, confirming `NOPASSWD` was functioning.
9. Attempted an out-of-scope command (`sshd` restart) from Elena's session to prove the grant didn't extend beyond the one intended service. Hit a tty limitation testing this interactively through `su -c`, so re-ran with `sudo -n` to force a deterministic, non-interactive result instead.

---

## Results

- Elena successfully restarts `backup.service` with no password prompt — the grant works exactly as configured.
- Elena's attempt to restart `sshd` is blocked (`sudo: a password is required` under `-n`) — the grant's scope holds, proven from her actual session rather than inferred from the sudoers text alone.
- Two pieces of real friction surfaced and were resolved independently of the core lesson: a suspended `visudo` session holding a file lock, and a terminal limitation on password prompts when testing through `su -c`. Both were diagnosed and worked around rather than treated as blockers to abandon the task.

---

## Evidence

Key output snippets (VM terminal, Rocky Linux 9):

```
[root@rocky9-air ~]# su - elena -c 'sudo systemctl restart backup.service'
/etc/sudoers.d/elena-backup:1:68: syntax error
elena ALL=(root) NOPASSWD /usr/bin/systemctl restart backup.service
                                                                    ^
sudo: a terminal is required to read the password; either use the -S option to read from standard input or configure an askpass helper
sudo: a password is required

[root@rocky9-air ~]# visudo -c
/etc/sudoers.d/elena-backup:1:68: syntax error
elena ALL=(root) NOPASSWD /usr/bin/systemctl restart backup.service
                                                                    ^

[root@rocky9-air ~]# cat /etc/sudoers.d/elena-backup
elena ALL=(root) NOPASSWD /usr/bin/systemctl restart backup.service

[root@rocky9-air ~]# visudo -f /etc/sudoers.d/elena-backup
[1]+  Stopped                 visudo -f /etc/sudoers.d/elena-backup

[root@rocky9-air ~]# visudo -f /etc/sudoers.d/elena-backup
visudo: /etc/sudoers.d/elena-backup busy, try again later

[root@rocky9-air ~]# kill %1
visudo exiting due to signal: Terminated
[1]+  Exit 15                 visudo -f /etc/sudoers.d/elena-backup

[root@rocky9-air ~]# jobs
[root@rocky9-air ~]# cat /etc/sudoers.d/elena-backup
elena ALL=(root) NOPASSWD:/usr/bin/systemctl restart backup.service

[root@rocky9-air ~]# visudo -c
/etc/sudoers: parsed OK
/etc/sudoers.d/elena-backup: parsed OK

[root@rocky9-air ~]# su - elena -c 'sudo systemctl restart backup.service && echo OK'
OK

[root@rocky9-air ~]# su - elena -c 'sudo -n systemctl restart sshd'
sudo: a password is required
```

---

## Key Takeaways

- A syntax error in a sudoers drop-in doesn't degrade a rule, it invalidates it completely — a missing colon after `NOPASSWD` is enough to break the whole line.
- `visudo -c` checks every file under `/etc/sudoers.d/`, not just the file you think you edited — the correct first diagnostic step after any sudo denial.
- `visudo -f` is a genuine safety gate, not a stylistic choice — it will not let a broken edit save, which is exactly the protection a plain text editor doesn't offer.
- Suspending an editor (Ctrl+Z) is not the same as saving or exiting — it leaves the process alive and can hold a file lock that blocks the next legitimate attempt.
- A privilege grant isn't proven until both the intended command succeeds AND an out-of-scope command is confirmed still blocked — and when the test environment itself has limitations (like no controlling terminal), the right move is to adjust the test (`sudo -n`) rather than accept an ambiguous result.

---

## What This Demonstrates

This lab proves sudoers parsing has no partial-credit mode — a single missing character invalidates an entire rule, and both `sudo` itself and `visudo -c` independently confirm this the same way. It also demonstrates that real system administration friction rarely stays contained to the one system being fixed — a shell job-control mistake (Ctrl+Z) produced a second, unrelated blocker (a locked file) that had to be diagnosed on its own terms before the original fix could even be attempted again.

---

## Security / Administration Relevance

This is the exact pattern behind real "can you set up X so I don't have to keep asking IT" requests: a scoped privilege grant needs both a functional proof and a scope-limit proof before it's trusted, and hand-edited sudoers files are exactly how those grants quietly break or over-grant in production. The job-lock and tty friction encountered here are the kind of real-world snags that don't show up in a clean tutorial — recognizing "file busy" as a job-control problem rather than sudoers corruption, and reaching for `sudo -n` when a tty isn't available, are both genuine operator instincts, not just command memorization.

---

## Time Spent

30 minutes

---

## Conclusion

Validated the full sudo/visudo diagnostic and repair loop — syntax diagnosis, safety-gated repair, functional proof, and scope-limit proof — end to end on a live Rocky Linux 9 system, including recovering cleanly from two pieces of real friction that weren't part of the original script. This builds directly toward RHCSA-level privilege escalation tasks, and reinforces that a fix isn't complete until it's proven correct, proven scoped, and proven to actually be the result of the command intended — not an artifact of the environment it was tested in.
