# Lab 08 — Account Provisioning: Diagnosing and Repairing a Four-Gate Login Failure

---

## Objective

Practiced diagnosing and fully provisioning a Linux user account that "existed" in the passwd database but could not actually log in — walking a fixed four-gate chain (shell, home directory, group membership, password/shadow) to identify exactly which gate was closed and closing it correctly, then proving the fix from the account's own session. This is the core diagnostic loop behind any real "new hire can't log in" ticket.

---

## Environment

- Distribution: Rocky Linux 9
- Platform: VM (VirtualBox)
- User Context: root (switching into the standard user session via `su -` for diagnosis and final verification)

---

## Scenario

A new user account (`diego`) was created ahead of onboarding but left in a partial state — the record exists in `/etc/passwd`, but the account cannot log in, has no home directory, isn't part of the `ops` group it needs, and has no password set. The task: confirm the actual cause of the login failure rather than assuming it, close each gap with the correct tool (not a shortcut like bare `mkdir`), and verify the account works end to end from its own session — the same pattern behind any real account-provisioning or access-restoration ticket.

---

## Technical Concepts Covered

- The passwd record as a claim vs. the filesystem as the actual truth
- `useradd -M` — deliberately skips home directory creation
- `usermod -s` — the login shell as the first and hardest login gate
- `mkhomedir_helper` — the system-correct way to create a home directory (skel population, ownership), same tool PAM calls automatically
- `usermod -aG` vs. bare `usermod -G` — append vs. full replace of the supplementary group list
- `passwd` / shadow lock state — password gate checked completely independently of shell, home, and group
- `su - user -c '...'` — one-shot verification from the actual account's session, not assumed from root

---

## Commands Used

```bash
groupadd -f ops
useradd -M -s /sbin/nologin diego

su - diego
getent passwd diego
ls -ld /home/diego

usermod -s /bin/bash diego
getent passwd diego

mkhomedir_helper diego
ls -ld /home/diego

usermod -aG ops diego
id diego

passwd diego
ls -ld /home/diego
id diego
getent passwd diego

su - diego -c 'pwd && id && ls -la ~'
```

---

## Procedure

1. Attempted to log in as diego (`su - diego`) to reproduce the reported failure firsthand, before reading any config.
2. Read diego's own passwd record with `getent passwd diego` to see what the account claimed about itself.
3. Checked whether the filesystem backed up that claim with `ls -ld /home/diego` — confirmed the home directory did not exist.
4. Closed gate 1 by changing the shell field to a real interactive shell (`usermod -s /bin/bash diego`), then re-read the record to confirm the change took.
5. Closed gate 2 the system-correct way with `mkhomedir_helper diego`, which creates the home directory at the path already recorded in passwd, populates it from `/etc/skel`, and sets correct ownership — then verified with `ls -ld`.
6. Closed gate 3 by appending diego to the `ops` group with `usermod -aG ops diego` (append, not replace), then verified his full group list with `id`.
7. Closed gate 4 by setting a real password with `passwd diego`, replacing the locked shadow placeholder.
8. Re-verified every layer as root (`ls -ld`, `id`, `getent passwd`) before trusting the fix.
9. Proved the complete fix from diego's own session in one pass: `su - diego -c 'pwd && id && ls -la ~'` — working directory, full identity/groups, and home directory contents all confirmed from his actual login, not from root's assumption.

---

## Results

- diego's own session confirms all four gates open at once: correct home directory, correct groups (`diego`, `ops`), and a fully populated home directory from `/etc/skel`.
- Every intermediate check (`getent passwd`, `ls -ld`, `id`) matched the expected state transition exactly — no gate was skipped, no gate was assumed closed-then-open without direct verification.
- One deviation from prediction: the home directory landed `drwxr-xr-x` rather than the expected `drwx------`, indicating this VM's `mkhomedir_helper` is working under a looser default umask than assumed — noted for follow-up, not a failure of the procedure.

---

## Evidence

Key output snippets (VM terminal, Rocky Linux 9):

