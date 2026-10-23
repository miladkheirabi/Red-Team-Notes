# SSH Tunneling & Port Forwarding

SSH can forward TCP connections through an SSH session.

## Local Port Forwarding — `-L`

Syntax:

```bash
ssh -L <local_port>:<destination>:<destination_port> <user>@<ssh_server>
```

The important point:

> A connection to the **local port** is forwarded through the SSH server to the destination.

### Example

Assume:

```text
Attacker:  10.10.10.5
Target:    10.10.10.10
Internal:  10.10.20.50:80
```

The attacker cannot directly reach `10.10.20.50:80`, but the SSH server can.

Run:

```bash
ssh -L 8080:10.10.20.50:80 user@10.10.10.10
```

Breakdown:

```text
-L
8080             → Listen on port 8080 on the attacker
10.10.20.50     → Destination from the SSH server's perspective
80               → Destination port
user@10.10.10.10 → SSH server
```

Traffic flow:

```text
Attacker
   │
   │ localhost:8080
   ▼
SSH Server
   │
   │ 10.10.20.50:80
   ▼
Internal Server
```

Now accessing:

```bash
curl http://127.0.0.1:8080
```

actually reaches:

```text
10.10.20.50:80
```

through the SSH server.

---

## Remote Port Forwarding — `-R`

Syntax:

```bash
ssh -R <remote_port>:<destination>:<destination_port> <user>@<ssh_server>
```

The important point:

> A connection to the **remote port** is forwarded through the SSH connection to the destination.

### Example

```bash
ssh -R 8080:127.0.0.1:80 user@10.10.10.10
```

Breakdown:

```text
-R
8080          → Listen on port 8080 on the SSH server
127.0.0.1    → Destination from the SSH client's perspective
80            → Destination port
```

Traffic flow:

```text
Remote System
    │
    │ SSH-Server:8080
    ▼
SSH Connection
    │
    ▼
SSH Client
    │
    │ 127.0.0.1:80
    ▼
Local Web Server
```

This is useful when a service accessible from the SSH client needs to be exposed through the SSH server.

---

## Remote Forwarding Example

A common forwarding pattern:

```bash
ssh -R 0.0.0.0:443:127.0.0.1:8080 <user>@<ip>
```

Breakdown:

```text
0.0.0.0:443
    │
    └── Listen on port 443 on the SSH server

127.0.0.1:8080
    │
    └── Forward traffic to port 8080 from the SSH client's perspective

<user>@<ip>
    │
    └── SSH server
```

Traffic flow:

```text
External Host
     │
     │ SSH Server:443
     ▼
SSH Server
     │
     │ SSH tunnel
     ▼
SSH Client
     │
     │ 127.0.0.1:8080
     ▼
Local Service
```

### Important

Binding to `0.0.0.0` makes the remote forwarded port potentially accessible from other hosts, depending on firewall rules and SSH configuration.

By default, remote forwarding may only listen on loopback. To allow non-loopback connections, `GatewayPorts` may need to be enabled.

---

# SSH Server Configuration

SSH server settings are commonly configured in:

```text
/etc/ssh/sshd_config
```

After changing the configuration, validate it:

```bash
sshd -t
```

If there is no output, the configuration is syntactically valid.

---

## AllowTcpForwarding

Controls whether TCP forwarding is permitted.

```text
AllowTcpForwarding yes
```

Common values:

```text
yes
no
local
remote
```

For tunneling, forwarding must be permitted by the SSH server configuration.

---

## GatewayPorts

Controls whether remote forwarded ports can bind to non-loopback addresses.

```text
GatewayPorts yes
```

This is particularly relevant when using:

```bash
ssh -R 0.0.0.0:443:127.0.0.1:8080 <user>@<ip>
```

Without an appropriate `GatewayPorts` setting, the server may restrict the remote forward to loopback.

---

## Restart SSH

After changing `sshd_config`, restart the SSH service.

Depending on the distribution:

```bash
sudo systemctl restart ssh
```

or:

```bash
sudo systemctl restart sshd
```

Before restarting an SSH server over a remote session, validate the configuration first:

```bash
sudo sshd -t
```

Keep an existing SSH session open while testing the new configuration to avoid locking yourself out.

---

# SOCKS Proxy — `-D`

Dynamic forwarding creates a SOCKS proxy.

```bash
ssh -D 1080 <user>@<ip>
```

This makes the SSH client listen locally on:

```text
127.0.0.1:1080
```

Applications configured to use this SOCKS proxy can send their traffic through the SSH server.

Traffic flow:

```text
Application
    │
    │ SOCKS
    ▼
127.0.0.1:1080
    │
    │ SSH tunnel
    ▼
SSH Server
    │
    ▼
Destination
```

---

## Using Proxychains

Configure `proxychains` to use the local SOCKS proxy:

```text
socks5 127.0.0.1 1080
```

Then commands can be routed through the SSH tunnel:

```bash
proxychains nmap -sT -Pn <target>
```

For example:

```bash
ssh -D 1080 <user>@<ip>
```

Then:

```bash
proxychains nmap -sT -Pn 10.10.20.50
```

The connection path becomes:

```text
Nmap
  │
  ▼
Proxychains
  │
  ▼
SOCKS 127.0.0.1:1080
  │
  ▼
SSH Server
  │
  ▼
10.10.20.50
```

### Nmap Note

Use TCP connect scanning (`-sT`) when working through a SOCKS proxy. Raw-packet techniques such as SYN scanning (`-sS`) generally do not work through a normal SOCKS proxy.

---

## Quick Reference

```bash
# Local forwarding
ssh -L 8080:10.10.20.50:80 user@10.10.10.10

# Remote forwarding
ssh -R 8080:127.0.0.1:80 user@10.10.10.10

# Remote forwarding exposed on all interfaces
ssh -R 0.0.0.0:443:127.0.0.1:8080 user@10.10.10.10

# Dynamic SOCKS proxy
ssh -D 1080 user@10.10.10.10

# Validate SSH configuration
sudo sshd -t

# Restart SSH
sudo systemctl restart ssh
# or
sudo systemctl restart sshd
```
