# Lab 01 – Interface and Port Visibility

## Objective
Practice inspecting network interfaces, active connections, and listening ports.

---

## Commands Practiced

ip a
ip route
ss -tulnp
ping
curl
netstat (if available)

---

## What I Did

- Inspected network interfaces using `ip a`.
- Reviewed routing table with `ip route`.
- Identified listening ports using `ss -tulnp`.
- Tested connectivity with `ping`.
- Verified HTTP response using `curl`.

---

## Observations

- `ip a` shows interface state and assigned addresses.
- `ip route` reveals default gateway and routing behavior.
- `ss -tulnp` exposes listening services and their PIDs.
- Every open port corresponds to a running process.
- Connectivity tests confirm routing and DNS functionality.

---

## Operator Notes

Networking visibility answers:

- What interfaces are active?
- What ports are exposed?
- What services are listening?
- Where is traffic routed?

Port awareness is critical for both defense and troubleshooting.
