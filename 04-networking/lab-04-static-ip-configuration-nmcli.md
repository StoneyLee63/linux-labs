# Lab 04 — Static IP Configuration with nmcli

## Objective

Practice converting an interface from DHCP to a static IP using `nmcli`, and prove the configuration survives a reboot. This matters because the RHCSA exam grades persistence, not just whether a command appears to succeed in the moment.

## Environment

- **Distribution:** Rocky Linux 9
- **Platform:** VM (Oracle VirtualBox)
- **User Context:** Standard user (`studysol`), using `sudo` for the reboot

## Scenario

Simulating a server handoff where the requirement is: configure a static IP, gateway, and DNS that persist across a restart. Standard real-world task — replacing DHCP-assigned addressing with a fixed, predictable address before the host goes into production use.

## Technical Concepts Covered

- NetworkManager connection profiles vs. live kernel state
- `nmcli connection modify` (staging) vs. `nmcli connection up` (activation)
- `ipv4.method manual` vs. `auto`
- `connection.autoconnect` and boot-time persistence
- Shell quoting behavior (unclosed string → hung prompt)

## Commands Used

```shell
nmcli device status
ip addr show
nmcli connection show
nmcli connection modify "enp0s3" ipv4.addresses 192.168.1.50/24
nmcli connection modify "enp0s3" ipv4.gateway 192.168.1.1
nmcli connection modify "enp0s3" ipv4.dns "8.8.8.8 1.1.1.1"
nmcli connection modify "enp0s3" ipv4.method manual
nmcli connection modify "enp0s3" connection.autoconnect yes
nmcli connection up "enp0s3"
ip addr show enp0s3
nmcli connection show "enp0s3" | grep -E "ipv4.address|ipv4.gateway|ipv4.dns|ipv4.method|autoconnect"
sudo reboot
nmcli device status
ip addr show enp0s3
```

## Procedure

1. Captured baseline state with `nmcli device status` and `ip addr show` — confirmed `enp0s3` was active with a DHCP-assigned address.
2. Used `nmcli connection show` to identify the exact profile name bound to the device, since modifications target the profile, not the device directly.
3. Staged the static address, gateway, and DNS into the profile with `nmcli connection modify` — no live effect at this point.
4. Switched `ipv4.method` to `manual`, the field that actually stops DHCP from overriding the staged values.
5. Set `connection.autoconnect yes` to guarantee the profile activates on boot without manual intervention.
6. Applied the staged profile live with `nmcli connection up "enp0s3"`.
7. Verified the result at both the kernel layer (`ip addr show`) and the profile layer (`nmcli connection show | grep`).
8. Rebooted the system with `sudo reboot` and re-checked both layers post-boot to confirm persistence.

## Results

The configuration worked as expected. After activation, `ip addr show enp0s3` matched the staged profile exactly: address `192.168.1.50/24`, gateway `192.168.1.1`, DNS `8.8.8.8` and `1.1.1.1`. After a full reboot, `enp0s3` returned in `connected` state carrying the same static address with zero manual reapplication — confirming the config was genuinely persistent, not just active in the live session.

## Evidence

```shell
[studysol@localhost ~]$ nmcli connection show "enp0s3" | grep -E "ipv4.address|ipv4.gateway|ipv4.dns|ipv4.method|autoconnect"
connection.autoconnect:                yes
ipv4.method:                            manual
ipv4.dns:                               8.8.8.8,1.1.1.1
ipv4.addresses:                         192.168.1.50/24
ipv4.gateway:                           192.168.1.1
```

```shell
[studysol@localhost ~]$ nmcli device status
DEVICE  TYPE      STATE                   CONNECTION
enp0s3  ethernet  connected               enp0s3
lo      loopback  connected (externally)  lo

[studysol@localhost ~]$ ip addr show enp0s3
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:54:93:14 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.50/24 brd 192.168.1.255 scope global noprefixroute enp0s3
```
*(captured after reboot — confirms persistence)*

## Key Takeaways

- A static IP only counts as "configured" once it survives a reboot — live-session success and persistence are separate claims.
- `ipv4.method` and `connection.autoconnect` are the two fields responsible for almost every "static IP that didn't stick" failure.
- An unclosed quote in a multi-line shell command doesn't error out — it hangs the prompt and silently swallows whatever's typed next, which can look like a frozen terminal if you don't recognize the `>` continuation prompt.

## What This Demonstrates

This lab validates that Linux network configuration is a three-layer write — connection profile, NetworkManager, kernel — and all three must agree for a configuration to be both live and persistent. It confirms the specific mechanism (`ipv4.method` + `connection.autoconnect`) that Linux uses to decide whether a network config is rebuilt identically at every boot, and validates the verification discipline needed to prove that rather than assume it.

## Security / Administration Relevance

Predictable, static addressing is a prerequisite for firewall rules, access control lists, and monitoring configurations scoped to a specific host IP — none of that holds up against an address that can silently change. Confirming persistence also has direct incident response value: a host that "loses" its intended static configuration on reboot can come back on an unexpected address, breaking monitoring coverage and firewall scoping during the exact window when visibility matters most.

## Time Spent

45 minutes (Execution + validation + documentation)

## Time Tracking Rule

- Start timer at hands-on execution.
- Stop timer after documentation is complete.
- Round to nearest 5 minutes.
- Record honestly.
- No lab is published without time logged.

## Conclusion

Static IP configuration via `nmcli` is a three-stage process — stage the profile, switch the method to manual, set autoconnect, then apply — and persistence depends specifically on `ipv4.method manual` and `connection.autoconnect yes`. This lab validated the full workflow end to end on a real RHEL-family system, including a recovery moment (an unclosed-quote shell hang) that reinforced the same diagnostic discipline the configuration itself required: don't react to the symptom, confirm what the system is actually waiting on.
