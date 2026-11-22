# Chapter 9: Network Analysis

**Cybersecurity Guide Book | Roadmap step 9 of 12** | Author: [ashihh07](https://github.com/ashihh07)

[Back to roadmap](../README.md) | [Previous: Chapter 8 Penetration Testing](../08-penetration-testing/README.md) | [Download the PDF version](Chapter-09-Network-Analysis.pdf)

> Every chapter so far has mentioned traffic without truly looking at it. This chapter opens the packet. If you can read what is actually on the wire, you can confirm or disprove almost any alert.

---

## About this chapter

Chapters 6 and 8 generated traffic: scans, exploits, shells, exfiltration. Chapter 4 and 5 generated HTTP requests. All of it travelled as packets. **Network analysis** is the skill of capturing that traffic and reading it well enough to answer: what really happened, in what order, and is it normal?

The roadmap lists four items:

- **Wireshark:** the standard tool for capturing and inspecting packets.
- **Packet analysis:** reading protocols from Chapter 1 as they actually appear on the wire.
- **Network traffic:** recognising normal patterns so abnormal ones stand out.
- **IDS/IPS basics:** how automated tools watch traffic for you, at scale.

### How each topic is built

- **Concept**: what it is and why it exists.
- **Attacker view**: what shows up in a capture when this chapter's earlier attacks happen.
- **Defender view**: how to find it, and how detection rules work.
- **Labs**: capture your own lab traffic and read it.

### Contents

| # | Section | Roadmap item covered |
|---|---------|----------------------|
| 1 | [Capturing traffic](#1-capturing-traffic) | Wireshark |
| 2 | [Wireshark: filters and navigation](#2-wireshark-filters-and-navigation) | Wireshark |
| 3 | [Packet analysis by protocol](#3-packet-analysis-by-protocol) | Packet Analysis |
| 4 | [Reading traffic patterns](#4-reading-traffic-patterns) | Network Traffic |
| 5 | [Finding this book's attacks in a capture](#5-finding-this-books-attacks-in-a-capture) | Packet Analysis, Network Traffic |
| 6 | [IDS and IPS basics](#6-ids-and-ips-basics) | IDS/IPS Basics |
| 7 | [Hands-on labs](#7-hands-on-labs) | Practice for all of the above |
| 8 | [Quick revision sheet](#8-quick-revision-sheet) | Tables and commands |
| 9 | [Self-check quiz and answers](#9-self-check-quiz) | Test yourself |

> [!WARNING]
> **Ground rule for every lab in this book**
> - Capture traffic only on networks and devices you own or have written permission to monitor. Capturing someone else's traffic without consent, even "just to learn", can be illegal (wiretapping and interception laws) and is a serious trust violation.
> - Practise inside your host-only lab network from earlier chapters.
> - A `.pcap` file can contain real passwords, cookies and personal data (yours or someone else's). Never commit a real capture to a public repository; use the sanitised or purpose-built captures named in this chapter's labs instead.

---

## 1. Capturing traffic

### Concept

A **packet capture (pcap)** is a recording of frames as they cross a network interface. **Wireshark** is the standard graphical tool for reading them, built on the same capture engine (`libpcap`/`Npcap`) as the command-line tools `tcpdump` (Linux, used since Chapter 1) and `dumpcap`.

### Where you can capture

| Point | Sees |
|-------|------|
| **On the host itself** | Only that host's own traffic |
| **A switch port with port mirroring (SPAN)** | A copy of traffic to and from chosen ports |
| **A network tap** | A full, passive copy of a link, including errors |
| **A router or gateway** | Everything crossing that boundary |
| **Promiscuous mode** | Lets a network card read frames not addressed to it, when the capture point allows it (mirrored or hubbed traffic) |

On a modern **switched** network, a host normally only sees its own unicast traffic, broadcasts and multicasts. That is why SPAN ports and taps exist: without one, a laptop plugged into a random switch port cannot simply "sniff everything."

### Capturing in Wireshark

```text
1. Open Wireshark -> the start screen lists available interfaces with live traffic sparklines
2. Double-click an interface to start capturing (may need sudo/administrator rights)
3. Generate some traffic (browse, ping, ssh) in another window
4. Red square (or Ctrl+E) to stop the capture
5. File -> Save As -> name.pcapng to keep it
```

### Capturing from the command line

```bash
tcpdump -D                                    # list interfaces
sudo tcpdump -i eth0 -w capture.pcap          # capture everything on eth0 to a file
sudo tcpdump -i eth0 -w capture.pcap port 80  # only port 80 traffic
sudo tcpdump -i eth0 -c 500 -w capture.pcap   # stop after 500 packets
```

A **capture filter** (used with `tcpdump`, or in Wireshark's capture options) decides what gets written to disk at all, using Berkeley Packet Filter (BPF) syntax: `host 192.168.56.101`, `port 443`, `tcp and port 80`. Capture filters are coarse and set before capturing. **Display filters** (next section) are far more expressive and are applied after capture, so when in doubt, capture broadly and filter narrowly while reading.

> [!CAUTION]
> **Attacker view**
> - A compromised host or a rogue device plugged into an unprotected switch port can run its own capture and see more than it should, especially on an unswitched or carelessly mirrored segment.
> - ARP spoofing (Chapter 1) is often used specifically to force traffic through an attacker's machine so it can be captured.

> [!TIP]
> **Defender view**
> - Capture **at chokepoints**: internet edge, between network segments, in front of critical servers. You cannot analyse what you never captured.
> - Full packet capture is expensive to store at scale. Many SOCs capture metadata (NetFlow, Zeek logs) everywhere and full packets only at a few key points or during an active investigation.
> - Treat capture files as sensitive data: they often contain credentials and personal information.

---

## 2. Wireshark: filters and navigation

### Concept

Wireshark's three panes are the whole interface: the **packet list** (one line per packet), the **packet detail** (the selected packet's layers, expandable), and the **bytes** pane (the raw hex and ASCII).

### Display filters

Display filters use **field names**, not the BPF syntax from capture filters.

| Filter | Shows |
|--------|-------|
| `ip.addr == 192.168.56.101` | Any packet to or from this IP |
| `ip.src == 10.0.0.5` | Only packets **from** this source |
| `tcp.port == 443` | Any TCP traffic on port 443 |
| `http` | HTTP requests and responses |
| `http.request.method == "POST"` | Only HTTP POST requests |
| `dns` | DNS queries and answers |
| `dns.qry.name contains "example"` | DNS lookups containing this text |
| `tcp.flags.syn == 1 && tcp.flags.ack == 0` | SYN packets only (the first step of a handshake, Chapter 1) |
| `tcp.analysis.retransmission` | Packets Wireshark flags as retransmitted |
| `tls.handshake.type == 1` | TLS ClientHello packets |
| `frame contains "password"` | Any packet with this text anywhere in it (plain-text protocols only) |
| `!(arp or icmp)` | Hide ARP and ICMP to reduce noise |

Right-click any field in the packet detail pane and choose **Apply as Filter** to build filters without memorising syntax.

### Navigation and analysis tools

| Feature | Menu | Use |
|---------|------|-----|
| **Follow TCP Stream** | right-click a packet | Reassembles one conversation into readable text, in order, both directions |
| **Statistics > Conversations** | Statistics menu | Who talked to whom, how much data, for how long |
| **Statistics > Protocol Hierarchy** | Statistics menu | What protocols make up this capture, by percentage |
| **Statistics > Endpoints** | Statistics menu | Every unique address seen, with traffic totals |
| **Export Objects > HTTP** | File menu | Pull out files (images, downloads) transferred over plain HTTP |
| **Expert Information** | Analyze menu | Wireshark's own flagged warnings: retransmissions, errors, malformed packets |
| **Coloring Rules** | View menu | Why a row is a certain colour (default: black on red usually means an error) |

```bash
# tshark: Wireshark's command-line sibling, for scripting (Chapter 3 territory)
tshark -r capture.pcap -Y "http.request" -T fields -e ip.src -e http.host -e http.request.uri
```

> [!CAUTION]
> **Attacker view**
> - "Follow TCP Stream" on an **unencrypted** capture can reveal full HTTP bodies, FTP and Telnet credentials (Chapter 1), and any other plain-text protocol content, in one click.
> - This is exactly why Chapter 1's "TLS everywhere" and Chapter 4's "never send passwords over GET or plain HTTP" matter: once someone else can capture the traffic, plain text is already lost.

> [!TIP]
> **Defender view**
> - Protocol Hierarchy is a fast first look: a capture that is "supposed to be" mostly HTTPS but shows a slice of plain Telnet or FTP is worth investigating immediately.
> - Export Objects over HTTP is a quick way to recover a file an attacker downloaded or exfiltrated in a capture, during an investigation.

---

## 3. Packet analysis by protocol

### Concept

This section re-reads Chapter 1's protocols, this time as they actually look inside Wireshark.

### Ethernet and IP headers (the bottom layers)

Expanding a packet's detail pane shows nested headers matching Chapter 1's encapsulation diagram exactly: **Frame > Ethernet II > Internet Protocol Version 4 > Transmission Control Protocol > [Application data]**. Useful fields: `ip.ttl` (hops remaining, and a loose OS hint — Linux often starts at 64, Windows at 128), `ip.flags.df` (don't fragment), and `eth.src`/`eth.dst` (MAC addresses, Chapter 1's layer 2).

### TCP in a capture

```text
Filter: ip.addr == 192.168.56.101 && tcp.port == 80

No.   Time     Source            Destination       Info
1     0.000    10.0.0.5          192.168.56.101    51514 -> 80 [SYN] Seq=0
2     0.001    192.168.56.101    10.0.0.5          80 -> 51514 [SYN, ACK] Seq=0 Ack=1
3     0.001    10.0.0.5          192.168.56.101    51514 -> 80 [ACK] Seq=1 Ack=1
4     0.002    10.0.0.5          192.168.56.101    GET / HTTP/1.1
5     0.010    192.168.56.101    10.0.0.5          HTTP/1.1 200 OK
```

This is the Chapter 1 three-way handshake made visible, followed by the actual HTTP exchange riding on top of it. The `tcp.analysis` fields Wireshark adds are worth knowing:

| Field | Meaning |
|-------|---------|
| `tcp.analysis.retransmission` | This segment was sent again, usually because the first was lost or unacknowledged |
| `tcp.analysis.duplicate_ack` | The receiver is repeating an ACK, often a sign of packet loss ahead |
| `tcp.analysis.zero_window` | The receiver's buffer is full and it is asking the sender to pause |
| `[RST]` | A connection was abruptly closed or refused (Chapter 1) |

Occasional retransmissions are normal on real networks. A capture **dominated** by them points to a congested link, a failing device, or (rarely) deliberate disruption.

### DNS in a capture

```text
Filter: dns

No.   Info
10    Standard query 0x1a2b A example.com
11    Standard query response 0x1a2b A example.com A 203.0.113.50
```

The query and response share the same transaction ID (`0x1a2b`), which is how you match a question to its answer in a busy capture. A response with a different source address than the resolver you queried, or an unusually short TTL, deserves a second look (Chapter 1's cache poisoning concern).

### HTTP and TLS in a capture

For plain HTTP, Follow TCP Stream shows the entire request and response in Chapter 4's raw text form, headers and body included. For HTTPS, Wireshark shows the TLS handshake (Chapter 1's ClientHello/ServerHello, certificate exchange) but **not** the encrypted application data, unless you have the session keys. In a lab you control, you can capture both sides of the decryption:

```bash
# In a lab browser, set this environment variable BEFORE launching it, to log TLS session keys
export SSLKEYLOGFILE=~/sslkeys.log
```

In Wireshark, set **Preferences > Protocols > TLS > (Pre)-Master-Secret log filename** to that file, and HTTPS payloads in your own lab traffic become readable. This only works because you control both the browser and the key log; it is not a way to break other people's TLS.

> [!CAUTION]
> **Attacker view**
> - On a network where an attacker can capture traffic, every plain-text protocol is fully readable: HTTP bodies, FTP and Telnet logins, unencrypted DNS content.
> - TCP sequence and acknowledgement numbers, visible in any capture, are part of what historically enabled TCP session hijacking and spoofing attacks against poorly randomised stacks.

> [!TIP]
> **Defender view**
> - A sudden shift in `ip.ttl` for packets claiming to be from the same host can indicate spoofing (the real packets take a different path, with a different hop count, than forged ones).
> - A spike in `tcp.analysis.retransmission` or `tcp.analysis.zero_window` from one host often explains a "the app feels slow" complaint better than any application log.

---

## 4. Reading traffic patterns

### Concept

A single packet rarely tells a story. **Patterns across many packets** do, which is why Statistics > Conversations and Protocol Hierarchy matter as much as individual packets.

### What "normal" looks like

| Traffic | Typical shape |
|---------|---------------|
| **Web browsing** | Many short TCP connections, mostly to CDN and ad/tracker IPs, bursty |
| **DNS** | Small UDP packets, roughly matching the pace of new connections being made |
| **File transfer / backup** | One or a few long-lived connections moving a large, steady volume of data |
| **VoIP / video call** | Steady-rate UDP in both directions for the call's duration |
| **A server's normal client base** | A fairly stable, repeating set of IP addresses and ports, each session short |

### What stands out

| Pattern | Possible meaning | Chapter reference |
|---------|-------------------|---------------------|
| **One source, many destination ports, few completed handshakes** | Port scan | Chapters 1, 6 |
| **One source, many destination hosts, same port** | Host sweep / network scan | Chapter 6 |
| **Long-lived outbound connection from an unusual local process** | Reverse shell / C2 beacon | Chapter 8 |
| **Regular, fixed-interval small connections to the same external host** | Beaconing malware or scheduled check-in | Beyond this book; a Chapter 11 topic |
| **A burst of DNS queries for random-looking subdomains** | DNS tunneling, or a domain generation algorithm | Chapter 1 |
| **A sudden, large, one-directional data transfer** | Possible exfiltration | Chapter 8 |
| **Plain FTP/Telnet where only encrypted protocols are expected** | Insecure legacy service, or a downgrade attempt | Chapter 1 |

### Baselining

You cannot spot "unusual" without knowing "usual." A simple baseline, built once with Chapter 3's tools, is often enough:

```bash
tshark -r normal_day.pcap -T fields -e ip.dst | sort | uniq -c | sort -nr > baseline.txt
tshark -r todays_capture.pcap -T fields -e ip.dst | sort | uniq -c | sort -nr > today.txt
diff baseline.txt today.txt | head -40          # new or very different destinations stand out
```

> [!TIP]
> **Defender view**
> - Build a rough baseline for anything you are responsible for monitoring: which hosts normally talk to which, roughly how much, and on what ports. Deviation from that baseline is often the first and only sign of a problem.
> - Statistics > Conversations, sorted by bytes, is the fastest way to answer "what is actually using this link right now?"

---

## 5. Finding this book's attacks in a capture

This section ties the whole book together: what the earlier chapters' techniques actually look like once you open the packet.

| From chapter | Activity | What a capture shows |
|--------------|----------|------------------------|
| **Chapter 1 and 6** | Nmap SYN scan | Many `[SYN]` packets from one source to sequential ports on one host, overwhelmingly answered by `[SYN, ACK]` (open) or `[RST]` (closed), very few full three-way handshakes completed |
| **Chapter 6** | Host sweep | ICMP echo requests, or SYNs to one common port, spread across many sequential destination addresses |
| **Chapter 5** | SQL injection attempt over HTTP | An HTTP request whose URI or POST body (visible via Follow TCP Stream, if unencrypted) contains quotes, `UNION SELECT`, or SQL comment markers |
| **Chapter 5** | Reflected XSS attempt | An HTTP request URI containing `<script`, `%3Cscript`, or `onerror=` |
| **Chapter 6** | DNS zone transfer | A DNS packet with query type `AXFR`, followed by an unusually large response |
| **Chapter 8** | A Meterpreter reverse shell callback | A new outbound TCP connection from an internal host to an external (or lab-external) address, often on an uncommon port, that stays open and exchanges small, irregular bursts of data |
| **Chapter 8** | File exfiltration after compromise | A long, one-directional, high-volume transfer from an internal host outward, outside normal baseline hours or volume |
| **Chapter 2** | A rogue DHCP server (if captured on the segment) | A `DHCP OFFER` arriving from an unexpected source address, or two different servers answering the same `DHCP DISCOVER` |

> [!IMPORTANT]
> A capture proves **what crossed the wire**, which is stronger evidence than a log line alone, but it only covers the segment you captured on, and it tells you nothing about encrypted payload content unless you hold the keys (section 3). Correlate it with the host and application logs from Chapters 2, 4 and 6 for the full picture.

---

## 6. IDS and IPS basics

### Concept

Reading every packet by hand does not scale. An **Intrusion Detection System (IDS)** watches traffic continuously and raises alerts against known patterns. An **Intrusion Prevention System (IPS)** sits inline and can actively block matching traffic, not just alert on it.

| | IDS | IPS |
|---|-----|-----|
| **Position** | Usually out-of-band, watching a copy (SPAN/tap) | Inline, in the direct path of traffic |
| **Action** | Alerts only | Can drop, reject or block matching traffic |
| **Risk if wrong** | A false positive is just noise | A false positive can block real, legitimate traffic |

### Detection approaches

| Approach | How it works | Strength | Weakness |
|----------|---------------|----------|----------|
| **Signature-based** | Matches traffic against known attack patterns (rules) | Low false positives for known threats, explainable | Misses anything new or well-obfuscated |
| **Anomaly-based** | Learns a baseline (section 4) and flags deviation | Can catch unknown attacks | More false positives, needs tuning and a good baseline |

### Well-known tools

| Tool | Type | Notes |
|------|------|-------|
| **Snort** | Signature-based IDS/IPS | One of the oldest and most widely deployed, large community rule sets |
| **Suricata** | Signature-based IDS/IPS | Multi-threaded, also does protocol logging (similar role to Zeek for some use cases) |
| **Zeek** (formerly Bro) | Network security monitor | Produces rich structured logs of connections, DNS, HTTP, files and more, rather than just alerts. Very strong for hunting and the kind of baselining in section 4 |

### A Suricata-style rule, read line by line

```text
alert tcp any any -> $HOME_NET 22 (msg:"Possible SSH brute force"; flow:to_server; threshold:type both, track by_src, count 5, seconds 60; sid:1000001; rev:1;)
```

| Part | Meaning |
|------|---------|
| `alert` | Action: generate an alert (an IPS rule could use `drop` instead) |
| `tcp any any -> $HOME_NET 22` | Protocol and direction: any source, any port, to our network on port 22 |
| `msg:"..."` | The human-readable alert text |
| `flow:to_server` | Only match traffic heading to the server side of the connection |
| `threshold:...count 5, seconds 60` | Only alert after 5 matches from the same source within 60 seconds, which is exactly Chapter 2's "many failed SSH attempts" pattern, built into the rule itself |
| `sid` / `rev` | A unique rule ID and its revision number |

```bash
sudo apt install suricata
sudo suricata -r capture.pcap -l logs/ -c /etc/suricata/suricata.yaml    # analyse an existing pcap
tail -f logs/fast.log                                                    # human-readable alerts
jq . logs/eve.json | less                                                # structured JSON alerts and logs (Chapter 3)
```

### Where IDS/IPS fits with the rest of this book

```text
Packet on the wire -> IDS/IPS inspects it against rules/baseline -> alert (and action, if IPS)
                                                                        |
                                                                        v
                                                        SOC analyst investigates, using:
                                                 this chapter's manual packet analysis,
                                                 Chapter 2/4/6 host and application logs,
                                                 and Chapter 7's context on asset value
```

An IDS/IPS alert is a **lead**, exactly like a vulnerability scanner finding in Chapter 7. It still needs the analyst's judgement and corroborating evidence before it becomes a confirmed incident.

> [!CAUTION]
> **Attacker view**
> - Attackers deliberately slow down and fragment their activity (slow scans, traffic spread across time or source addresses) specifically to stay under threshold-based rules like the one above.
> - Encrypting command-and-control traffic, and mimicking normal-looking HTTPS beaconing, is a direct response to signature and volume-based detection.

> [!TIP]
> **Defender view**
> - Tune thresholds to your real environment. A rule copied unchanged from the internet, with no tuning, usually produces either too much noise or misses your actual traffic patterns.
> - Combine **signature-based** (Snort/Suricata) with **log-based monitoring** (Zeek, or your own Chapter 2/4/6 `grep`/`awk` searches) and a SIEM (Chapter 11) that correlates both. No single layer catches everything.
> - An IDS/IPS alert without the underlying packet or flow data to confirm it is much harder to act on confidently; keep enough retained capture or flow data to go back and check.

---

## 7. Hands-on labs

Capture only inside your own host-only lab network from earlier chapters. Write each lab up: goal, the filter or command used, evidence (a screenshot of the relevant packets), and what you learned.

### Lab 1: Capture and save your first pcap

```bash
sudo tcpdump -i any -w lab1.pcap &
curl http://neverssl.com > /dev/null
dig example.com > /dev/null
kill %1
```

Open `lab1.pcap` in Wireshark.

**Goal:** find the TCP handshake, the HTTP request, and the DNS query and response in the packet list, and explain what each one is.

### Lab 2: Filters and Follow TCP Stream

Using the same capture (or a fresh one of `curl http://neverssl.com`), apply `http.request`, then right-click the request and choose **Follow > TCP Stream**.

**Goal:** read the full plain-text HTTP request and response in one window, and identify the `Host` and `User-Agent` headers from Chapter 4 inside it.

### Lab 3: Capture an Nmap scan from both sides

On the attacker VM: `sudo tcpdump -i eth0 -w scan.pcap &`, then run `sudo nmap -sS -p 1-100 <target-ip>` (Chapter 6). On the target, you can also capture with `sudo tcpdump -i eth0 -w scan_target.pcap &`.

**Goal:** in Wireshark, apply `tcp.flags.syn==1 && tcp.flags.ack==0`, count the SYNs, then use Statistics > Conversations to show one source talking to many destination ports on one host. Describe the pattern in your own words.

### Lab 4: Capture and recognise a Chapter 5 SQL injection attempt

Run your Chapter 5 `xss_server.py` or SQL injection lab while capturing with `tcpdump` on loopback (`sudo tcpdump -i lo -w sqli.pcap`), then send `curl "http://127.0.0.1:8000/?q=1%27%20OR%20%271%27%3D%271"`.

**Goal:** find the request in Wireshark with a filter such as `http.request.uri contains "OR"`, and explain why this would be invisible if the server used HTTPS and you had not set `SSLKEYLOGFILE`.

### Lab 5: Capture a Meterpreter callback

Repeat Chapter 8's Lab 3 (msfvenom reverse shell and handler) while running `sudo tcpdump -i eth0 -w meterpreter.pcap` on either VM.

**Goal:** find the new outbound connection in the capture, note its destination port, and describe (without needing to decrypt anything) what about this connection's pattern would draw a defender's attention compared to normal traffic on that host.

### Lab 6: Baseline a short capture

```bash
sudo tcpdump -i any -w normal.pcap -G 120 -W 1     # capture 2 minutes of your own normal activity
tshark -r normal.pcap -T fields -e ip.dst | sort | uniq -c | sort -nr > baseline.txt
```

Then generate the Lab 3 scan traffic again, capture it separately, and build `today.txt` the same way.

**Goal:** a `diff` between the two showing the scan's destinations standing out against your normal baseline, in your own write-up.

### Lab 7: Run Suricata against a pcap

```bash
sudo apt install suricata
sudo suricata -r scan.pcap -l suricata_logs/
cat suricata_logs/fast.log
```

**Goal:** confirm whether Suricata's default rules flagged your Chapter 6 Nmap scan, and if not, write the custom rule (using the example in [section 6](#6-ids-and-ips-basics) as a template) that would.

### Lab 8: Export a file from a capture

While capturing, download any small file over plain HTTP in your lab (for example from your Chapter 4 `python3 -m http.server`), then in Wireshark use **File > Export Objects > HTTP**.

**Goal:** recover the exact file from the capture alone, and explain in one sentence why this would not work if the download had used HTTPS without the session keys.

> [!NOTE]
> **Next step on TryHackMe:** work through a Wireshark fundamentals room and a Snort or Suricata room, and write each up with your own filters and findings, not just the walkthrough's.

---

## 8. Quick revision sheet

### Display filters

| Filter | Shows |
|--------|-------|
| `ip.addr == x.x.x.x` | To or from this IP |
| `tcp.port == 443` | TCP traffic on this port |
| `http.request.method == "POST"` | HTTP POST requests |
| `dns.qry.name contains "x"` | DNS queries matching text |
| `tcp.flags.syn==1 && tcp.flags.ack==0` | SYN-only packets (scan signature) |
| `tcp.analysis.retransmission` | Retransmitted segments |
| `tls.handshake.type == 1` | TLS ClientHello |

### Commands

| Task | Command |
|------|---------|
| **Capture to a file** | `sudo tcpdump -i eth0 -w file.pcap` |
| **Capture with a filter** | `sudo tcpdump -i eth0 -w file.pcap port 80` |
| **Read fields from a pcap** | `tshark -r file.pcap -T fields -e ip.src -e http.host` |
| **Run Suricata on a pcap** | `sudo suricata -r file.pcap -l logs/` |
| **List interfaces** | `tcpdump -D` |

### Patterns to remember

| Pattern | Likely meaning |
|---------|-----------------|
| One source, many ports, few full handshakes | Port scan |
| One source, many hosts, same port | Host sweep |
| New, long-lived, odd-port outbound connection | Reverse shell / C2 |
| Large, one-way, unusual-time transfer | Possible exfiltration |
| Random-looking subdomains, high volume | DNS tunneling |

### One-line summaries

- **Capture filters decide what you record; display filters decide what you see.**
- **Follow TCP Stream** reassembles a whole plain-text conversation in one view.
- **A single packet rarely tells a story. Patterns across many do.**
- **Signature-based IDS catches the known; anomaly-based tries to catch the unknown, with more false positives.**
- **An alert is a lead, confirmed by the actual packets and the host/app logs around it.**

---

## 9. Self-check quiz

Answer without looking back, then open the key.

1. What is the difference between a capture filter and a display filter?
2. Why can't a laptop plugged into a normal switch port simply see all traffic on that switch?
3. Which Wireshark feature reassembles a full plain-text HTTP exchange into one readable view?
4. What does `tcp.flags.syn==1 && tcp.flags.ack==0` show, and why is it useful for spotting a scan?
5. Why can't Wireshark show you the content of a normal HTTPS conversation by default?
6. Name two traffic patterns that suggest a scan is happening.
7. What is the key difference between an IDS and an IPS?
8. What is the main trade-off of anomaly-based detection compared to signature-based detection?
9. In the example Suricata rule in this chapter, what does the `threshold` option do?
10. Why is an IDS alert described as "a lead, not a confirmed incident"?

<details>
<summary>Show answer key</summary>

1. A capture filter (BPF syntax) decides what traffic is written to the capture at all, applied before or during capture. A display filter (Wireshark field syntax) decides what is shown from an already-captured file, applied afterward.
2. A switch only sends each frame out the port where its destination lives, so a host on one port normally sees only its own traffic, broadcasts and multicasts, unless a SPAN port, tap or ARP spoofing changes that.
3. Follow TCP Stream.
4. It shows packets that are SYN-only (not SYN-ACK), which are the first packet of a TCP handshake. Many of these from one source to many ports or hosts, without full handshakes completing, is the classic pattern of a SYN scan.
5. The actual application data in HTTPS is encrypted during the TLS session; Wireshark can show the handshake but not the payload unless it is given the session keys (for example via `SSLKEYLOGFILE` in a lab you control).
6. Any two of: one source hitting many ports on one host, one source hitting many hosts on one port, a burst of SYNs with very few completed handshakes.
7. An IDS only monitors and alerts; an IPS sits inline and can actively block or drop matching traffic.
8. Anomaly-based detection can catch new, previously unknown attacks by noticing deviation from a baseline, but it typically produces more false positives and needs a well-tuned baseline to be useful.
9. It stops the rule from alerting on every single match, and only fires once a source has matched the rule a set number of times (5) within a set time window (60 seconds), reducing noise from a single stray attempt.
10. Because it is based on a pattern match (or a deviation from baseline), which can be a false positive; an analyst must corroborate it with the actual packets, flow data, and host or application logs before treating it as a confirmed security incident.

</details>

---

## What comes next

**Chapter 10: Cloud Security.** You can now read traffic on a network you control end to end. Next you move into environments you do not fully control in the same way: AWS and Azure basics, identity and access management, storage security, and the misconfigurations that make cloud breaches so common.

[Back to roadmap](../README.md) | [Previous: Chapter 8 Penetration Testing](../08-penetration-testing/README.md) | [Download the PDF version](Chapter-09-Network-Analysis.pdf)
