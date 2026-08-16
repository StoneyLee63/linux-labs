# Lab 09 — ACLs: Granting Scoped, Named Access to a Single File

---

## Objective

Practiced granting one named contractor precise, auditable access to exactly one file inside a directory he has no other business in — without adding him to the owning group and without loosening any broader permission bit — then proved the scope holds by testing both the access that should succeed and the access that should still be denied. This is the core diagnostic and access-control loop behind any real "external party needs one specific file" request.

---

## Environment

- Distribution: Rocky Linux 9
- Platform: VM (VirtualBox)
- User Context: root (switching into the standard user session via `su -` for diagnosis and verification)

---

## Scenario

A contractor (`malik`) needs to read one specific compliance file (`contract.txt`) inside the legal team's shared directory (`/srv/legal`) for a one-time review. Policy requires malik not join the `legal` group and not gain visibility into anything else in that directory — access scoped to exactly one file, nothing broader, with the scope proven rather than assumed.

---

## Technical Concepts Covered

- Standard permission bits (owner/group/other) as category-only, with no lever for a single named identity
- `getfacl` — reading a file's full access control list, including default mirror entries
- `setfacl -m u:<user>:<perms>` — adding a named user entry scoped to one identity
- The ACL mask as a ceiling over every named entry's effective permission
- Directory traversal (`x`) vs. file read (`r`) as two independent access gates
- `su - user -c '...'` — one-shot verification from the actual account's session, including verifying a denial holds, not just a grant

---

## Commands Used

```bash
groupadd -f legal
useradd -m malik
mkdir -p /srv/legal
echo "Q2 contract terms" > /srv/legal/contract.txt
chown root:legal /srv/legal /srv/legal/contract.txt
chmod 770 /srv/legal
chmod 660 /srv/legal/contract.txt

su - malik -c 'cat /srv/legal/contract.txt'
getfacl /srv/legal/contract.txt

setfacl -m u:malik:r-- /srv/legal/contract.txt
getfacl /srv/legal/contract.txt

su - malik -c 'cat /srv/legal/contract.txt'

setfacl -m u:malik:--x /srv/legal
getfacl /srv/legal/contract.txt

su - malik -c 'cat /srv/legal/contract.txt'
su - malik -c 'ls /srv/legal'
```

---

## Procedure

1. Confirmed the baseline denial from malik's own session (`su - malik -c 'cat ...'`) before touching any configuration — the file's standard bits (`root:legal`, mode `660`) left no path in for a non-legal user.
2. Read the file's existing ACL with `getfacl` to establish the baseline — confirmed every file carries an ACL by default, even as a plain mirror of the classic bits.
3. Added a named user entry with `setfacl -m u:malik:r--`, scoping read access to malik specifically, independent of the `legal` group.
4. Re-read the ACL to confirm the new entry and observed the `mask::` line, which only becomes visible once a named entry exists.
5. Re-tested from malik's session and confirmed the file-level grant alone was not enough — directory traversal is a separate, independent gate.
6. Granted execute-only on the directory (`setfacl -m u:malik:--x /srv/legal`) — enough to traverse into a known filename without read permission to list the directory's other contents.
7. Verified the grant succeeded from malik's own session: file contents printed.
8. Verified the scope limit held by attempting `ls /srv/legal` from malik's session — confirmed denied, proving the access was scoped to the one file and did not extend to directory listing.

---

## Results

- Malik reads `contract.txt` successfully from his own session — the targeted grant proven, not assumed.
- Malik is denied `ls /srv/legal` from his own session — the least-privilege scope holds, proven from the same account the grant was made for, not inferred from the ACL entries alone.
- Every intermediate `getfacl` check matched the expected ACL state exactly through the file-level grant and the mask.
- One process note: the first attempt to prove the scope limit was run as `ls/srv/legal` (missing the space), which bash parsed as a nonexistent command and returned "No such file or directory" rather than the intended "Permission denied." Caught on review and rerun correctly before accepting the result as proof — see Evidence.

---

## Evidence

Key output snippets (VM terminal, Rocky Linux 9):

```
[root@rocky9-air ~]# su - malik -c 'cat /srv/legal/contract.txt'
cat: /srv/legal/contract.txt: Permission denied

[root@rocky9-air ~]# getfacl /srv/legal/contract.txt
# file: srv/legal/contract.txt
# owner: root
# group: legal
user::rw-
group::rw-
other::---

[root@rocky9-air ~]# setfacl -m u:malik:r-- /srv/legal/contract.txt
[root@rocky9-air ~]# getfacl /srv/legal/contract.txt
# file: srv/legal/contract.txt
# owner: root
# group: legal
user::rw-
user:malik:r--
group::rw-
mask::rw-
other::---

[root@rocky9-air ~]# su - malik -c 'cat /srv/legal/contract.txt'
cat: /srv/legal/contract.txt: Permission denied

[root@rocky9-air ~]# setfacl -m u:malik:--x /srv/legal
[root@rocky9-air ~]# su - malik -c 'cat /srv/legal/contract.txt'
Q2 contract terms

[root@rocky9-air ~]# su - malik -c 'ls/srv/legal'
-bash: line 1: ls/srv/legal: No such file or directory

[root@rocky9-air ~]# su - malik -c 'ls /srv/legal'
ls: cannot open directory '/srv/legal': Permission denied
```

---

## Key Takeaways

- Standard permission bits can only speak in categories (owner/group/other) — ACLs exist specifically to grant to one named identity without disturbing group membership or wider permissions.
- Every file carries an ACL by default; `getfacl` on an untouched file still returns full output, just mirroring the classic bits.
- Directory traversal and file read are independently checked gates — a correct file-level grant is worthless without execute on every directory in the path.
- The ACL mask only becomes visible once a named entry exists, and it acts strictly as a ceiling — it can cap a grant, never expand one.
- A scope claim isn't proven until both the intended access is confirmed working and the unintended access is confirmed still blocked, tested from the actual account's own session.
- A result that looks like the expected outcome isn't proof by itself — the command that produced it has to be the one actually intended, not a typo that happens to fail in a similar-looking way.

---

## What This Demonstrates

This lab proves Linux's access control model layers cleanly: classic owner/group/other bits handle broad categories, and ACLs extend that model with named, individually-scoped grants that don't require touching group membership at all. It also demonstrates that directory and file permissions are evaluated as fully separate, ordered checks — the kernel never infers file-level access from directory-level access or vice versa, which is exactly what makes "walk through without looking around" (`--x` without `r`) a real, usable scoping tool rather than a theoretical edge case.

---

## Security / Administration Relevance

This is the exact pattern behind real one-time or contractor access requests: an external party needs precisely one resource, and the correct fix is the narrowest grant that satisfies the request — not a group addition that quietly hands over broader access than intended. Proving the denial side (not just the grant) is the step most often skipped in practice, and it's precisely what a compliance audit checks for — access that looks correctly scoped on paper but was never actually verified to stop where it claims to stop. The mid-lab command typo is its own real-world lesson: a verification step is only as trustworthy as the command that produced it, and mistaking a syntax error for a permission denial is an easy, quiet way to certify a fix that was never actually tested.

---

## Time Spent

30 minutes

---

## Conclusion

Validated the full ACL scoping loop — baseline diagnosis, named-entry grant, mask verification, directory traversal as a separate gate, and dual-direction proof of both access and denial — end to end on a live Rocky Linux 9 system. This builds directly toward RHCSA-level access control tasks, and reinforces two operator habits: default to the narrowest grant that satisfies a request, and never accept a verification step's output as proof without confirming the command that ran was actually the one intended.
