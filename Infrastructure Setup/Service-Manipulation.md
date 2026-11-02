# Service Manipulation

Linux services can be managed using the traditional `service` command or, on systems using `systemd`, with `systemctl`.

For Red Team operations, service manipulation is useful for:

- Checking whether a service is running
- Starting/stopping services
- Enabling/disabling services at boot
- Identifying available services
- Troubleshooting remote access

---

## 1. Listing Services

With `systemctl`:

```bash
systemctl list-unit-files --type=services
```

This lists installed service unit files and their enablement state.

Common states include:

| State | Meaning |
|---|---|
| `enabled` | Service is configured to start automatically |
| `disabled` | Service exists but is not configured to start automatically |
| `static` | Cannot be enabled directly; usually started as a dependency or manually |
| `indirect` | Not enabled directly, but may be enabled through another unit |
| `masked` | Service is completely blocked from being started |
| `alias` | Alternative name pointing to another service unit |

### Important

`enabled` and `running` are not the same thing.

For example:

```text
enabled + stopped
```

means the service will start automatically during boot, but it is **not currently running**.

Similarly:

```text
disabled + running
```

means the service is currently running, but it is not configured to start automatically after reboot.

---

## 2. Using `service`

The traditional `service` command can still be used on many modern Linux distributions.

List service statuses:

```bash
service --status-all
```

Check a specific service:

```bash
service sshd status
```

Depending on the distribution, the service may be named `ssh` instead of `sshd`:

```bash
service ssh status
```

---

## 3. Checking Service Status with systemctl

The modern equivalent is:

```bash
systemctl status sshd
```

This provides information such as:

- Whether the service is running
- When it started
- The main process PID
- Recent log output
- Whether the service failed

For example:

```text
Active: active (running)
```

means the service is currently running.

---

## 4. Enable and Disable

Enable a service to start automatically during boot:

```bash
systemctl enable sshd
```

Disable automatic startup:

```bash
systemctl disable sshd
```

These commands control **startup behavior**, not necessarily the current running state.

For example:

```bash
systemctl disable sshd
```

does not necessarily stop an already-running SSH server.

To stop the currently running service:

```bash
systemctl stop sshd
```

---

## 5. Start and Stop

Stop a running service:

```bash
systemctl stop sshd
```

Start a stopped service:

```bash
systemctl start sshd
```

Restart a service:

```bash
systemctl restart sshd
```

Reload its configuration without fully restarting it, when supported:

```bash
systemctl reload sshd
```

A useful distinction:

```text
start / stop / restart
        |
        v
Current runtime state

enable / disable
        |
        v
Boot-time behavior
```

---

## 6. SSH Session and Restarting sshd

A common question is:

> If you are connected over SSH and `sshd` is restarted from another terminal, why doesn't your existing SSH session necessarily disconnect?

Because restarting `sshd` normally affects the **server process that accepts new connections**, while existing SSH connections are handled by their already-established session processes; therefore, restarting the listening daemon does not inherently terminate existing sessions.

This is also why:

```bash
systemctl restart sshd
```

can normally be performed without immediately killing existing SSH sessions.

However, configuration changes, service failures, or commands that explicitly terminate the session processes can still disconnect existing users.

---

## Quick Reference

### List installed services

```bash
systemctl list-unit-files --type=services
```

### List service status

```bash
service --status-all
```

### Check service status

```bash
service sshd status
```

or:

```bash
systemctl status sshd
```

### Enable at boot

```bash
systemctl enable sshd
```

### Disable at boot

```bash
systemctl disable sshd
```

### Start

```bash
systemctl start sshd
```

### Stop

```bash
systemctl stop sshd
```

### Restart

```bash
systemctl restart sshd
```

### Reload configuration

```bash
systemctl reload sshd
```
