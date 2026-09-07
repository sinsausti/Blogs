---
title: "Getting Started with Wireshark"
description: "Install and use Wireshark for capturing and analyzing network traffic. Covers display filters, stream following, statistics, and production tips."
author: "Sebastian Insausti"
date: "2025-09-21"
tags: ["Linux", "Monitoring", "Networking"]
canonical_url: "https://insaustis.com/blog/wireshark-getting-started.html"
---

# Getting Started with Wireshark

If tcpdump is a pocket knife, Wireshark is the full toolkit. It's a GUI-based packet analyzer that lets you capture, filter, and deeply inspect network traffic with a level of visibility that's hard to match from the command line alone. It's the standard tool for network troubleshooting, protocol analysis, and security investigations.

This guide will get you from installation to productive analysis quickly.

## Installation

### Linux

```
# Debian / Ubuntu
apt install wireshark

# RHEL / AlmaLinux / Rocky
dnf install wireshark

# During installation, choose "Yes" to allow non-root users to capture.
# Then add your user to the wireshark group:
usermod -aG wireshark $USER
# Log out and back in for the group change to take effect.
```

### macOS

```
brew install --cask wireshark
```

### Windows

Download the signed installer from [wireshark.org](https://www.wireshark.org/download.html). It can install Npcap for live packet capture; WinPcap is obsolete and unsupported.

## The Interface at a Glance

When you open Wireshark, you'll see three main areas once a capture is running:

- **Packet List** (top) — A row per packet, with time, source, destination, protocol, and summary.
- **Packet Details** (middle) — A tree view of all protocol layers for the selected packet.
- **Packet Bytes** (bottom) — The raw hex and ASCII of the packet.

Click any packet in the list to inspect it in the detail and bytes panels.

## Starting a Capture

On the welcome screen, double-click an interface to start capturing immediately, or go to **Capture → Start**. To capture on a server without a GUI, capture with tcpdump to a `.pcap` file and open it in Wireshark on your workstation:

```
# On the server:
tcpdump -i eth0 -n -w /tmp/capture.pcap -c 5000

# Transfer to your machine:
scp user@server:/tmp/capture.pcap ./

# Open in Wireshark:
wireshark capture.pcap
```

This is the most common workflow for server-side analysis — capture headlessly, analyze with the GUI.

## Display Filters

Wireshark has two filter systems: *capture filters* (BPF syntax, same as tcpdump, applied before capture) and *display filters* (Wireshark's own syntax, applied to already-captured packets). Display filters are far more powerful.

```
# Show only HTTP traffic
http

# Show traffic to/from a specific IP
ip.addr == 10.0.1.50

# Show traffic on a specific port
tcp.port == 3306

# Show only TCP RST packets
tcp.flags.reset == 1

# Show only DNS queries (not responses)
dns.flags.response == 0

# Combine with && and ||
ip.addr == 10.0.1.50 && tcp.port == 3306
```

Type filters into the filter bar at the top — Wireshark will autocomplete and color the bar green (valid) or red (invalid) as you type.

## Following a Stream

One of Wireshark's most useful features: right-click any TCP packet and choose **Follow → TCP Stream**. Wireshark reconstructs the full back-and-forth conversation between client and server and displays it in a readable format. This is invaluable for HTTP debugging, seeing what a client is actually sending, or spotting authentication errors in plain-text protocols.

## Useful Built-in Statistics

Under the **Statistics** menu:

- **Conversations** — See all TCP/UDP sessions, sortable by bytes transferred. Great for spotting unexpected talkers.
- **Protocol Hierarchy** — A breakdown of which protocols make up your capture by packet count and bytes.
- **I/O Graph** — Visualize traffic volume over time. Useful for spotting bursts or drops.
- **TCP Stream Graphs → Time-Sequence (tcptrace)** — Diagnose retransmissions, window scaling issues, and latency.

## Coloring Rules

By default, Wireshark colors packets by protocol (light blue for TCP, green for HTTP, dark red for TCP errors/bad checksums, etc.; UDP has no dedicated default color). You can customize these under **View → Coloring Rules** — for example, highlighting all traffic from a specific IP in yellow during an investigation.

## Common Analysis Workflows

### Is there packet loss?

```
# Display filter for retransmissions
tcp.analysis.retransmission

# Or for all TCP issues
tcp.analysis.flags
```

### Slow application responses

Use **Statistics → Service Response Time** to measure how long servers take to respond. For HTTP, **Statistics → HTTP → Requests** shows per-request counts and response codes.

### Who is talking to whom?

**Statistics → Conversations**, sorted by bytes, gives you a quick picture of the top talkers in any capture.

## Tips for Production Use

- Always use a **ring buffer** when capturing long-running traces: `tcpdump -w /tmp/cap-%H%M.pcap -G 300 -C 100` rotates files every 5 minutes or 100MB.
- Set a **capture filter** before starting to avoid capturing irrelevant traffic and growing your pcap unnecessarily.
- Be aware of **privacy**: pcap files contain raw packet data, including credentials over unencrypted protocols. Handle them accordingly.

Wireshark has a steep learning curve, but once you're comfortable with display filters and stream following, it becomes one of the most powerful diagnostic tools in your arsenal. Pair it with tcpdump for server-side captures and you have a complete packet analysis workflow for any environment.

---

Got a Wireshark workflow you swear by? I'd love to hear it — [get in touch.](https://insaustis.com/#contact)
