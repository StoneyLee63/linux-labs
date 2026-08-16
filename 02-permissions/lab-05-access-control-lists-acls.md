# Lab 05 — Access Control Lists (ACLs)

---

## Objective

- Built command-line fluency granting named-user and named-group access to a file without altering ownership or the base permission bits
- Verified the ACL mask as a live, recalculating ceiling rather than a static setting
- Configured a directory-level default ACL and proved inheritance actually fires on newly created files
- Practiced precise, surgical ACL removal (single entry vs. full strip) and confirmed reversibility back to plain permissions
- Directly applicable to RHCSA exam tasks requiring fine-grained access grants beyond the owner/group/other model

---

## Environment

- **Distribution:** Ubuntu (WSL2)
- **Platform:** WSL2 on Windows, host DESKTOP-N8MS0U8
- **User Context:** Standard user (ltksol) with sudo privileges
- **Identities used:** `wateruser01`, `svc_water`, `watercrew` (created in Lab 10)

---

## Scenario

A file needs to be readable by a specific service account and writable by a specific team group — without changing who owns it or restructuring its primary group. A shared directory also needs new files to automatically pick up the same access rule the moment they're created, without re-running permission commands on every new file. This lab built both: named ACL entries on a single file, and a default ACL on a directory with live inheritance verification.

---

## Technical Concepts Covered

- ACLs as a parallel permission layer — named user/group entries on top of the standard owner/group/other model
- `setfacl -m` for adding/modifying entries, `-x` for removing one entry, `-b` for stripping all entries
- The ACL mask as the effective ceiling for every named entry, and its automatic recalculation behavior on both addition and removal
- Default ACLs (`-d`) on directories as forward-looking inheritance rules, not retroactive grants
- Inheritance filtered through a new file's actual creation mode — default ACL grants don't override file-type creation rules
- The `+` flag in `ls -l` as the diagnostic signal that ACLs are present
- Real-world friction: missing `acl` package on Ubuntu, ownership-driven `chmod` permission denial, and mask recalculation silently undoing a manual lock

---

## Commands Used

```bash
mkdir -p ~/water_lab/acl_test && cd ~/water_lab/acl_test
echo "confidential deployment notes" > report.txt
sudo chown wateruser01:wateruser01 report.txt
sudo chmod 640 report.txt
ls -l report.txt

sudo apt install acl -y

sudo setfacl -m u:svc_water:r-- report.txt
getfacl report.txt
ls -l report.txt

sudo setfacl -m g:watercrew:rwx report.txt
getfacl report.txt

sudo setfacl -m m::r-- report.txt
getfacl report.txt

mkdir acl_dir
sudo setfacl -d -m u:svc_water:rwx acl_dir
getfacl acl_dir

touch acl_dir/new_file.txt
getfacl acl_dir/new_file.txt

sudo setfacl -x u:svc_water report.txt
getfacl report.txt

sudo setfacl -b report.txt
getfacl report.txt
ls -l report.txt
```

---

## Procedure

1. Built a test file, assigned it to `wateruser01`, and attempted `chmod 640` as the calling user — failed with "Operation not permitted" since ownership had already moved off the calling account. Corrected with `sudo chmod`.
2. Discovered `setfacl` wasn't installed at all — Ubuntu doesn't ship ACL tooling by default the way most RHEL builds do. Installed via `sudo apt install acl`.
3. Granted `svc_water` a read-only named-user ACL entry — confirmed via `getfacl` and the new `+` flag in `ls -l`.
4. Granted `watercrew` a full rwx named-group ACL entry — confirmed the mask auto-recalculated to `rwx` to accommodate the new grant.
5. Manually capped the mask to `r--` — confirmed `watercrew`'s granted `rwx` now showed `#effective:r--`, proving granted and effective permission are tracked separately.
6. Created `acl_dir` and set a default ACL granting `svc_water` rwx on any future file created inside it.
7. Created a new file with `touch` inside `acl_dir` and ran `getfacl` on it without ever touching it directly — confirmed it inherited the default ACL, but with execute stripped (`#effective:rw-`) since `touch` never creates an executable file.
8. Removed only the `svc_water` entry from `report.txt` with `setfacl -x` — discovered this silently reset the manually-locked mask from Step 5 back to `rwx`, undoing the earlier cap.
9. Stripped all ACLs from `report.txt` with `setfacl -b` — confirmed full reversion to plain permissions, no `+` flag, no named entries remaining.

---

## Results

- Named-user and named-group ACL grants applied and verified successfully without any change to file ownership or base permission bits.
- **Package divergence:** `acl` is not installed by default on this Ubuntu image — required manual installation, unlike most RHEL environments where ACL support ships in the base install.
- **Ownership/chmod interaction confirmed live:** losing ownership of a file via `chown` immediately removed the calling user's ability to `chmod` it — required `sudo`. Real-world proof, not just theory, of Day 1's ownership/permission independence lesson.
- **Mask behavior confirmed volatile in both directions:** the mask recalculated automatically not just when adding an entry (Step 4) but also when removing one (Step 8) — a manually locked mask does not survive a later, unrelated `setfacl -x`.
- **Default ACL inheritance confirmed live but filtered:** a new file created inside a directory with a default ACL inherited the rule automatically, but the execute bit was stripped because `touch`-created files never carry execute permission at creation, regardless of what the default ACL specifies.
- Full ACL removal (`-x` for one entry, `-b` for all) confirmed surgical and fully reversible.

