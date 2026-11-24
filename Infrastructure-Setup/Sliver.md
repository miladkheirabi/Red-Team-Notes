# Sliver

A redirector is an intermediate server used to relay network traffic between systems. Before configuring any forwarding mechanism, the server should be hardened and its exposed network services should be restricted.

## 1. Restrict SSH Access

First, identify the network interface:

```bash
ip -br addr
```

Then allow SSH access only from a trusted source IP:

```bash
iptables -A INPUT -i <interface> -p tcp --dport 22 -m conntrack --ctstate NEW,ESTABLISHED -s <your_ip> -j ACCEPT
```

### Command Breakdown

| Component | Description |
|---|---|
| `iptables` | Linux firewall administration utility |
| `-A INPUT` | Append a rule to the INPUT chain |
| `-i <interface>` | Match packets arriving through a specific interface |
| `-p tcp` | Match TCP packets |
| `--dport 22` | Match packets destined for TCP port 22 |
| `-m conntrack` | Enable connection-tracking state matching |
| `--ctstate NEW,ESTABLISHED` | Match new and established connections |
| `-s <your_ip>` | Match packets originating from the specified IP |
| `-j ACCEPT` | Accept matching packets |

The source IP restriction is important: SSH connections from other addresses will not match this rule.

## 2. Verify Firewall Rules

After adding a rule, inspect the current firewall configuration:

```bash
iptables -nvL
```

Useful options:

- `-n`: Display IP addresses and port numbers numerically.
- `-v`: Display detailed information, including packet and byte counters.
- `-L`: List firewall chains and their rules.

Example output:

```text
Chain INPUT (policy ACCEPT)
 pkts bytes target  prot opt in     out source       destination
   12   720 ACCEPT  tcp  --  eth0   *   203.0.113.10 0.0.0.0/0 tcp dpt:22
```

The `pkts` and `bytes` columns show how much traffic has matched a rule.

**Important:** A zero counter does not necessarily mean the rule is incorrect. It may simply mean that no matching traffic has reached it yet.

It is good practice to inspect the firewall after every configuration change.

## 3. Configure a Default-Deny Policy

Once the required access rules have been configured and tested, the default INPUT policy can be changed:

```bash
iptables -P INPUT DROP
```

This changes the default policy of the INPUT chain to DROP.

Packets that do not match an explicit ACCEPT or other terminating rule will be dropped.

Before applying this policy, make sure that you have accounted for:

- Loopback traffic (`lo`)
- Established and related connections
- SSH access from your trusted IP
- Any other services that must remain reachable

For example, a general connection-tracking rule can be added:

