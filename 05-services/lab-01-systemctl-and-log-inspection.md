# Lab 01 – systemctl and Log Inspection

## Objective
Practice managing services using systemctl and inspecting logs with journalctl.

---

## Commands Practiced

systemctl status
systemctl start
systemctl stop
systemctl restart
systemctl enable
systemctl disable
journalctl
journalctl -xe
journalctl -u <service-name>

---

## What I Did

- Checked service status using `systemctl status`.
- Started and stopped a service.
- Restarted a service to apply changes.
- Enabled a service to start on boot.
- Disabled a service from auto-start.
- Reviewed logs using `journalctl`.
- Filtered logs for a specific service.

---

## Observations

- `systemctl status` provides both state and recent log entries.
- Services can fail silently without log review.
- `journalctl -u` isolates service-specific logs.
- Boot persistence is controlled through enable/disable.

---

## Operator Notes

Service management determines:

- What runs automatically
- What survives reboot
- What fails and why

Logs are not optional. They are primary evidence in troubleshooting and incident response.
