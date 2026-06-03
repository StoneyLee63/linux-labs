# Lab 03 — Controlling Linux Processes with Signals

---

## Objective

Practice identifying running processes and controlling them using Linux signals in order to safely terminate or manage system activity without restarting services or rebooting the machine.

---

## Environment

- **Distribution:** Linux WSL
- **User Context:** Standard user with sudo privileges

---

## Scenario

An administrator discovers a process consuming system resources and needs to inspect and terminate the process safely.

Instead of rebooting the machine, Linux administrators interact with processes using signals.

This lab demonstrates how to:

- Create a background process
- Identify the process in the system process table
- Terminate the process gracefully
- Force termination when necessary
- Verify system state after termination

---

## Technical Concepts Covered

- Process IDs (PID)
- Background process execution
- Process inspection
- Signal-based process control
- Graceful vs forced termination

---

## Commands Used

```bash
sleep 300 &
ps aux | grep sleep
kill PID
kill -9 PID
ps aux | grep sleep
```

---

## Procedure

1. Created a background process that runs independently of the terminal.
```bash
sleep 300 &
```

2. Verified the running process in the system process table.
```bash
ps aux | grep sleep
```

3. Identified the process ID (PID) associated with the sleep command.

4. Sent a graceful termination signal to the process.
```bash
kill PID
```

5. Verified that the process terminated successfully.
```bash
ps aux | grep sleep
```

6. Created another background process for further testing.
```bash
sleep 300 &
```

7. Sent a forced termination signal to the process.
```bash
kill -9 PID
```

8. Verified that the process was removed from the system process table.
```bash
ps aux | grep sleep
```

---

## Results

The lab confirmed several important Linux behaviors:

- Background processes continue running after the terminal prompt returns
- Each process is assigned a unique Process ID (PID) by the kernel
- Sending `kill` sends the SIGTERM signal, which requests graceful shutdown
- Sending `kill -9` sends SIGKILL, forcing immediate termination
- Verifying process lists confirms whether the process was successfully removed

---

## Evidence

```
ps aux | grep sleep
ltksoul+ 1933  sleep 300
ltksoul+ 1935  grep sleep

# After graceful termination:
[1]+ Terminated    sleep 300

# After forced termination:
[1]+ Killed        sleep 300
```

Verification showed no remaining `sleep` processes.

---

## Key Takeaways

- Every running program in Linux is a process managed by the kernel
- Administrators control processes using signals, not direct termination commands
- SIGTERM allows programs to shut down cleanly
- SIGKILL forces immediate termination when necessary
- Always verify system state after terminating a process

---

## What This Demonstrates

- Administrators interact with processes through signals
- Graceful shutdown should be attempted before forced termination
- Process state can be verified through system inspection tools

Understanding this mechanism is essential for system troubleshooting and operational control.

---

## Security / Administration Relevance

Process control is critical for:

- Stopping runaway processes
- Troubleshooting system slowdowns
- Managing system resources
- Responding to incidents
- Controlling long-running tasks

Administrators frequently rely on signals to stabilize systems without interrupting other services.

---

## Time Spent

25 minutes

---

## Conclusion

This lab demonstrated how Linux administrators inspect and control processes using signals. By creating, identifying, and terminating processes, the exercise reinforced how Linux manages system activity and how administrators maintain control of running programs without disrupting the entire system.