```
[root@rocky9-air ~]# su - diego
su: warning: cannot change directory to /home/diego: No such file or directory
This account is currently not available.

[root@rocky9-air ~]# getent passwd diego
diego:x:1003:1005::/home/diego:/sbin/nologin

[root@rocky9-air ~]# ls -ld /home/diego
ls: cannot access '/home/diego': No such file or directory

[root@rocky9-air ~]# usermod -s /bin/bash diego
[root@rocky9-air ~]# getent passwd diego
diego:x:1003:1005::/home/diego:/bin/bash

[root@rocky9-air ~]# mkhomedir_helper diego
[root@rocky9-air ~]# ls -ld /home/diego
drwxr-xr-x. 3 diego diego 78 Jul 13 16:33 /home/diego

[root@rocky9-air ~]# usermod -aG ops diego
[root@rocky9-air ~]# id diego
uid=1003(diego) gid=1005(diego) groups=1005(diego),1004(ops)

[root@rocky9-air ~]# passwd diego
Changing password for user diego.
New password:
Retype new password:
passwd: all authentication tokens updated successfully.

[root@rocky9-air ~]# su - diego -c 'pwd && id && ls -la ~'
/home/diego
uid=1003(diego) gid=1005(diego) groups=1005(diego),1004(ops) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
total 12
drwxr-xr-x. 4 diego diego  92 Jul 13 16:51 .
drwxr-xr-x. 6 root root    62 Jul 13 16:33 ..
-rw-r--r--. 1 diego diego  18 Jul 13 16:33 .bash_logout
-rw-r--r--. 1 diego diego 141 Jul 13 16:33 .bash_profile
-rw-r--r--. 1 diego diego 492 Jul 13 16:33 .bashrc
drwx------. 2 diego diego   6 Jul 13 16:51 .cache
drwxr-xr-x. 4 diego diego  39 Jul 13 16:33 .mozilla
```

---

## Key Takeaways

- A user account is not one fact but four independently-checked gates — shell, home directory, group membership, password/shadow — and a failed login only proves which gate stopped it, not the state of the others.
- The passwd record is a claim, never a guarantee; filesystem state has to be checked separately with `ls`, never assumed from `getent`.
- `useradd -M` and `mkhomedir_helper` exist as a deliberate pair — provisioning tools that skip filesystem work, and repair tools that do it the system-correct way (skel population, correct ownership) instead of a bare `mkdir`.
- `usermod -aG` is the safe default for adding a group; bare `usermod -G` silently replaces the entire supplementary group list and is only correct when that full replacement is the actual intent.

---

## What This Demonstrates

This lab proves Linux account state is evaluated as a set of independent, ordered gates rather than a single pass/fail flag — the kernel and PAM stack check shell validity, home directory presence, group membership, and password state as genuinely separate facts, and none of them imply the others. It also demonstrates that provisioning tools and repair tools are intentionally decoupled: `useradd -M` leaves a gap on purpose, and `mkhomedir_helper` exists specifically to close that gap the same way the system itself would on first login via `pam_mkhomedir`.

---

## Security / Administration Relevance

This is the exact diagnostic pattern behind real onboarding and access-restoration tickets: an account "exists" but a new hire can't log in, and the fix requires walking a fixed sequence of checks rather than guessing at the first error message. The `usermod -aG` vs. `-G` distinction is a genuine production hazard — both commands succeed silently, and using the wrong one on an account with existing group memberships causes a quiet access regression that won't surface until someone notices they lost permissions they used to have. Verifying the final state from the account's own session, rather than trusting root's assumption, is the habit that catches gaps before the user does.

---

## Time Spent

30 minutes

---

## Conclusion

Validated the full four-gate account provisioning and repair loop — shell, home directory, group membership, and password — end to end on a live Rocky Linux 9 system, with one minor deviation from predicted permission mode noted rather than treated as failure. This builds directly toward RHCSA-level user management tasks, which are graded on final verified account state, and reinforces the operator habit of treating a passwd record as a claim to be tested, never a fact to be trusted.