```bash
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

Allow loopback traffic:

```bash
iptables -A INPUT -i lo -j ACCEPT
```

**Warning:** Applying a restrictive policy without testing the rules can lock you out of a remote server. Keep an existing administrative session open and have console access or a recovery mechanism available.

Verify the policy and rules again:

```bash
iptables -nvL
```

## 4. Allowing Web Services

For a legitimate web server that needs to receive HTTP and HTTPS traffic, TCP ports 80 and 443 can be allowed.

Example:

```bash
iptables -A INPUT -i <interface> -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -i <interface> -p tcp --dport 443 -j ACCEPT
```

Here:

- `--dport 80`: Matches HTTP traffic.
- `--dport 443`: Matches HTTPS traffic.
- `-j ACCEPT`: Allows matching traffic.

The `-m tcp` option is unnecessary here because `-p tcp` already selects the TCP protocol.

Opening these ports only permits inbound TCP connections. It does not automatically provide encryption, TLS configuration, or application-layer protection.

## 5. Understanding SSH Port Forwarding

SSH supports several types of port forwarding:

- Local forwarding (`-L`)
- Remote forwarding (`-R`)
- Dynamic forwarding (`-D`)

Remote forwarding is used when a service running on the SSH client needs to become reachable through a listening port on the SSH server.

Its general syntax is:

```bash
ssh -R <remote_bind_address>:<remote_port>:<destination_host>:<destination_port> <user>@<ssh_server>
```

### Command Breakdown

| Component | Description |
|---|---|
| `ssh` | Starts an SSH connection |
| `-R` | Requests remote port forwarding |
| `remote_bind_address` | Address on the SSH server where the forwarded port listens |
| `remote_port` | Port opened on the SSH server |
| `destination_host` | Host address reached from the SSH client side |
| `destination_port` | Destination service port |
| `user@ssh_server` | SSH account and remote server |

### Important SSH Configuration

Remote forwarding is controlled by SSH server settings, including:

- `AllowTcpForwarding`
- `GatewayPorts`

By default, remote forwarded ports are generally bound to loopback on the SSH server. Exposing them on external interfaces requires appropriate configuration.

### Why Root May Be Required

On Linux, binding to privileged ports (ports below 1024) normally requires elevated privileges.

For example, a process binding to port 80 may need additional privileges or an appropriate capability.

However, logging in as root over SSH is not automatically required for every forwarding scenario. Prefer a dedicated account with the minimum necessary permissions.

## 6. Verify Listening Ports

Use:

```bash
ss -lntp
```

Useful options:

- `-l`: Show listening sockets.
- `-n`: Display numerical addresses and ports.
- `-t`: Show TCP sockets.
- `-p`: Show the associated process, when permissions allow.

For UDP sockets as well:

```bash
ss -lntup
```

Remember that a listening socket does not necessarily mean the service is reachable externally. Firewall rules, routing, NAT, and the address on which the service is bound all affect reachability.

## Sliver

[Sliver](https://github.com/BishopFox/sliver) is an open-source Command and Control (C2) framework designed for authorized security assessments and Red Team operations.

It provides an infrastructure for managing agents, communication channels, and operator interactions.

### 1. Sliver Server

Sliver uses a client-server architecture.

- **Sliver Server:** The central component responsible for managing the C2 infrastructure and connected sessions.
- **Sliver Client:** The operator interface used to interact with the server.
- **Implant:** The agent running on an endpoint and communicating with the C2 infrastructure.
- **Listener:** A network service that accepts incoming connections from implants.
- **Session:** An active connection established by an implant.
- **Redirector:** An intermediary host that relays network traffic between an endpoint and the C2 infrastructure.

The server binary can be launched from a Linux terminal:

```bash
sudo ./sliver-server_linux-amd64
```

Use the following commands inside the Sliver console to inspect the available functionality:

```text
help
version
info
```

### 2. Communication Protocols

Sliver supports multiple communication protocols. The choice of protocol affects how implants communicate with the server.

#### HTTP

HTTP is an application-layer protocol used to transfer data between clients and servers.

#### HTTPS

HTTPS uses TLS to encrypt HTTP communication between endpoints.

Encryption protects the contents of network traffic from passive inspection, but it does not make the communication inherently undetectable.

#### Multiplayer

Multiplayer functionality allows multiple operators to work with the same Sliver infrastructure.

This is useful when several team members need access to a shared C2 environment.

### 3. Implant Generation

An implant is the endpoint-side component responsible for communicating with the Sliver infrastructure.

Sliver supports different communication profiles and transport mechanisms. The selected transport determines how the implant communicates with its configured server infrastructure.

An important distinction:

- The **server** manages the infrastructure.
- The **implant** runs on the endpoint.
- The **client** provides the operator interface.

Implant generation and deployment should only be performed in an isolated, explicitly authorized lab.

### 4. Endpoint Security Considerations

Security products such as Microsoft Defender may identify known Sliver binaries through signatures or behavioral detection.

For controlled testing, do not disable endpoint protection on a normal workstation or production system.

Instead, use a disposable Windows virtual machine with:

- An isolated virtual network.
- No access to production systems.
- A snapshot that can be restored.
- Controlled network access.
- Endpoint protection enabled wherever possible.

This allows detection behavior to be observed without weakening the security of the host environment.

### 5. Understanding Sessions

A session represents an active connection between an implant and the Sliver infrastructure.

When a connection is established, the framework can report the new session to the operator.

The Sliver console provides session-management functionality, including:

- Listing active sessions.
- Inspecting session metadata.
- Monitoring connection status.
- Identifying the endpoint associated with a session.

A session is not the same as a listener: a listener accepts incoming connections, while a session represents an established connection.

### 6. Redirector Architecture

A redirector is an intermediary server used to relay network traffic between the endpoint and the actual C2 infrastructure.

Using a separate redirector can help isolate the C2 server from direct exposure.

A simplified architecture looks like this:

```text
Windows Lab VM
      |
      | Network Communication
      |
      v
  Redirector VPS
      |
      | Forwarded Traffic
      |
      v
  Sliver Server
      |
      v
  Operator Client
```

The redirector should be treated as an exposed infrastructure component and hardened accordingly.

Important considerations include:

- Restricting administrative access.
- Minimizing exposed services.
- Monitoring incoming connections.
- Keeping management interfaces private.
- Maintaining firewall rules and system updates.

### 7. Further Reading

- [Sliver Official Repository](https://github.com/BishopFox/sliver)
- [Sliver Documentation](https://sliver.sh/)
- [Microsoft Defender Documentation](https://learn.microsoft.com/en-us/defender-endpoint/)

---

| ← Previous Topic |
|:---|
| [TCPDump & TCPReplay](TCPDump-TCPReplay.html) |
