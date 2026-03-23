# Lab 02 — Inspecting Process Ownership in Linux

---

## Objective

Practice identifying process ownership and user identity in Linux to understand how running programs inherit permissions from the user that launched them.

---

## Environment

- Distribution: Ubuntu
- Platform: WSL
- User Context: Standard user with sudo privileges

---

## Scenario

An administrator needs to determine which user owns specific processes running on a Linux system.

This is a common task when:

- diagnosing permission issues
- auditing system activity
- investigating suspicious processes

This lab verifies how Linux assigns user identity to processes and how administrators inspect that ownership.

---

## Technical Concepts Covered

- User identity (UID)
- Group membership
- Process ownership
- User-based process filtering
- Process inheritance

---

## Commands Used

whoami  
id  
ps aux  
ps -u $(whoami)  
ps -u root  
sleep 200  
ps aux | grep sleep  
ps -fp 5509  
sudo ps -fp 5511

---

## Procedure

1. Verified the current user session.

whoami

2. Inspected UID and group memberships.

id

3. Listed all running processes and their owners.

ps aux

4. Filtered processes owned by the current user.

ps -u $(whoami)

5. Inspected processes owned by the root user.

ps -u root

6. Created a temporary test process.

sleep 200

7. Located the process in the system process table.

ps aux | grep sleep

8. Inspected the running process using its PID.

ps -fp 5509

9. Attempted to inspect the grep process after execution.

sudo ps -fp 5511

---

## Results

The lab confirmed several important behaviors:

- The active user was UID 1000 (ltksoul1205)
- System services were running under root ownership
- User shell sessions and commands ran under the user UID
- Creating a `sleep 200` command produced a visible process owned by the user
- The `grep` process appeared briefly and terminated before inspection

This demonstrates how Linux assigns ownership to processes based on the user that launched them.

---

## Evidence

Example process discovery:

ps aux | grep sleep  
  
ltksoul+ 5509  sleep 200  
ltksoul+ 5511  grep sleep

Inspection of the running process:

ps -fp 5509  
  
UID        PID  PPID  CMD  
ltksoul+  5509  ...   sleep 200

Attempting to inspect the grep process returned no output because the process had already exited.

---

## Key Takeaways

- Every Linux process runs under a specific user identity
- Processes inherit the UID of the user that launched them
- System services typically run as root
- Many command-line processes are short-lived

---

## What This Demonstrates

This lab validates how Linux enforces process ownership and identity.

It confirms that:

- user identity governs process permissions
- administrators can audit process ownership using `ps`
- temporary processes may disappear quickly after execution

---

## Security / Administration Relevance

Process ownership inspection is important for:

- troubleshooting permission errors
- auditing system activity
- detecting abnormal user behavior
- investigating potential intrusions
- validating service ownership

Security analysts frequently analyze process ownership and hierarchy during incident response.

---

## Time Spent

60 minutes  
(Execution + validation + documentation)

---

## Conclusion

This lab demonstrated how Linux assigns ownership to processes and how administrators inspect that ownership.

Using `whoami`, `id`, and `ps`, administrators can determine which users control running programs, enabling effective system monitoring, troubleshooting, and security analysis.
