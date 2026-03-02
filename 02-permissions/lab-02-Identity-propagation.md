# Lab 02 — Linux User Identity & Process Ownership

## Objective
Examine how Linux determines user identity and how that identity propagates into processes, permissions, and session behavior.

---

## Core Question
How does Linux determine who a user is, and how does that identity influence process ownership and system behavior?

---

## Technical Concepts Covered

- User Identity (UID)
- Primary and Supplementary Group Membership (GID)
- Process Ownership
- Session Initialization (PAM → user systemd → shell → child processes)

---

## Commands Executed

whoami
id
groups
ps -u $USER
ps -u ltksoul1205


---

## Expected Outcome

- View all processes associated with the active user identity  
- Confirm identity persistence across multiple terminal sessions  
- Verify that `$USER` resolves to the currently authenticated account  

---

## Observations

- `$USER` resolves to `ltksoul1205`, confirming environment variable alignment with login identity.
- The `id` command exposes the numeric UID and group memberships used for permission enforcement.
- Multiple `bash` processes appear due to separate terminal sessions (`pts/0`, `pts/1`).
- A user-level `systemd` instance runs within the session, maintaining user-scoped services.
- `(sd-pam)` persists as part of the authentication and session lifecycle.
- Each command execution spawns a process owned by the invoking UID.

---

## What This Demonstrates

- Linux enforces authorization based on numeric identity (UID), not username strings.
- Group membership directly impacts file access and permission checks.
- Process ownership is inherited from the parent shell.
- Identity persists across session layers through PAM and user-level systemd services.

---

## Security Relevance

- Privilege escalation requires a change in effective UID.
- Excessive or misconfigured group membership can expand access beyond intended scope.
- Understanding identity inheritance is essential for auditing, incident response, and forensic analysis.

---

## Time Spent
55 minutes including execution and documentation

---

## Conclusion

Linux behavior is identity-driven.  
Process control, file access, and authorization decisions are determined by UID and group context.
