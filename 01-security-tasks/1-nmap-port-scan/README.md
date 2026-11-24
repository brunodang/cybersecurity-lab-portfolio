# Task 1 — Network Port Scan & Service Enumeration (Nmap)

> **Authorized lab work.** All scanning was performed against a Metasploitable2 virtual machine I own, running on an **isolated VirtualBox Host-Only network with no internet access**. No external or third-party systems were touched.

## Objective

Perform a port scan of a target host to identify every open port, the service behind it, and its version — then assess which exposed services present a security risk. This is the first step of the reconnaissance phase and a core SOC / analyst skill.

## Environment

| Component | Detail |
|---|---|
| Hypervisor | Oracle VirtualBox |
| Attacker | Kali Linux |
| Target | Metasploitable 2 (intentionally vulnerable Linux VM) |
| Network | Host-Only adapter (isolated, no internet) |
| Tool | Nmap 7.99 |

## Method

**1. Isolated the lab network.** Placed both VMs on the same VirtualBox **Host-Only** adapter so the vulnerable target could never reach the internet.

**2. Confirmed connectivity** from the attacker to the target:
```bash
ping -c 4 <target-ip>
# replies received — host reachable
```

**3. Ran the scan** with service/version detection, default scripts, and output saved in all formats:
```bash
nmap -sV -sC -oA metasploitable2_scan <target-ip>
```

| Flag | Purpose |
|---|---|
| `-sV` | Detect the service **version** on each open port |
| `-sC` | Run Nmap's default **scripts** for extra detail |
| `-oA` | Save the log in all three formats (`.nmap`, `.gnmap`, `.xml`) as evidence |

Scan completed in ~197 seconds. **23 open TCP ports** were found.

## Results (summary)

| Port | Service | Version |
|---|---|---|
| 21 | FTP | vsftpd 2.3.4 (anonymous login allowed) |
| 22 | SSH | OpenSSH 4.7p1 |
| 23 | Telnet | Linux telnetd |
| 25 | SMTP | Postfix (SSLv2 / weak ciphers) |
| 53 | DNS | ISC BIND 9.4.2 |
| 80 | HTTP | Apache 2.2.8 |
| 111 / 2049 | RPC / NFS | rpcbind, NFS exports |
| 139 / 445 | SMB | Samba 3.0.20 (signing disabled) |
| 512 / 513 / 514 | r-services | rexec, rlogin, rsh |
| 1099 | Java RMI | GNU Classpath grmiregistry |
| 1524 | Bind shell | **Root shell backdoor** |
| 2121 | FTP | ProFTPD 1.3.1 |
| 3306 | MySQL | 5.0.51a |
| 5432 | PostgreSQL | 8.3.x |
| 5900 | VNC | protocol 3.3 (weak auth) |
| 6000 | X11 | exposed |
| 6667 | IRC | UnrealIRCd |
| 8009 / 8180 | AJP / HTTP | Apache Tomcat/Coyote 1.1 |

Full logs: `metasploitable2_scan.nmap` (and `.gnmap`, `.xml`).

## Analysis — what the findings mean

The host exposes 23 services with effectively no hardening. Key risk categories:

- **Cleartext protocols** (FTP 21, Telnet 23, r-services 512–514): credentials and data travel unencrypted and can be captured by anyone on the network. These should be replaced with SSH/SFTP.
- **Services with known backdoors** (vsftpd 2.3.4, UnrealIRCd on 6667, the bind shell on 1524): these specific versions ship with built-in backdoors that grant remote access.
- **Weak or anonymous authentication** (anonymous FTP, VNC, MySQL, Tomcat): several services allow login with no credentials or defaults.
- **Exposed file sharing and data services** (Samba with signing disabled, NFS, PostgreSQL/MySQL): sensitive data and remote code paths exposed to the network.
- **Outdated software** (Apache 2.2.8, OpenSSH 4.7, BIND 9.4.2, SSLv2 support): years of unpatched vulnerabilities.

## Recommended remediation

1. **Disable cleartext services** — remove Telnet, FTP and r-services; use SSH/SFTP instead.
2. **Patch or remove outdated software** — every service here is years past end of support.
3. **Close unused ports** — apply least functionality; only run services the host actually needs.
4. **Enforce authentication** — no anonymous FTP, no default/blank database or VNC passwords.
5. **Harden SMB** — enable signing; restrict shares.
6. **Segment the host** — a box like this should never be internet-facing; firewall it to only required traffic.

## Skills demonstrated

- Network reconnaissance and service enumeration with Nmap
- Reading service/version output and mapping it to real risk
- Building an isolated, safe testing lab (VirtualBox Host-Only)
- Translating technical findings into prioritized remediation
