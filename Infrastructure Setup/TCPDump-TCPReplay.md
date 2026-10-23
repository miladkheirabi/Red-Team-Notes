# TCPDump & TCPReplay

## TCPDump

`tcpdump` captures and filters network traffic from an interface.

Basic syntax:

```bash
tcpdump [options] [filter]
```

---

## Capture Traffic to a PCAP

```bash
tcpdump -i eth0 -XX -w <OUTPUT_PATH>.pcap
```

Breakdown:

```text
-i eth0       → Capture traffic on eth0
-XX           → Print packet data in hex and ASCII
-w <file>     → Write captured packets to a PCAP file
```

Example:

```bash
tcpdump -i eth0 -XX -w capture.pcap
```

When `-w` is used, the packets are written to the PCAP file instead of being displayed normally.

---

# TCPDump Filters

Filters use the Berkeley Packet Filter (BPF) syntax.

## Filter by Protocol and Port

```bash
tcpdump tcp port 80 and dst 2.2.2.2
```

Meaning:

```text
tcp       → Only TCP traffic
port 80   → Source or destination port 80
and       → Both conditions must match
dst       → Destination address
2.2.2.2   → Destination IP
```

This captures TCP traffic involving port `80` where the destination is `2.2.2.2`.

---

## Filter by Interface, Destination and Port

```bash
tcpdump -i eth0 -tttt dst 192.168.1.22 and dst port 22
```

Breakdown:

```text
-i eth0          → Capture on eth0
-tttt            → Human-readable full timestamp
dst 192.168.1.22 → Destination IP
dst port 22      → Destination TCP/UDP port 22
```

This captures traffic going to:

```text
192.168.1.22:22
```

---

## Display Packet Contents

```bash
tcpdump -nSX port 443
```

Options:

```text
-n  → Do not resolve IP addresses to hostnames
-S  → Print absolute TCP sequence numbers
-X  → Display packet contents in hex and ASCII
```

The filter:

```text
port 443
```

matches traffic where either the source or destination port is `443`.

---

# Source and Destination Filters

## Source IP

```bash
tcpdump src 1.1.1.1
```

Captures packets originating from:

```text
1.1.1.1
```

---

## Destination IP

```bash
tcpdump dst 1.0.0.1
```

Captures packets whose destination is:

```text
1.0.0.1
```

---

# Combining Filters

BPF filters can be combined using:

```text
and
or
not
```

Examples:

```bash
tcpdump src 10.10.10.10 and dst 10.10.10.20
```

```bash
tcpdump port 80 or port 443
```

```bash
tcpdump not port 22
```

Parentheses can be used to group conditions:

```bash
tcpdump 'host 10.10.10.10 and (port 80 or port 443)'
```

---

# Searching Captured Traffic for Cleartext Credentials

For protocols that transmit credentials in plaintext, packet contents can sometimes be searched directly.

Example:

```bash
tcpdump port http or port ftp or port smtp or port imap or port pop3 or port telnet -lA |
egrep -i -B 5 'pass=|pwd=|log=|login=|user=|username=|pw=|passw=|passwd=|password=|pass:|user:|username:|password:|login:|pass|user'
```

Important options:

```text
-l  → Line-buffered output
-A  → Print packet contents as ASCII
```

`egrep`/`grep` is then used to search the ASCII output for common credential-related strings.

`-B 5` displays five lines before a matching line, which can provide additional context.

### Important

This technique only works when the relevant application data is visible to `tcpdump`.

For encrypted protocols such as HTTPS or SSH, credentials are not normally available as plaintext packet contents.

---

# Useful TCPDump Filters

### Host

```bash
tcpdump host 10.10.10.10
```

Traffic to or from a host.

### Source host

```bash
tcpdump src host 10.10.10.10
```

### Destination host

```bash
tcpdump dst host 10.10.10.10
```

### Port

```bash
tcpdump port 443
```

### Source port

```bash
tcpdump src port 443
```

### Destination port

```bash
tcpdump dst port 443
```

### Protocol

```bash
tcpdump tcp
tcpdump udp
tcpdump icmp
```

### Combine conditions

```bash
tcpdump 'src host 10.10.10.10 and dst port 443'
```

---

# TCPReplay

`tcpreplay` replays packets from a PCAP file through a network interface.

Basic syntax:

```bash
tcpreplay -i <interface> <input.pcap>
```

---

## Basic Replay

```bash
tcpreplay -i eth0 <INPUT_PATH>.pcap
```

Breakdown:

```text
-i eth0            → Send packets through eth0
<INPUT_PATH>.pcap  → PCAP file to replay
```

---

## Replay at Maximum Speed

```bash
tcpreplay --topspeed -i eth0 <INPUT_PATH>.pcap
```

`--topspeed` attempts to replay the traffic as fast as possible, subject to the capabilities of the system and interface.

Useful when testing how a network device or monitoring system handles high packet rates.

---

## Replay One Packet at a Time

```bash
tcpreplay --oneatatime --verbose -i eth0 <INPUT_PATH>.pcap
```

Options:

```text
--oneatatime  → Replay one packet at a time
--verbose     → Display additional information during replay
```

Useful when manually inspecting or troubleshooting a replay.

---

## Replay Multiple Times

```bash
tcpreplay --loop=10 -i eth0 <INPUT_PATH>.pcap
```

Replays the PCAP 10 times.

```text
--loop=10 → Repeat the capture 10 times
```

---

## Infinite Replay

```bash
tcpreplay --loop=0 -i eth0 <INPUT_PATH>.pcap
```

A loop count of `0` means replay indefinitely.

Stop it with:

```text
Ctrl+C
```

---

# Quick Reference

## TCPDump

```bash
# Capture to PCAP
tcpdump -i eth0 -w capture.pcap

# TCP traffic to a destination
tcpdump tcp port 80 and dst 2.2.2.2

# Destination IP and port
tcpdump -i eth0 -tttt dst 192.168.1.22 and dst port 22

# Display packet contents
tcpdump -nSX port 443

# Source
tcpdump src 1.1.1.1

# Destination
tcpdump dst 1.0.0.1

# Host + port
tcpdump 'host 10.10.10.10 and port 443'
```

## TCPReplay

```bash
# Basic replay
tcpreplay -i eth0 capture.pcap

# Maximum speed
tcpreplay --topspeed -i eth0 capture.pcap

# One packet at a time
tcpreplay --oneatatime --verbose -i eth0 capture.pcap

# Replay 10 times
tcpreplay --loop=10 -i eth0 capture.pcap

# Infinite replay
tcpreplay --loop=0 -i eth0 capture.pcap
```
