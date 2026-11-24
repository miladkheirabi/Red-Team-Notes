# Screen

`screen` is a terminal multiplexer that allows you to create persistent terminal sessions.

It is useful when working over SSH because processes running inside a `screen` session can continue running after the SSH connection is terminated.

---

## Start a Screen Session

```bash
screen
```

Or create a named session:

```bash
screen -S <session_name>
```

Example:

```bash
screen -S pentest
```

---

## Detach from a Session

Detach without terminating the session:

```text
Ctrl+A
D
```

The session continues running in the background.

This is especially useful before disconnecting from an SSH session.

---

## List Sessions

```bash
screen -ls
```

Example:

```text
There are screens on:
    12345.pentest    (Detached)
    67890.scan       (Detached)
```

---

## Reattach to a Session

```bash
screen -r <session_name>
```

Example:

```bash
screen -r pentest
```

You can also use the session ID:

```bash
screen -r 12345
```

If there is only one detached session:

```bash
screen -r
```

---

## Create a Detached Session

Start a session without immediately attaching to it:

```bash
screen -dmS <session_name>
```

Example:

```bash
screen -dmS scan
```

Run a command directly inside a new detached session:

```bash
screen -dmS scan nmap -sT -Pn 10.10.10.0/24
```

---

## Detach and Reattach

Typical SSH workflow:

```text
SSH into target
      │
      ▼
screen -S work
      │
      ▼
Run long-running command
      │
      ▼
Ctrl+A, D
      │
      ▼
Disconnect SSH
      │
      ▼
Reconnect later
      │
      ▼
screen -r work
```

The process continues running while the SSH connection is disconnected.

---

## Multiple Windows

A single `screen` session can contain multiple terminal windows.

Create a new window:

```text
Ctrl+A
C
```

Move to the next window:

```text
Ctrl+A
N
```

Move to the previous window:

```text
Ctrl+A
P
```

List windows:

```text
Ctrl+A
"
```

---

## Name a Window

Rename the current window:

```text
Ctrl+A
Shift+A
```

Enter the new name and press Enter.

---

## Scrollback

Enter scrollback/copy mode:

```text
Ctrl+A
Esc
```

Use the arrow keys or Page Up/Page Down to navigate.

Press `Esc` to exit.

---

## Terminate a Screen Session

Inside the session:

```bash
exit
```

Or:

```text
Ctrl+D
```

To terminate a detached session from outside:

```bash
screen -S <session_name> -X quit
```

Example:

```bash
screen -S pentest -X quit
```

---

## Useful Commands

### List sessions

```bash
screen -ls
```

### Create named session

```bash
screen -S pentest
```

### Reattach

```bash
screen -r pentest
```

### Create detached session

```bash
screen -dmS pentest
```

### Run a command in a detached session

```bash
screen -dmS scan nmap -sT -Pn 10.10.10.0/24
```

### Kill a session

```bash
screen -S pentest -X quit
```

---

## Red Team Use Cases

`screen` is useful for:

- Long-running scans
- Large file transfers
- Enumeration scripts
- Monitoring commands
- Maintaining terminal sessions over unstable SSH connections
- Running multiple tasks on a compromised host

### Example

Start a long-running scan:

```bash
screen -dmS scan nmap -sT -Pn -p- 10.10.10.20
```

Later, reconnect to the host:

```bash
ssh user@target
```

List the sessions:

```bash
screen -ls
```

Reattach:

```bash
screen -r scan
```

---

## Quick Reference

```text
Ctrl+A, D       Detach
Ctrl+A, C       New window
Ctrl+A, N       Next window
Ctrl+A, P       Previous window
Ctrl+A, "       List windows
Ctrl+A, Esc     Scrollback mode
```

```bash
screen -S name
screen -ls
screen -r name
screen -dmS name
screen -dmS name <command>
screen -S name -X quit
```
