# Lab 01 – User, Group, and Permission Basics

## Objective
Practice creating users and groups, modifying ownership, and adjusting permissions.

---

## Commands Practiced

useradd
passwd
groupadd
usermod -aG
chown
chgrp
chmod

---

## What I Did

- Created a test user.
- Created a test group.
- Added the user to the group.
- Created a file owned by root.
- Changed ownership using `chown`.
- Modified group ownership with `chgrp`.
- Tested numeric vs symbolic permission changes using `chmod`.

---

## Observations

- Numeric permissions (e.g., 755, 644) are faster to apply once memorized.
- Symbolic permissions are clearer when adjusting specific access.
- Ownership determines authority before permissions even matter.
- Group membership changes require re-login to take effect.

---

## Operator Notes

Permissions are about least privilege.
Every file and directory represents access control.
Understanding ownership is critical before hardening a system.
