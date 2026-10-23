# SSH

## Known Hosts

SSH stores the host keys of previously contacted servers in a `known_hosts` file.

### User-Level

```text
~/.ssh/known_hosts
```

### System-Wide

```text
/etc/ssh/ssh_known_hosts
```

The exact location can vary depending on the SSH configuration and operating system.

---

## Unknown Host Fingerprint

When connecting to a host for the first time, SSH may display a message similar to:

```text
The authenticity of host '10.10.10.10' can't be established.
ED25519 key fingerprint is SHA256:...
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

This means SSH does not have a previously trusted host key for this host.

It is **not automatically an indication of compromise**. Common legitimate reasons include:

- First connection to the host
- The host was reinstalled
- SSH host keys were regenerated
- The IP address now points to a different system

### Why It Matters

The fingerprint is used to verify that you are connecting to the expected SSH server.

Ignoring the warning and blindly accepting an unknown or changed fingerprint can allow a **Man-in-the-Middle (MITM)** attack:

```text
Client
  ↓
Attacker
  ↓
Real SSH Server
```

If an attacker can intercept the connection and present their own host key, blindly accepting it can result in establishing an SSH connection with the attacker's machine instead of the intended server.

### Verify Before Accepting

For a new or changed host key, verify the fingerprint through a trusted channel before accepting it.

If the fingerprint unexpectedly changes for a previously known host, investigate it instead of blindly accepting the new key.

---

## SCP

`scp` copies files between systems over SSH.

### Local → Remote

```bash
scp <local_file> <username>@<target_ip>:/path/to/destination/
```

Example:

```bash
scp payload.txt user@10.10.10.10:/tmp/
```

### Remote → Local

```bash
scp <username>@<target_ip>:/path/to/remote_file <local_destination>
```

Example:

```bash
scp user@10.10.10.10:/tmp/results.txt .
```

### Directory Transfer

Use `-r` to recursively copy a directory:

```bash
scp -r <local_directory> <username>@<target_ip>:/path/to/destination/
```

Or:

```bash
scp -r <username>@<target_ip>:/path/to/remote_directory .
```

### Using a Specific SSH Key

```bash
scp -i ~/.ssh/<key_name> <local_file> <username>@<target_ip>:/tmp/
```

`scp` uses SSH for authentication and transport, so the same SSH keys, host-key verification, and SSH configuration apply.
