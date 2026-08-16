# Lab 05 — Hostname Resolution with hostnamectl and /etc/hosts

## Objective

Practice setting a persistent static hostname and controlling local name resolution before DNS is ever consulted. This matters because several RHCSA-adjacent tasks (NFS mounts, remote service checks, cluster-style configs) silently depend on correct hostname resolution underneath them — a broken resolution chain produces failures that look unrelated to naming.

## Environment

- **Distribution:** Rocky Linux 9.8 (Blue Onyx)
- **Platform:** VM (Oracle VirtualBox)
- **User Context:** Standard user (`studysol`), using `sudo` for hostname change and `/etc/hosts` write

## Scenario

Simulating a freshly provisioned server with no real identity — `hostnamectl` showed a static hostname of `(unset)` and only a generic transient `localhost`. Task: assign a proper FQDN, anchor a local resolution entry for it, and confirm the resolution order before relying on it.

## Technical Concepts Covered

- Static vs. transient hostname persistence
- `/etc/hosts` as a local, file-based resolution source
- `/etc/nsswitch.conf` resolution order (`files` vs `dns`)
- FQDN vs. short alias
- `getent` as a chain-level verification tool vs. raw file inspection

## Commands Used

```shell
hostnamectl status
sudo hostnamectl set-hostname rocky9-air.echolab.local
cat /etc/hostname
hostnamectl status
echo "192.168.1.50 rocky9-air.echolab.local rocky9-air" | sudo tee -a /etc/hosts
cat /etc/nsswitch.conf | grep ^hosts
getent hosts rocky9-air
ping -c 2 rocky9-air.echolab.local
```

## Procedure

1. Ran `hostnamectl status` to capture baseline — found **Static hostname: (unset)**, transient hostname `localhost`.
2. Set a real static hostname with `sudo hostnamectl set-hostname rocky9-air.echolab.local`.
3. Verified persistence directly by reading `/etc/hostname`, then cross-checked with `hostnamectl status` to confirm both layers agreed.
4. Appended a local resolution entry to `/etc/hosts`, mapping the Day 1 static IP (`192.168.1.50`) to both the FQDN and a short alias.
5. Checked `/etc/nsswitch.conf` to confirm the resolution order — `files` listed before `dns`, meaning local entries are always checked first.
6. Verified resolution through the actual chain with `getent hosts rocky9-air`, then confirmed end-to-end reachability with `ping -c 2`.

## Results

Worked as expected at every step. The static hostname changed immediately and persisted to disk — `/etc/hostname` and `hostnamectl status` matched exactly. The `/etc/hosts` entry resolved correctly through `getent`, proving the lookup chain (not just the file) was functioning. `ping` confirmed end-to-end reachability with `0% packet loss` and sub-millisecond RTT, since the address resolves to a locally-bound interface from the prior lab.

## Evidence

```shell
[studysol@localhost ~]$ hostnamectl status
  Static hostname: (unset)
Transient hostname: localhost
...
[studysol@localhost ~]$ sudo hostnamectl set-hostname rocky9-air.echolab.local
[studysol@localhost ~]$ cat /etc/hostname
rocky9-air.echolab.local
[studysol@localhost ~]$ hostnamectl status
 Static hostname: rocky9-air.echolab.local
...
```

```shell
[studysol@localhost ~]$ echo "192.168.1.50 rocky9-air.echolab.local rocky9-air" | sudo tee -a /etc/hosts
192.168.1.50 rocky9-air.echolab.local rocky9-air
[studysol@localhost ~]$ cat /etc/nsswitch.conf | grep ^hosts
hosts:      files dns myhostname
[studysol@localhost ~]$ getent hosts rocky9-air
192.168.1.50    rocky9-air.echolab.local rocky9-air
[studysol@localhost ~]$ ping -c 2 rocky9-air.echolab.local
PING rocky9-air.echolab.local (192.168.1.50) 56(84) bytes of data.
64 bytes from rocky9-air.echolab.local (192.168.1.50): icmp_seq=1 ttl=64 time=0.177 ms
64 bytes from rocky9-air.echolab.local (192.168.1.50): icmp_seq=2 ttl=64 time=0.155 ms

--- rocky9-air.echolab.local ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1029ms
rtt min/avg/max/mdev = 0.155/0.166/0.177/0.011 ms
```

## Key Takeaways

- An unset static hostname with only a generic transient name is a red flag on a real system — not a harmless default.
- `/etc/hosts` is checked before DNS by default on Rocky 9 — a stale local entry will silently win over a correct DNS record, with zero warning.
- `getent hosts` is the correct verification command because it tests the full resolution chain as an application would use it, not just whether a file contains the expected line.

## What This Demonstrates

This lab validates that hostname identity and name resolution are governed by separate but related mechanisms — `hostnamectl` controls persistent machine identity, while `/etc/hosts` plus `/etc/nsswitch.conf` control how any name (including the hostname itself) actually resolves on this system. It confirms the local-before-DNS resolution order that Rocky 9 ships with by default, and validates the discipline of checking the resolution chain directly rather than assuming a file edit is sufficient.

## Security / Administration Relevance

Correct hostname and resolution configuration underpins service identity verification, certificate matching (FQDN-bound certs), and log/monitoring attribution — a host that resolves inconsistently or under the wrong name can produce misleading audit trails and break services that validate identity by name. Knowing that `/etc/hosts` overrides DNS by default is also a hardening and troubleshooting consideration: an attacker or a misconfiguration that plants a bad entry there can silently redirect name resolution for any service that trusts hostnames over IPs, with no indication unless someone checks the file directly.

## Time Spent

25 minutes (Execution + validation + documentation)

## Time Tracking Rule

- Start timer at hands-on execution.
- Stop timer after documentation is complete.
- Round to nearest 5 minutes.
- Record honestly.
- No lab is published without time logged.

## Conclusion

Hostname identity (`hostnamectl`) and local name resolution (`/etc/hosts`, `/etc/nsswitch.conf`) are distinct mechanisms that have to be configured and verified together — a real hostname means nothing if resolution can't find it, and a resolution entry means nothing if checked with the wrong tool. This lab validated the full chain end to end on Rocky Linux 9, using `getent` as the correct verification layer rather than relying on raw file contents alone.
