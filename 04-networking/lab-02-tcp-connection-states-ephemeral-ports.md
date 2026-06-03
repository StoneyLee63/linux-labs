# Lab 02 — TCP Connection States & Ephemeral Ports

---

## Objective

- Understand what an outbound HTTPS request looks like at the TCP socket level
- Observe how connection states appear using Linux networking tools
- Build a working mental model of the TCP connection lifecycle

---

## Environment

- **Distribution:** Linux
- **Platform:** Terminal
- **User Context:** Standard user
- **Date:** 2026-03-04

---

## Scenario

An administrator wants to observe TCP socket behavior during a live outbound HTTPS request. Instead of assuming how connections work, the goal is to watch the system directly — before, during, and after a request — to see how the kernel manages socket states and port assignment.

---

## Technical Concepts Covered

- TCP socket states
- Listening sockets vs active connections
- Ephemeral ports
- Outbound HTTPS connections
- TCP connection lifecycle

---

## Commands Used

```bash
ss -tanp
ss -tan
curl https://example.com
ss -s
```

---

## Procedure

1. Ran `ss -tanp` to capture baseline socket state before any network activity.
2. Identified all services in `LISTEN` state.
3. Executed `curl https://example.com` to initiate an outbound HTTPS request.
4. Ran `ss -tan` immediately after to observe connection state.
5. Ran `ss -s` to review connection statistics.

---

## Results

**Before the request** — system showed only `LISTEN` state sockets:

- Port 53 (DNS)
- Port 22 (SSH)
- Port 80 (HTTP)

**After `curl https://example.com`** — connection retrieved the HTML page successfully.

**`ss -tan` output showed:**

```
TIME-WAIT   172.29.34.255:50834 → 104.18.26.120:443
```

- Local ephemeral port assigned: `50834`
- Remote server port: `443` (HTTPS)

**`ss -s`** showed no active established connections at capture time — the request had already completed.

---

## Evidence

```
# Baseline - LISTEN state only
ss -tan
# Port 53, 22, 80 in LISTEN

# After curl
ss -tan
TIME-WAIT   172.29.34.255:50834 → 104.18.26.120:443

# Statistics
ss -s
# 0 established connections
```

---

## Key Takeaways

- The OS assigns a temporary ephemeral port for each outbound connection
- Servers bind to fixed ports; clients use ephemeral ports from a dynamic range
- `TIME-WAIT` confirms the connection completed the full TCP lifecycle
- Short-lived requests can transition from `ESTABLISHED` to `TIME-WAIT` before you can observe them
- `ss` is the modern replacement for `netstat` for socket inspection

---

## What This Demonstrates

- Linux manages outbound connections at the kernel level with automatic port assignment
- TCP states are observable and predictable — not invisible OS magic
- The connection lifecycle (SYN → ESTABLISHED → TIME-WAIT → CLOSED) is verifiable in real time

---

## Operator Rule

Outbound connections use temporary ephemeral ports. Servers listen on fixed ports. Every connection has two endpoints — local (ephemeral) and remote (fixed service port).

---

## Security / Administration Relevance

- Identifying unexpected `ESTABLISHED` connections reveals unauthorized outbound traffic
- Knowing ephemeral port ranges helps with firewall rule design
- `TIME-WAIT` accumulation can indicate connection exhaustion under load
- Baseline socket inspection is the first step in any network troubleshooting workflow

---

## Time Spent

~30 minutes

---

## Conclusion

This lab confirmed that outbound HTTPS requests are visible at the socket level using `ss`. The TCP connection lifecycle — from ephemeral port assignment through `TIME-WAIT` — was directly observable, turning an abstract protocol concept into a real, verifiable system behavior.
