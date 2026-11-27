# Cybersecurity Lab Portfolio

Hands-on security labs documenting practical skills in **network reconnaissance, vulnerability analysis, packet capture, memory forensics, and file carving** — the core day-to-day work of a SOC / security analyst.

Each task is written up as a short report: what I did, the tools and commands used, what the findings mean, and how I'd remediate them.

> ⚠️ **Authorized lab work only.** Every exercise here was performed against intentionally vulnerable practice systems (e.g. Metasploitable 2) that I own, running on an **isolated virtual network with no internet access**. No external, third-party, or production systems were ever touched.

---

## Tasks

| # | Task | Focus | Tools | Status |
|---|------|-------|-------|--------|
| 1 | [Network Port Scan & Service Enumeration](01-security-tasks/1-nmap-port-scan/) | Reconnaissance | Nmap | ✅ Complete |
| 2 | [Packet Capture Analysis](02-packet-capture-analysis/) | Packet Capture Analysis/Network traffic forensics | Wireshark | ✅ Complete |
| 3 | [Memory Dump Analysis](03-memory-forensics/) | Memory forensics | Volatility 3 | ✅ Complete |
| 4 | [File Carving](04-file-carving/) | Data recovery: self-built lab + evidence image | scalpel / foremost / Sleuth Kit | ✅ Commplete |
| 5 | [Vulnerability Scan](05-vulnerability-scan/) | Vulnerability assessment | OpenVAS / GVM | ✅ Complete |
| 6 | [Wireless Network Scan](06-wireless-scan/) | Wireless recon | airodump-ng | ✅ Complete |
| 7 | [Exploitation & Defense](07-exploitation-defense/) | Attack lifecycle & defense | Concept write-up | ✅ Complete |

---

## Skills demonstrated

- **Network reconnaissance** — identifying open ports, services, and versions, then mapping them to real-world risk
- **Vulnerability analysis** — recognizing outdated software, backdoors, weak authentication, and cleartext protocols
- **Digital forensics** — analyzing captured traffic, memory dumps, and recovering deleted files
- **Lab engineering** — building safe, isolated test environments with VirtualBox (Host-Only networking)
- **Technical reporting** — translating findings into clear, prioritized remediation that non-experts can act on

## Tools & environment

`Nmap` · `Wireshark` · `Volatility` · `OpenVAS` · `foremost` · `Kali Linux` · `Oracle VirtualBox` · `Metasploitable 2`

---

## About

I'm transitioning into cybersecurity with a focus on SOC / security analyst roles, currently working toward **CCNA** and **CompTIA Security+**. This repo is where I document the hands-on labs behind the coursework.

*Each write-up is sanitized for public sharing — real IPs, MAC addresses, and any sensitive output are redacted.*