---

## Evidence

**Ownership/chmod permission denial**
```
$ chmod 640 report.txt
chmod: changing permissions of 'report.txt': Operation not permitted
$ sudo chmod 640 report.txt
$ ls -l report.txt
-rw-r----- 1 wateruser01 wateruser01 30 Jun 22 15:37 report.txt
```

**Named-user ACL grant**
```
# file: report.txt
# owner: wateruser01
# group: wateruser01
user::rw-
user:svc_water:r--
group::r--
mask::r--
other::---
```
```
-rw-r-----+ 1 wateruser01 wateruser01 30 Jun 22 15:37 report.txt
```

**Mask capped below a granted entry**
```
user::rw-
user:svc_water:r--
group::r--
group:watercrew:rwx          #effective:r--
mask::r--
other::---
```

**Default ACL on directory**
```
# file: acl_dir
# owner: ltksol
# group: ltksol
user::rwx
group::r-x
other::r-x
default:user::rwx
default:user:svc_water:rwx
default:group::r-x
default:mask::rwx
default:other::r-x
```

**Inheritance on a newly created file, execute bit filtered out**
```
# file: acl_dir/new_file.txt
user::rw-
user:svc_water:rwx           #effective:rw-
group::r-x                   #effective:r--
mask::rw-
other::r--
```

**Mask silently recalculated after entry removal**
```
$ sudo setfacl -x u:svc_water report.txt
$ getfacl report.txt
user::rw-
group::r--
group:watercrew:rwx
mask::rwx
other::---
```

**Full strip back to plain permissions**
```
# file: report.txt
user::rw-
group::r--
other::---
```
```
-rw-r----- 1 wateruser01 wateruser01 30 Jun 22 15:37 report.txt
```

---

## Key Takeaways

- ACLs solve a real gap in the owner/group/other model: precise, named access without restructuring ownership or group membership — directly useful for service accounts and cross-team file sharing.
- The mask is not a "set once" value — it recalculates on both additions and removals unless explicitly suppressed, meaning a manually locked mask can be silently undone by a later, seemingly unrelated `setfacl` command.
- Default ACLs describe future inheritance, not current access, and that inheritance is filtered through the new file's actual creation mode — a default ACL granting execute will not produce an executable plain file via `touch`.
- Losing file ownership via `chown` immediately removes your own ability to `chmod` that file — ownership and permission control are tied together in a way that only shows up as a real denial once you've actually given ownership away.
- Package availability is not guaranteed across distros — verify tooling (`setfacl`, `getfacl`) is installed before building a lab around it, especially on minimal or fresh Ubuntu environments.

---

## What This Demonstrates

- **Layered permission model proven, not assumed:** named ACL entries coexist with and don't replace the standard owner/group/other bits — confirmed by `getfacl` showing both simultaneously.
- **Mask volatility proven through direct contradiction:** a manual lock set in Step 5 was confirmed gone by Step 8's `getfacl` output — direct evidence the mask is recalculated on a wider range of operations than just additions.
- **Inheritance behavior proven through controlled test:** creating an untouched file and immediately checking its ACL state is the only way to confirm default ACLs actually fire, rather than just trusting the directory's stated intent.
- **Cross-domain reinforcement:** Day 1's ownership/permission independence lesson surfaced again here as a live permission denial, not just a repeated explanation — proof the concept holds under a different real scenario.

---

## Security / Administration Relevance

- **Precise access grants without privilege sprawl:** ACLs let an admin grant exactly the access a specific account needs without adding it to a broader group or changing file ownership — reduces blast radius compared to looser group-based solutions.
- **Mask awareness as an audit requirement:** any permission audit involving ACLs has to check the mask explicitly — granted permissions in `getfacl` output can be misleading if the effective value isn't separately confirmed.
- **Default ACL planning for shared infrastructure:** the exact mechanism behind setting up shared directories where every new file automatically gets the correct access rule — critical for team folders, log directories, and deployment staging areas.
- **Package availability checks as a hardening step:** confirming ACL tooling is installed (and not silently missing) before relying on it for an access control strategy avoids a false sense of security on minimal Ubuntu installs.

---

## Time Spent

Approximately 40 minutes

---

## Conclusion

Ran the full ACL lifecycle — grant, mask interaction, default inheritance, surgical removal, full strip — against real files and real identities built in the previous rep. Three genuine friction points surfaced: a missing package, a live ownership/chmod permission denial, and a mask recalculation that silently undid an earlier manual lock. The inheritance test on `new_file.txt` was the most valuable single result, proving default ACLs filter through actual file-creation mode rather than blindly copying the directory's stated rule. Water's access-control layer is now functional on top of Day 1's permissions and Day 2's identities — the domain's core mechanics (ownership, identity, fine-grained access) are all verified and connected.
