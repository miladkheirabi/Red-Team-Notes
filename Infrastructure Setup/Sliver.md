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

## 7. Sliver Architecture: Conceptual Overview

Sliver separates several responsibilities:

- **Server:** Runs the C2 infrastructure and manages operator interactions.
- **Client:** Provides the operator interface to the server.
- **Listener:** Represents a network endpoint used by the framework.
- **Redirector:** An intermediary network host that relays traffic.
- **Session:** Represents an active connection managed by the framework.

For an authorized lab, keep management interfaces isolated, use dedicated virtual networks, and monitor both the redirector and the endpoint. Avoid exposing management services to the public Internet.
