# SSH Key Setup

## 1. Generate an SSH Key Pair

Generate a key pair and explicitly define where it should be stored:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/<key_name>
```

When prompted, **set a passphrase** for the private key.

This creates:

```text
~/.ssh/<key_name>       # Private key
~/.ssh/<key_name>.pub   # Public key
```

Never share the private key.

---

## 2. Connect to the Target Using a Password

If key-based authentication has not been configured yet:

```bash
ssh <username>@<target_ip>
```

Enter the user's password when prompted.

---

## 3. Create the `.ssh` Directory

On the target, create the SSH configuration directory if it does not already exist:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
```

---

## 4. Add the Public Key

Append your public key to:

```text
~/.ssh/authorized_keys
```

For example:

```bash
cat >> ~/.ssh/authorized_keys
```

Then paste the contents of your local:

```text
~/.ssh/<key_name>.pub
```

Alternatively, from your local machine:

```bash
ssh-copy-id -i ~/.ssh/<key_name>.pub <username>@<target_ip>
```

### OPSEC: Remove the Key Comment

SSH public keys commonly end with a comment such as:

```text
ssh-ed25519 AAAA... user@hostname
```

Remove the comment:

```text
user@hostname
```

The comment can reveal the identity or hostname associated with the key that generated it.

The resulting key should look like:

```text
ssh-ed25519 AAAA...
```

---

## 5. Set `authorized_keys` Permissions

On the target:

```bash
chmod 600 ~/.ssh/authorized_keys
```

Recommended permissions:

```text
~/.ssh/              → 700
~/.ssh/authorized_keys → 600
```

---

## 6. Login Using the SSH Key

From your local machine:

```bash
ssh -i ~/.ssh/<key_name> <username>@<target_ip>
```

If the key has a passphrase, SSH will prompt for it.

### Quick Reference

```bash
# Generate key
ssh-keygen -t ed25519 -f ~/.ssh/<key_name>

# Connect with password
ssh <username>@<target_ip>

# Target: create SSH directory
mkdir -p ~/.ssh
chmod 700 ~/.ssh

# Target: add public key
cat >> ~/.ssh/authorized_keys

# Target: secure authorized_keys
chmod 600 ~/.ssh/authorized_keys

# Login with key
ssh -i ~/.ssh/<key_name> <username>@<target_ip>
```
