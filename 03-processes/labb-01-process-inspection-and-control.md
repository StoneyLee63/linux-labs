# Lab 01 – Process Inspection and Control

## Objective
Practice inspecting running processes, monitoring resource usage, and controlling process behavior.

---

## Commands Practiced

ps aux
top
htop
kill
kill -9
jobs
bg
fg
nohup

---

## What I Did

- Listed all running processes using `ps aux`.
- Identified specific processes by PID.
- Monitored live CPU and memory usage with `top`.
- Started a background process.
- Brought a job to foreground using `fg`.
- Sent termination signals using `kill`.
- Compared SIGTERM vs SIGKILL behavior.
- Used `nohup` to allow a process to continue after logout.

---

## Observations

- `ps aux` gives static visibility; `top` gives live monitoring.
- SIGTERM allows graceful shutdown.
- SIGKILL forces termination immediately.
- Background jobs are useful but must be tracked.
- PID awareness is critical for safe control.

---

## Operator Notes

Every system compromise, crash, or overload eventually involves processes.

Understanding:
- What is running
- Who owns it
- How much it consumes
- How to stop it safely

...is foundational to system control.
