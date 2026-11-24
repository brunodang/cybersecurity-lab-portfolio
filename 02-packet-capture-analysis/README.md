# Task 2 — Packet Capture Analysis (Wireshark)

> **Authorized lab work.** All traffic was captured on an **isolated VirtualBox Host-Only network** between two VMs I own (Kali and Metasploitable 2), with no internet access. The credentials used (`analyst` / `Demo123!`) are **fake demo values** created for this exercise — no real accounts or systems were involved.

## Objective

Demonstrate *why* cleartext network protocols are a critical security risk. Task 1 flagged FTP (port 21) and Telnet (port 23) as cleartext services — this task **proves the impact** by capturing live FTP traffic and extracting the transmitted username and password straight from the packets.

## Environment

| Component | Detail |
|---|---|
| Analyst host | Kali Linux — `192.168.223.4` |
| Target | Metasploitable 2 — `192.168.223.3` |
| Network | VirtualBox Host-Only (isolated, no internet) |
| Tool | Wireshark |
| Protocol observed | FTP (port 21) |

## Method

**1. Started a live capture.** Opened Wireshark in Kali and began capturing on the Host-Only interface (`eth0`), the link carrying lab traffic.

**2. Generated a login over FTP** from the Kali terminal to the target, using harmless demo credentials:
```bash
ftp 192.168.223.3
# Name: analyst
# Password: Demo123!
```
The login itself failing is irrelevant — the credentials are transmitted the moment they're typed.

**3. Stopped the capture** and narrowed to the relevant traffic with a display filter:
```
ftp
```

**4. Reconstructed the conversation.** Right-clicked an FTP packet → **Follow → TCP Stream**, which reassembles the entire exchange into plain readable text.

## Findings

The FTP control channel transmitted both the username and password in **plain, unencrypted text**, fully visible in the captured packets:

| Packet (Info column) | What it exposed |
|---|---|
| `Request: USER analyst` | The **username**, in cleartext |
| `Request: PASS Demo123!` | The **password**, in cleartext |

> 📸 *Evidence screenshot (Follow TCP Stream) is stored in `screenshots/` in this folder.*

Anyone positioned on the network path — through packet sniffing, ARP spoofing, or a rogue access point — could read these credentials with no decryption required.

## Analysis — why it matters

FTP, Telnet, HTTP, and the r-services (all found open in Task 1) send data, **including logins, in cleartext**. On a shared or compromised network this means:

- **Credential theft** — usernames and passwords are captured directly off the wire.
- **Identity theft & account takeover** — stolen credentials are reused against other systems (people reuse passwords).
- **Lateral movement** — one sniffed admin login can open the rest of the network.
- **Session hijacking** — cleartext session data can be replayed or manipulated.

This is why a packet analyst treats *any* cleartext authentication as a high-severity finding, not a minor one.

## Recommended remediation

1. **Replace cleartext protocols with encrypted equivalents** — SFTP/FTPS instead of FTP, SSH instead of Telnet, HTTPS instead of HTTP.
2. **Disable the legacy services entirely** — Telnet and r-services have no safe modern use.
3. **Segment and monitor the network** — limit who can be on the same broadcast domain; watch for ARP spoofing and interfaces in promiscuous mode.
4. **Enforce unique, non-reused credentials** so a single leak can't cascade.

## Skills demonstrated

- Live traffic capture with Wireshark and correct interface selection
- Using **display filters** to isolate a protocol
- **Follow TCP Stream** to reassemble and read a full session
- Protocol-level analysis — recognizing cleartext authentication and its real-world impact
- Connecting a scan finding (Task 1) to a demonstrated, evidenced risk

---

*Related: [Task 1 — Nmap Port Scan](../../01-security-tasks/1-nmap-port-scan/) first identified ports 21 and 23 as cleartext services. This task demonstrates the consequence.*
