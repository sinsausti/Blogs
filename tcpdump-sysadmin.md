---
title: "tcpdump: A Sysadmin's Pocket Knife"
description: "A practical guide to tcpdump for sysadmins — installation, essential filters, saving captures, and real-world troubleshooting scenarios."
author: "Sebastian Insausti"
date: "2025-09-21"
tags: ["Linux", "Monitoring", "Networking"]
canonical_url: "https://insaustis.com/blog/tcpdump-sysadmin.html"
---

# tcpdump: A Sysadmin's Pocket Knife

When something is wrong with your network and you need answers fast, `tcpdump` is your best friend. Unlike GUI tools that require a desktop environment, tcpdump runs entirely in the terminal — making it the go-to tool for diagnosing network issues on headless servers, inside containers, and over SSH sessions where you have no alternatives.

This guide covers practical capture patterns for troubleshooting production systems.

## Installation

tcpdump is available in every major Linux distribution:

```
# Debian / Ubuntu
apt install tcpdump

# RHEL / CentOS / AlmaLinux
dnf install tcpdump

# Arch
pacman -S tcpdump
```

You'll need root privileges (or the `cap_net_raw` capability) to capture traffic.

## Basic Usage

The simplest invocation captures everything on all interfaces:

```
tcpdump -i any
```

That's usually too noisy. Use `-i` to specify an interface and `-n` to skip DNS lookups (crucial in production — you don't want DNS queries polluting your capture or adding latency):

```
tcpdump -i eth0 -n
```

## The Filters You'll Use Every Day

### Filter by host

```
# Traffic to or from a specific IP
tcpdump -i eth0 -n host 10.0.1.50

# Only traffic coming FROM that host
tcpdump -i eth0 -n src host 10.0.1.50

# Only traffic going TO that host
tcpdump -i eth0 -n dst host 10.0.1.50
```

### Filter by port

```
# MySQL traffic
tcpdump -i eth0 -n port 3306

# HTTPS
tcpdump -i eth0 -n port 443

# Port range
tcpdump -i eth0 -n portrange 8000-9000
```

### Filter by protocol

```
tcpdump -i eth0 -n tcp
tcpdump -i eth0 -n udp
tcpdump -i eth0 -n icmp
```

### Combining filters

Use `and`, `or`, and `not` to build precise filters:

```
# MySQL traffic from a specific app server
tcpdump -i eth0 -n host 10.0.1.50 and port 3306

# All traffic except SSH (so you don't flood your terminal)
tcpdump -i eth0 -n not port 22
```

## Saving Captures to a File

For deep analysis in Wireshark or to share with a colleague, save the raw packets to a `.pcap` file:

```
tcpdump -i eth0 -n -w /tmp/capture.pcap

# Later, read it back:
tcpdump -r /tmp/capture.pcap
```

Combine with a filter and a packet count so captures don't grow unbounded:

```
# Capture 1000 packets on port 3306, save to file
tcpdump -i eth0 -n port 3306 -c 1000 -w /tmp/mysql.pcap
```

## Reading Packet Contents

By default, tcpdump shows packet headers only. To see content:

```
# ASCII output (good for HTTP, plain-text protocols)
tcpdump -i eth0 -n -A port 80

# Hex + ASCII (for binary protocols)
tcpdump -i eth0 -n -X port 3306
```

> Be careful with -A and -X in production — you may capture sensitive data including credentials in plain-text protocols.

## Real-World Scenarios

### Is my app actually reaching the database?

```
tcpdump -i any -n host 10.0.1.100 and port 3306
```

Run this on the database server. SYN packets without a SYN-ACK narrow the problem to the server or return path: check the listening socket, host firewall, service health, and routing. If no packets arrive, verify name resolution, client routing, upstream firewalls, and that you are capturing on the correct interface.

### Diagnosing connection resets

```
tcpdump -i eth0 -n 'tcp[tcpflags] & tcp-rst != 0'
```

This captures only packets with the RST flag set — invaluable for diagnosing sudden connection drops.

### Check if a host is reachable (beyond ping)

```
tcpdump -i eth0 -n icmp and host 10.0.1.200
```

## Quick Reference

- `-i <iface>` — Interface to listen on (`any` for all)
- `-n` — Don't resolve hostnames or port names
- `-c <n>` — Stop after capturing n packets
- `-w <file>` — Write to pcap file
- `-r <file>` — Read from pcap file
- `-A` — Print packet payload as ASCII
- `-X` — Print payload in hex and ASCII
- `-v / -vv / -vvv` — Increase verbosity

tcpdump is one of those tools that rewards investment. The more comfortable you are with its filter syntax (based on BPF — Berkeley Packet Filter), the faster you can isolate network problems under pressure. Keep it in your toolkit.

---

Questions or a tcpdump pattern you rely on that I missed? [Drop me a message.](https://insaustis.com/#contact)
