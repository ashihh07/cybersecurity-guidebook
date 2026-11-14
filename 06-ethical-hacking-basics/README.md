# Chapter 6: Ethical Hacking Basics

**Cybersecurity Guide Book | Roadmap step 6 of 12** | Author: [ashihh07](https://github.com/ashihh07)

[Back to roadmap](../README.md) | [Previous: Chapter 5 Web Security](../05-web-security/README.md) | [Download the PDF version](Chapter-06-Ethical-Hacking-Basics.pdf)

> Ethical hacking is a method, not a bag of tricks: get permission, learn about the target, find what is exposed, ask each exposed service what it is. This chapter teaches that method, and how defenders see every step of it.

---

## About this chapter

Chapter 5 showed individual web flaws. Real assessments do not start with a flaw. They start with questions: what does this organisation own, what is reachable, what is running on it? Answering those questions is **reconnaissance, footprinting, scanning and enumeration**, and it is the first half of nearly every penetration test and every real-world attack.

The roadmap lists four items:

- **Reconnaissance:** collecting information about a target.
- **Footprinting:** turning that information into a map of the target's internet-facing presence.
- **Scanning (Nmap):** finding live hosts, open ports and running services.
- **Enumeration:** pulling detailed information out of each discovered service.

Defenders need this chapter as much as attackers do. Every technique here leaves traces in your logs, and the best way to shrink your attack surface is to look at yourself the way an attacker does.

### How each topic is built

- **Concept**: what it is and why it exists.
- **Attacker view**: what an authorised tester (or a real attacker) does with it.
- **Defender view**: how to detect it and how to reduce what it finds.
- **Labs**: run only inside your own lab network, or on targets that explicitly allow it.

### Contents

| # | Section | Roadmap item covered |
|---|---------|----------------------|
| 1 | [The ethical hacking method](#1-the-ethical-hacking-method) | Orientation |
| 2 | [Reconnaissance](#2-reconnaissance) | Reconnaissance |
| 3 | [Footprinting](#3-footprinting) | Footprinting |
| 4 | [Scanning with Nmap](#4-scanning-with-nmap) | Scanning (Nmap) |
| 5 | [Enumeration](#5-enumeration) | Enumeration |
| 6 | [Recording and reporting what you find](#6-recording-and-reporting-what-you-find) | Method |
| 7 | [Defending: detecting and reducing recon](#7-defending-detecting-and-reducing-recon) | SOC view of all of the above |
| 8 | [Hands-on labs](#8-hands-on-labs) | Practice for all of the above |
| 9 | [Quick revision sheet](#9-quick-revision-sheet) | Tables and commands |
| 10 | [Self-check quiz and answers](#10-self-check-quiz) | Test yourself |

> [!WARNING]
> **Ground rule. Read this twice.**
> - **Scanning is not harmless.** Port scans and enumeration against systems you do not own or have written permission to test can be a criminal offence, even if you "only looked". Laws such as India's Information Technology Act 2000, the UK Computer Misuse Act and the US Computer Fraud and Abuse Act cover unauthorised access.
> - Practise only on: your own lab network (host-only VMs), systems that explicitly invite it (such as `scanme.nmap.org`, within its stated limits), and training platforms (TryHackMe, Hack The Box, PortSwigger).
> - Never point these tools at your school, employer, neighbours, home router's ISP side, or "just a random site".
> - Keep deliberately vulnerable VMs (Metasploitable and similar) on a **host-only** network. Never bridge them to your real network.

---

## 1. The ethical hacking method

### Concept

An **ethical hacker** (penetration tester) uses attacker techniques, with permission, to find weaknesses before real attackers do, and then helps fix them. What separates ethical from criminal is not skill but **authorisation, scope and honesty in reporting**.

### The phases

| Phase | Question it answers | Chapter |
|-------|--------------------|---------|
| **1. Pre-engagement** | What am I allowed to test, how, and when? | This chapter |
| **2. Reconnaissance** | What can I learn about the target without touching it? | This chapter |
| **3. Scanning and enumeration** | What is exposed, and what exactly is running? | This chapter |
| **4. Vulnerability assessment** | Which of those services have known weaknesses? | Chapter 7 |
| **5. Exploitation** | Can a weakness really be used to gain access? | Chapter 8 |
| **6. Post-exploitation** | What could an attacker reach from there? | Chapter 8 |
| **7. Reporting** | What did I find, how bad is it, and how is it fixed? | Section 6 |

### Pre-engagement: the part beginners skip

Before any packet is sent, a professional engagement has a **signed authorisation** and a **rules of engagement** (RoE) document. It states:

- **Scope:** exact IP ranges, domains and applications that may be tested. Everything else is **out of scope**.
- **Allowed and forbidden techniques:** for example, no denial of service, no social engineering, no touching production databases.
- **Testing window:** dates and hours.
- **Contacts and escalation:** who to call if something breaks, and an emergency stop.
- **Data handling:** what you may copy, and how it is stored and destroyed.

This signed permission is informally called a "get out of jail letter". For your own lab, write a one-page version anyway. It builds the habit.

### Mapping to frameworks

| Framework | Where this chapter sits |
|-----------|-------------------------|
| **Lockheed Martin Cyber Kill Chain** | Stage 1: Reconnaissance |
| **MITRE ATT&CK** | Reconnaissance tactic (TA0043), for example *Active Scanning* (T1595), *Gather Victim Host Information* (T1592), *Search Open Technical Databases* (T1596). After a foothold, *Network Service Discovery* (T1046) is in the Discovery tactic |

Defenders map detections to these IDs, and your Chapter 1 and Chapter 5 write-ups can too.

> [!CAUTION]
> **Attacker view**
> - Real attackers skip pre-engagement. They do reconnaissance first because it is cheap, quiet and often reveals an easier door than the front one.
> - They pick targets by what is exposed, not by who the target is. Automated scanners sweep the whole internet continuously.

> [!TIP]
> **Defender view**
> - Assume you are being scanned right now. The skill is separating background noise from a focused, human-driven reconnaissance of your organisation.
> - Run these same techniques against yourself, with approval, on a schedule. What you find is what attackers find.

---

## 2. Reconnaissance

### Concept

**Reconnaissance** is gathering information about a target. It comes in two kinds:

| | Passive reconnaissance | Active reconnaissance |
|---|------------------------|------------------------|
| **What you do** | Collect information from public sources, without sending traffic to the target | Interact directly with the target's systems |
| **Examples** | WHOIS, search engines, certificate logs, social media, public code repositories | Port scans, banner grabbing, probing web pages |
| **Visible to the target?** | No (they cannot tell) | Yes, in logs and IDS |
| **Risk** | Low | Needs authorisation |

**OSINT** (open-source intelligence) is the passive kind: intelligence from publicly available information.

### What passive recon uses

| Source | What it reveals | How |
|--------|-----------------|-----|
| **WHOIS** | Registrar, creation date, name servers, sometimes contacts | `whois example.com` |
| **DNS records** | Mail servers, name servers, SPF/DMARC policy, other hostnames | `dig example.com ANY`, `dig example.com MX`, `dig example.com TXT` |
| **Certificate Transparency** | Every hostname that ever got a public certificate, including forgotten ones | crt.sh website, or `curl -s "https://crt.sh/?q=%25.example.com&output=json" \| jq -r '.[].name_value' \| sort -u` |
| **Search engines (Google dorking)** | Exposed files, login pages, directory listings | `site:example.com filetype:pdf`, `site:example.com inurl:admin`, `intitle:"index of"` |
| **Internet scanners** | Exposed devices, open ports and banners someone already scanned for you | Shodan, Censys (web search, no packets sent by you) |
| **Archives** | Old pages, removed files, forgotten endpoints | Wayback Machine |
| **Code hosting** | Leaked keys, passwords and internal hostnames in public repositories | GitHub search for your own organisation |
| **People and company data** | Names, roles, email format, technologies used (from job posts) | LinkedIn, company site, press releases |
| **Tools** | Automates gathering from many of these | `theHarvester`, `recon-ng`, Maltego (options change often, read each tool's `--help`) |

```bash
whois example.com | head -30
dig example.com NS +short
dig example.com MX +short
dig example.com TXT +short                  # SPF and other policy records
```

### Reading the DNS email records

`dig example.com TXT` shows the **SPF** record (which servers may send mail for the domain) and a separate `_dmarc.example.com` TXT record (what to do with mail that fails). Missing or weak records mean the domain can be spoofed in phishing, which is a finding by itself.

> [!CAUTION]
> **Attacker view**
> - Build a list of hostnames from certificate logs. Old test and staging servers often have weaker protection than the main site.
> - Use job adverts and developer posts to learn the technology stack ("experience with Nginx, Django, AWS S3").
> - Find employee names and the company email format to prepare phishing.
> - Search public repositories for committed passwords, tokens and internal URLs.

> [!TIP]
> **Defender view**
> - Passive recon cannot be detected, so **reduce what is public**: do not leave test hosts with public certificates unattended, strip metadata from documents, and keep job adverts and developer posts free of internal detail.
> - Search for yourself: dork your own domain, check Shodan for your own IP ranges, and scan your own public repositories for secrets.
> - Publish and enforce SPF, DKIM and DMARC.
> - Watch certificate transparency for **new certificates on your names**. An unexpected one can mean a forgotten host or a lookalike used for phishing.

---

## 3. Footprinting

### Concept

**Footprinting** takes the raw information from reconnaissance and builds a **map of the target's footprint**: every domain, subdomain, IP address, technology and person that forms its presence on the internet.

### The footprint checklist

| Area | Questions | Typical tools |
|------|-----------|---------------|
| **Domains and subdomains** | Which names exist? Which are forgotten? | crt.sh, `dig`, `subfinder`, `amass` |
| **IP ranges** | Which addresses and networks does the organisation own? | WHOIS on the IP (`whois 203.0.113.50`), BGP and regional registry lookups |
| **DNS** | Name servers, mail servers, zone transfer allowed? | `dig`, `dnsrecon` |
| **Web technology** | Server, framework, CMS, versions | `curl -I`, `whatweb`, browser tools |
| **Network path** | How does traffic reach the target? | `traceroute`, `mtr` |
| **People** | Staff, roles, email format, social media | OSINT only |
| **Documents** | Authors, software versions, internal paths hidden in metadata | `exiftool file.pdf` |

### DNS zone transfer: a classic misconfiguration

A **zone transfer** (AXFR) lets a secondary name server copy every record of a domain. If a primary server allows it to anyone, a single command reveals the whole internal naming scheme.

```bash
dig NS zonetransfer.me +short                     # find the name servers
dig axfr zonetransfer.me @nsztm1.digi.ninja       # a domain built on purpose for this practice
```

Only try zone transfers on domains you own or on practice domains like the one above.

### Technology fingerprinting

```bash
curl -I https://yourlab.example/                  # Server, X-Powered-By, cookies that reveal frameworks
whatweb -v http://192.168.56.101                  # guesses CMS, server, language, versions
exiftool report.pdf                               # author, software, creation tool, sometimes file paths
```

### Banner grabbing

A **banner** is the greeting a service gives when you connect, often including its name and version.

```bash
nc -nv 192.168.56.101 21                          # FTP banner appears immediately
printf 'HEAD / HTTP/1.0\r\n\r\n' | nc 192.168.56.101 80    # web server headers
```

A version string is gold for an attacker, because it can be matched to published vulnerabilities (Chapter 7). It is also your cue, as a defender, to hide or patch.

> [!CAUTION]
> **Attacker view**
> - Gather every subdomain, then check which ones respond and what each runs. The forgotten `old-portal.` or `dev.` host is the typical weak spot.
> - Read document metadata for usernames, internal server names and software versions.
> - Use the technology list to prepare version-specific tests.

> [!TIP]
> **Defender view**
> - Maintain an **asset inventory**. You cannot protect hosts you do not know exist. Compare it to what footprinting finds, and investigate every difference.
> - Restrict zone transfers to your secondary servers. Use split DNS so internal names are not public.
> - Remove metadata before publishing documents (`exiftool -all= file.pdf`), and hide server banners (Chapter 4).
> - Decommission old hosts and their DNS records, so they cannot be taken over (subdomain takeover).

---

## 4. Scanning with Nmap

### Concept

**Scanning** is active reconnaissance of networks: finding which hosts are alive, which ports are open, and which services answer. **Nmap** ("Network Mapper") is the standard tool, and it builds directly on Chapter 1: TCP flags, the handshake, ports and UDP.

### How a port gets its state

| State | Meaning | What happened (TCP SYN scan) |
|-------|---------|------------------------------|
| **open** | A service is listening | Target replied **SYN-ACK** |
| **closed** | Reachable, but nothing listens | Target replied **RST** |
| **filtered** | A firewall is dropping or blocking probes | **No reply**, or an ICMP "unreachable" |
| **open\|filtered** | Cannot tell (common with UDP) | No reply to a probe that open ports also ignore |
| **unfiltered** | Reachable, but open or closed is unknown | Seen only with ACK scans |

### The core commands

```bash
nmap -sn 192.168.56.0/24                  # host discovery only: who is alive? (no port scan)
nmap 192.168.56.101                       # default: top 1000 TCP ports
sudo nmap -sS 192.168.56.101              # SYN ("half-open") scan, needs root
nmap -sT 192.168.56.101                   # full TCP connect scan, works without root
nmap -p 22,80,443 192.168.56.101          # chosen ports
nmap -p- 192.168.56.101                   # all 65,535 ports
nmap --top-ports 100 192.168.56.101       # the 100 most common ports
sudo nmap -sU --top-ports 20 192.168.56.101   # UDP is slow, so keep it short
nmap -sV 192.168.56.101                   # probe services to detect names and versions
sudo nmap -O 192.168.56.101               # guess the operating system
nmap -sC 192.168.56.101                   # run the default NSE scripts
sudo nmap -A 192.168.56.101               # aggressive: -sV -O -sC and traceroute
nmap -Pn 192.168.56.101                   # skip host discovery, treat host as up
```

### Output you must keep

```bash
nmap -sV -p- -oA scans/target1 192.168.56.101
# writes target1.nmap (readable), target1.xml (for tools and scripts, Chapter 3), target1.gnmap (grep-friendly)
```

An example result (illustrative):

```text
PORT     STATE    SERVICE  VERSION
22/tcp   open     ssh      OpenSSH 8.9p1 Ubuntu
80/tcp   open     http     nginx 1.18.0
443/tcp  closed   https
3306/tcp filtered mysql
```

How to read it: SSH and a web server are reachable, nothing listens on 443, and something is filtering 3306 (a database port that should not be public, so the filter is doing its job).

### Timing and noise

| Option | Effect |
|--------|--------|
| `-T0` to `-T5` | Timing templates: `-T0` paranoid-slow, `-T3` default, `-T4` fast on a good network, `-T5` insane (can miss ports and disrupt hosts) |
| `-v` | More progress output while scanning |
| `--open` | Show only open ports |
| `--reason` | Show why a port got its state (for example `syn-ack`) |

Faster scans are noisier and more likely to be blocked or to upset fragile devices. In a lab, `-T4` is fine. On a client network, follow the RoE.

### The Nmap Scripting Engine (NSE)

NSE runs small scripts against services. Scripts are grouped into categories (`default`, `safe`, `discovery`, `vuln`, `intrusive`, `brute`, and more).

```bash
nmap --script http-title,http-headers -p 80 192.168.56.101
nmap --script ftp-anon -p 21 192.168.56.101                 # does the FTP server allow anonymous login?
nmap --script smb-os-discovery -p 445 192.168.56.101
nmap --script "vuln" 192.168.56.101                         # check for known vulnerabilities (Chapter 7)
nmap --script-help http-title                               # read what a script does before you run it
```

> [!WARNING]
> **Read before you run scripts.** Categories like `intrusive`, `brute` and `dos` can lock accounts, crash services or generate heavy traffic. Use them only where your RoE allows, and read `--script-help` first.

> [!CAUTION]
> **Attacker view**
> - Start wide and fast (host discovery, top ports), then narrow to deep scans (`-p-`, `-sV`, scripts) on interesting hosts.
> - Scan **all** ports. Services hidden on odd ports (8080, 8443, 2222, 31337) are often the weakest.
> - Real attackers slow down, spread scans across many source addresses, and vary timing to stay below detection thresholds. That is why simple "N ports in M seconds" rules are only a starting point.

> [!TIP]
> **Defender view**
> - A **SYN scan** shows as many SYNs from one source to many ports (or many hosts) with few completed handshakes. Chapter 1's handshake knowledge is exactly how you recognise it.
> - Nmap's default probes and NSE scripts have recognisable patterns. IDS tools (Suricata, Snort, Zeek) ship rules for them.
> - Default-deny firewall rules make most ports **filtered** and give the scanner little to work with.
> - After you harden, **scan yourself again** and confirm the result changed.

---

## 5. Enumeration

### Concept

**Enumeration** is asking each discovered service for as much detail as it will give: users, shares, directories, software, configuration. Scanning says "port 445 is open". Enumeration says "it exposes these shares, and these usernames exist".

It is **more intrusive** than scanning, because you are talking to the service in its own language. Authorisation matters even more here.

### What to check, by service

| Port | Service | First things to check |
|------|---------|-----------------------|
| **21** | FTP | Banner and version. Anonymous login. Writable directories |
| **22** | SSH | Banner and version, supported authentication methods, host key |
| **25** | SMTP | Banner, whether `VRFY` or `EXPN` reveals valid users, open relay |
| **53** | DNS | Zone transfer, version, recursion open to everyone |
| **80/443** | HTTP(S) | Headers, technologies, hidden paths, `robots.txt`, login pages, certificate details |
| **139/445** | SMB | Null session, share names, readable shares, OS and domain information |
| **161** | SNMP | Default community strings (`public`), which leak device and network details |
| **3306 / 5432** | MySQL / PostgreSQL | Should not be reachable. If it is, version and authentication settings |
| **3389** | RDP | Exposed to the internet at all? Network-level authentication on? |

### Commands

```bash
# Web: find hidden paths (this makes thousands of requests, so be sure it is in scope)
gobuster dir -u http://192.168.56.101 -w /usr/share/wordlists/dirb/common.txt
curl -s http://192.168.56.101/robots.txt

# FTP: anonymous access?
ftp 192.168.56.101                                 # user: anonymous, password: any text

# SMB: list shares without credentials (null session)
smbclient -L //192.168.56.101 -N
nmap --script smb-enum-shares -p 445 192.168.56.101

# SNMP: read device information using the default community string
snmpwalk -v2c -c public 192.168.56.101 | head

# SMTP: manual banner and user check
nc -nv 192.168.56.101 25                           # then type:  VRFY root
```

On Kali, `gobuster`, `smbclient`, `snmpwalk` and similar come preinstalled or one `apt install` away. Wordlist paths differ between systems, so use `locate common.txt` to find yours.

### From version to vulnerability (a preview of Chapter 7)

Enumeration produces facts such as "vsftpd 2.3.4" or "Apache 2.4.49". The next step is looking up whether that version has known weaknesses:

```bash
searchsploit apache 2.4.49          # local copy of the Exploit-DB index (sudo apt install exploitdb)
```

Also search the **NVD** (nvd.nist.gov) and the vendor's advisories. A match is a lead, not proof. Chapter 7 shows how to confirm and rate it.

> [!CAUTION]
> **Attacker view**
> - Collect usernames from SMB, SMTP, web pages and metadata. They feed password spraying (Chapter 5).
> - Look for **unauthenticated** access first: anonymous FTP, null SMB sessions, default SNMP strings, open databases. These need no exploit at all.
> - Directory brute-forcing finds admin panels, backups and old versions of the site.

> [!TIP]
> **Defender view**
> - Disable anonymous FTP, null sessions and SNMP v1/v2c with default strings. Do not expose SMB, databases or RDP to the internet.
> - **Brute-force directory scans are loud.** A web server log shows thousands of requests in seconds, mostly `404`, often with a tool name in the `User-Agent`. The Chapter 4 and Chapter 5 `awk` and `grep` searches find them.
> - Turn off verbose banners and error pages. Remove services you do not need.
> - Use **honeypots** and **canary tokens**: a fake admin page or share that no real user has any reason to touch. Any hit is a high-confidence alert.

---

## 6. Recording and reporting what you find

### Concept

Good testers are good note-takers. Findings you cannot reproduce or explain are worth nothing. Keep an organised record from the first command.

### A simple host record

```text
Host: 192.168.56.101   (Metasploitable 2, lab)        Date: 2026-11-10
Scope check: in scope per lab RoE v1

Port   Service   Version              Notes / next step
21     ftp       vsftpd 2.3.4         anonymous login? check NVD for version
22     ssh       OpenSSH 4.7p1        old, check supported auth methods
80     http      Apache 2.2.8         /dvwa, /mutillidae found with gobuster
445    smb       Samba 3.0.20         null session lists shares
```

Save every Nmap run with `-oA`, keep terminal logs (`script session.log`), and screenshot key evidence. Tools such as CherryTree or Obsidian work well for notes.

### What a finding contains

| Part | Content |
|------|---------|
| **Title** | Short and specific: "Anonymous FTP login enabled on 192.168.56.101" |
| **Severity** | Critical, High, Medium, Low, Info, using a consistent scale (Chapter 7 covers CVSS) |
| **Evidence** | The exact command and output or screenshot that proves it |
| **Impact** | What an attacker could do with it, in business terms |
| **Remediation** | A concrete fix: "disable anonymous access in `vsftpd.conf`" |
| **References** | CVE, vendor advisory, OWASP or CIS guidance |

Write for two readers: a manager who needs impact and priority, and an engineer who needs the exact fix. A report that only lists tools and output is not a report.

---

## 7. Defending: detecting and reducing recon

### What the traces look like

| Activity | Where it shows | What to look for |
|----------|----------------|------------------|
| **Port scan** | Firewall logs, IDS, packet captures | One source, many ports or hosts, many SYNs, few completed handshakes, many RSTs back |
| **Host sweep** | Firewall and IDS logs | One source, ICMP or SYN to many consecutive addresses |
| **Service probing** | IDS and application logs | Connections that open and close without real data, odd banners requested |
| **Directory brute force** | Web access log | Bursts of `404`, sequential wordlist-like paths, tool `User-Agent` |
| **Zone transfer attempt** | DNS server log | `AXFR` requests from unknown hosts |
| **SMB and SNMP enumeration** | Windows security log, network logs | Anonymous or null logons, SNMP requests with unknown community strings |

### A quick scan detector with tcpdump

```bash
# Count SYN-only packets per source address over 500 packets
sudo tcpdump -nn -l -c 500 'tcp[tcpflags] == tcp-syn' 2>/dev/null \
  | awk '{for(i=1;i<=NF;i++) if($i=="IP"){print $(i+1); break}}' \
  | rev | cut -d. -f2- | rev | sort | uniq -c | sort -nr | head
```

A single address with hundreds of SYNs, while the same capture shows few completed conversations, is a scan. In Wireshark, use the display filter `tcp.flags.syn==1 && tcp.flags.ack==0`, then **Statistics, Conversations**.

```bash
# Web server: who is brute-forcing paths? (Chapter 4 log format)
awk '$9==404 {print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
awk -F'"' '{print $6}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head    # user agents
```

### Reducing what recon finds (attack surface reduction)

- **Default-deny firewall:** only the ports that must be public. Everything else is filtered.
- **Do not expose management services** (SSH, RDP, databases, SNMP, SMB) to the internet. Put them behind a VPN or an allow-list.
- **Patch** so that the versions found are not vulnerable. Hide banners as a minor extra.
- **Inventory and decommission:** remove unused hosts, DNS records and certificates.
- **Self-scan regularly,** compare with your inventory, and alert on differences.
- **Detect and respond:** IDS/IPS (Suricata, Snort, Zeek), `fail2ban` for repeated failures, rate limiting at the proxy.
- **Limit what the public learns:** WHOIS privacy, trimmed job adverts, clean documents, no secrets in repositories.

> [!IMPORTANT]
> **Scans on the internet never stop.** A public server is probed constantly, so a lone scan alert is low priority. Raise severity when the same source also shows **authentication attempts, exploit strings or follow-up access**, when the scan comes from **inside** your network, or when it targets **one specific asset** and avoids the rest. That combination is how a SOC analyst turns noise into signal.

---

## 8. Hands-on labs

Everything runs in your own isolated lab or on explicit practice targets. Write each lab up (goal, commands, evidence, the defender view, what you learned).

### Lab 1: Build an isolated lab and a scope document

Create a **host-only** network in VirtualBox. Put two VMs on it: an attacker machine (Kali or any Linux you used earlier) and a deliberately vulnerable target such as Metasploitable 2. Disable bridged networking on the target. Take snapshots of both. Then write a one-page `lab-scope.md`: in-scope ranges, forbidden actions, dates.

**Goal:** from the attacker VM `ping 192.168.56.x` (the target) works, and from the target you cannot reach the internet or your home network.

### Lab 2: Passive recon on a domain you own (or a practice domain)

```bash
whois yourdomain.com | head -40
dig yourdomain.com NS MX TXT +short
dig _dmarc.yourdomain.com TXT +short
curl -s "https://crt.sh/?q=%25.yourdomain.com&output=json" | jq -r '.[].name_value' | sort -u
exiftool somefile.pdf                      # use a PDF you created
```

If you do not own a domain, use only passive public sources on a practice domain such as `zonetransfer.me`, or read about a well-known public organisation without sending it any traffic.

**Goal:** a short list of hostnames, mail servers and policy records, plus a note on what metadata a PDF of yours leaked.

### Lab 3: DNS zone transfer on a practice domain

```bash
dig NS zonetransfer.me +short
dig axfr zonetransfer.me @nsztm1.digi.ninja
```

**Goal:** read the records you received and explain what an attacker learns from them, and which two settings would stop the transfer.

### Lab 4: Nmap against your lab target

```bash
nmap -sn 192.168.56.0/24
sudo nmap -sS -p- --reason -oA scans/ms2-full 192.168.56.101
nmap -sV -p 21,22,80,445 -oA scans/ms2-services 192.168.56.101
sudo nmap -O 192.168.56.101
```

**Goal:** a table of open ports with service names and versions, and for three ports an explanation of why they are in that state (use `--reason`).

### Lab 5: One light scan of a public target that permits it

```bash
nmap -sV -F scanme.nmap.org               # -F scans the 100 most common ports
```

Read the rules on the `scanme.nmap.org` page first and stay within them: light scans only, no abuse, no repeated heavy scanning.

**Goal:** compare its open ports and versions with your lab target, and note one difference in how a managed host looks versus a deliberately vulnerable one.

### Lab 6: Banner grabbing and fingerprinting

```bash
nc -nv 192.168.56.101 21
printf 'HEAD / HTTP/1.0\r\n\r\n' | nc 192.168.56.101 80
curl -I http://192.168.56.101
whatweb -v http://192.168.56.101
```

Take two version strings and look each up in the NVD or with `searchsploit`.

**Goal:** a table of service, version and the number of known issues you found. Say plainly that a match is a **lead to verify**, not a confirmed vulnerability.

### Lab 7: Enumerate services on the lab target

```bash
nmap --script ftp-anon,smb-os-discovery,http-title -p 21,80,445 192.168.56.101
smbclient -L //192.168.56.101 -N
gobuster dir -u http://192.168.56.101 -w /usr/share/wordlists/dirb/common.txt
```

**Goal:** list the paths and shares you found, and the first three actions an engineer should take to close each exposure.

### Lab 8: Be the defender

Run `tcpdump` (or Wireshark) on the target or the host-only interface while you repeat the scan from Lab 4 and the `gobuster` run from Lab 7. Apply the SYN counter and the web log searches from [section 7](#7-defending-detecting-and-reducing-recon). Then make three hardening changes on the target (stop an unneeded service, add a firewall rule, hide a banner) and run the same scan again.

**Goal:** a before-and-after Nmap comparison, a detection rule in plain English for the scan and the directory brute force, and one way an attacker could evade each rule.

> [!NOTE]
> **Next step on TryHackMe:** work through the Nmap and "Network Services" rooms, and the introductory Pre Security reconnaissance content. Write each up with the commands, the evidence and the defender view.

---

## 9. Quick revision sheet

### Nmap cheat sheet

| Goal | Command |
|------|---------|
| **Who is alive?** | `nmap -sn 192.168.56.0/24` |
| **Quick scan** | `nmap host` |
| **SYN scan** | `sudo nmap -sS host` |
| **All ports** | `nmap -p- host` |
| **Services and versions** | `nmap -sV host` |
| **OS guess** | `sudo nmap -O host` |
| **Default scripts** | `nmap -sC host` |
| **Everything (aggressive)** | `sudo nmap -A host` |
| **UDP (top 20)** | `sudo nmap -sU --top-ports 20 host` |
| **Save all outputs** | `nmap -oA name host` |
| **Why that state?** | `nmap --reason host` |
| **Skip ping** | `nmap -Pn host` |

### Recon and enumeration tools

| Task | Tool |
|------|------|
| **Domain and registrar info** | `whois` |
| **DNS records and zone transfer** | `dig`, `dnsrecon` |
| **Subdomains** | crt.sh, `subfinder`, `amass` |
| **Internet-wide exposure search** | Shodan, Censys |
| **Web technology** | `whatweb`, `curl -I` |
| **Document metadata** | `exiftool` |
| **Banner grabbing** | `nc`, `nmap -sV` |
| **Web directories** | `gobuster`, `ffuf` |
| **SMB** | `smbclient`, `enum4linux-ng` |
| **SNMP** | `snmpwalk` |
| **Version to known issue** | NVD, `searchsploit` |

### One-line summaries

- **Permission, scope, written down.** No exceptions.
- **Passive** collects public information without touching the target. **Active** touches it, so it needs authorisation.
- **Footprinting** is the map: domains, ranges, technologies, people.
- **Open** = SYN-ACK, **closed** = RST, **filtered** = silence.
- **Scanning** finds services. **Enumeration** extracts detail. A version match is a lead, not a verdict.
- **Defend by shrinking the surface and watching for scans,** then scan yourself to prove it.

---

## 10. Self-check quiz

Answer without looking back, then open the key.

1. What two documents should exist before any professional test begins?
2. What is the difference between passive and active reconnaissance?
3. Which Nmap port state means "no reply, probably a firewall"?
4. Why does a SYN scan need root, and what does `-sT` do instead?
5. What does `nmap -sV` add to a basic port scan?
6. Name two things a DNS zone transfer can expose.
7. Why is directory brute-forcing easy for a defender to detect?
8. What is the difference between scanning and enumeration?
9. A banner shows "Apache 2.2.8". Is the server vulnerable? Explain.
10. A public web server shows a port scan in its logs. When should the analyst escalate?

<details>
<summary>Show answer key</summary>

1. A signed authorisation (permission) and rules of engagement that define scope, allowed techniques, timing and contacts.
2. Passive collects information from public sources without sending traffic to the target and cannot be detected. Active interacts directly with the target's systems, shows in its logs, and needs authorisation.
3. `filtered`.
4. A SYN scan crafts raw packets and never completes the handshake, which needs raw socket access. `-sT` uses the operating system's normal `connect()` call and completes the full handshake, so it works without root but is more visible.
5. It probes open ports to detect the service name and version.
6. Any two of: all hostnames, internal naming schemes, mail servers, IP addresses, forgotten test or staging hosts.
7. It sends hundreds or thousands of requests in seconds, mostly `404`, from one address, often with a tool name in the `User-Agent`.
8. Scanning finds which hosts, ports and services exist. Enumeration queries each service for detail such as users, shares, paths and configuration, and is more intrusive.
9. Not necessarily. It is a lead: that version has known issues, but the banner can be wrong or patched. It must be verified (Chapter 7).
10. Scans on the internet are constant background noise. Escalate when the same source follows with login attempts or exploit strings, when the scan comes from inside the network, or when it targets one specific asset.

</details>

---

## What comes next

**Chapter 7: Vulnerability Assessment.** You can now find what is exposed and what is running. Next you learn to find out which of it is vulnerable, using tools such as Nessus, OpenVAS, Nikto and SSL Labs, and to rate and prioritise the risk.

[Back to roadmap](../README.md) | [Previous: Chapter 5 Web Security](../05-web-security/README.md) | [Download the PDF version](Chapter-06-Ethical-Hacking-Basics.pdf)
