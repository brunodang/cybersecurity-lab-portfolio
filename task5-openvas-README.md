# Task 5: Vulnerability Scan with OpenVAS

> **Authorized lab work.** The scan targeted my own Metasploitable 2 VM (`192.168.223.3`) on an isolated VirtualBox Host-Only network. Kali had a second, NAT adapter only to download OpenVAS's vulnerability database. Any passwords that appear in the raw report are Metasploitable's publicly documented defaults, not real credentials.

## Objective

Task 1 used Nmap to find **which services were open**. This task goes a step further: run a full vulnerability scanner, **OpenVAS** (Greenbone Vulnerability Management), to check every open service against its database of known weaknesses, rate each finding, and produce a report someone could act on.

## Result at a glance

| Critical | High | Medium | Low | Total shown | CVEs referenced | Scan time |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **14** | **8** | **39** | **6** | 67 (of 630 raw results) | 34 | 1 h 11 min |

Highest severity: **10.0 / 10 (Critical)**. Only results OpenVAS was at least 70% confident in (Quality of Detection ≥ 70) are shown, which is the standard setting for cutting false alarms.

📄 Full scanner output: [`openvas-raw-report.pdf`](openvas-raw-report.pdf) (71 pages, unedited).
📄 Annotated walkthrough with screenshots: [`Task5_OpenVAS_Report_Summary.pdf`](Task5_OpenVAS_Report_Summary.pdf).

## Critical findings (CVSS 9.0–10.0)

| Port | Finding | CVE | Plain meaning | Fix |
|---|---|---|---|---|
| 21 | vsftpd 2.3.4 backdoor | CVE-2011-2523 | A tampered release of the FTP server opens a hidden root shell | Replace vsftpd with a clean, current version |
| 6200 | Shell opened by that backdoor | CVE-2011-2523 | The hidden shell, listening on the network | Same as above |
| 1524 | "Ingreslock" backdoor | — | A root shell left listening on a port | Remove it; firewall the port |
| 512 | rexec running | CVE-1999-0618 | Remote command service that sends logins in cleartext | Disable it; use SSH |
| 513 | rlogin with no password | — | Anyone can log in as root, no password asked | Disable it; use SSH |
| 3632 | DistCC remote code execution | CVE-2004-2687 | A build-helper service that runs anyone's commands | Disable it, or restrict to trusted hosts |
| 8787 | Distributed Ruby (dRuby) | — | A Ruby service that runs commands sent to it | Disable it, or restrict access |
| 8009 | Tomcat AJP "Ghostcat" | CVE-2020-1938 | Lets attackers read web-app files and possibly run code | Patch Tomcat; disable AJP if unused |
| 80 | TWiki < 4.2.4 | CVE-2008-5304, CVE-2008-5305 | Web wiki with command execution and XSS flaws | Update TWiki |
| 80 | PHP < 5.3.13 (PHP-CGI) | CVE-2012-1823 and others | Crafted web requests can run code on the server | Update PHP |
| 3306 | MySQL `root` with an empty password | — | Full database access with no password | Set a strong root password |
| 5432 | PostgreSQL default login | — | `postgres` / `postgres` works | Change the password |
| 5900 | VNC weak password | — | Remote desktop protected only by `password` | Strong password, or disable VNC |
| — | Operating system end of life | — | Ubuntu 8.04, end of life since May 2013: no security updates | Move to a supported OS |

High findings include **cleartext rsh and rlogin**, **Java RMI remote code execution** (1099), **default FTP logins** (`msfadmin:msfadmin` and others on 21 and 2121), the **OpenSSL CCS man-in-the-middle bug** (CVE-2014-0224), and **dangerous HTTP methods** on the web server.

## Comparing with Task 1 (Nmap)

**Confirmed:** the vsftpd backdoor, the 1524 root shell, rlogin and rexec, weak VNC, and the MySQL/PostgreSQL logins that Task 1 flagged as risky are all rated Critical here, now with CVE numbers and scores.

**New, found only by OpenVAS:**
- **Ports 3632 (DistCC) and 8787 (dRuby)** never showed up in Task 1. My Nmap scan checked only the 1,000 most common ports; this scan used the full list of registered TCP ports.
- **Port 6200** wasn't open during Task 1 either. The vsftpd backdoor opens it only when triggered, and OpenVAS's own *active check* most likely triggered it to prove the flaw. Active checks can change the target's state, so they need care on real production systems.
- **Web application flaws on port 80** (TWiki, PHP) that a port scan can't see.

**Different or missing:**
- **UnrealIRCd (6667)**, a known backdoor that Task 1 flagged, isn't in this report. No single tool catches everything, which is why analysts use more than one.
- **Samba "usermap script" (CVE-2007-2447)** appears only as **Medium 6.0**, even though it can give remote root access. The score comes from an older scoring scale. Severity scores are a starting point for prioritizing, not the final word.

## Remediation priorities

1. **Remove the backdoors now:** vsftpd 2.3.4, the 1524 shell, and anything listening on 6200.
2. **Turn off remote-execution and cleartext services:** rexec, rlogin, rsh, DistCC, dRuby, Telnet. Use SSH instead.
3. **Set real passwords** on MySQL, PostgreSQL, VNC and FTP accounts.
4. **Patch the web stack:** PHP, TWiki and Tomcat (or disable AJP).
5. **Replace the operating system:** Ubuntu 8.04 is end of life, so it can't be fully fixed by patching.
6. **Firewall the host** so only required ports are reachable.

## How it was done

```bash
sudo gvm-setup                    # database, certificates, admin user, first feed download
sudo greenbone-feed-sync          # download vulnerability tests and CVE data
sudo gvm-check-setup              # verify the installation
```

Then in the web interface (`https://127.0.0.1:9392`): **Target** `Metasploitable2` → `192.168.223.3`, all IANA-assigned TCP ports, "consider alive" → **Task** with scanner *OpenVAS Default* and scan config *Full and fast* → report exported as PDF.

Because the feed was still loading and the web interface kept the Start button locked, I started the scan through GVM's API instead:

```bash
sudo -u _gvm gvm-cli --gmp-username admin --gmp-password '<redacted>' socket \
  --xml "<start_task task_id='<task-id>'/>"
# → <start_task_response status="202" status_text="OK, request submitted">
```

## Troubleshooting log

| Problem | Cause | Fix |
|---|---|---|
| Feed downloads failed: `Temporary failure in name resolution` | Kali's internet/DNS was down during setup | Confirmed connectivity with `ping`, then re-ran the feed sync |
| `database "gvmd" has a collation version mismatch` | Installing other tools upgraded Kali's core C library (glibc 2.42 → 2.43), and the database was built under the old one | `REINDEX DATABASE gvmd;` and `ALTER DATABASE gvmd REFRESH COLLATION VERSION;` |
| Setup printed no admin password | An admin user already existed from an earlier install | Set a new one with `gvmd --user=admin --new-password=…` |
| `gvm-feed-update: command not found` | Renamed in newer GVM versions | Used `greenbone-feed-sync` |
| Login failed after pausing and resuming the VM | The long-running manager lost its database connection during the pause | `gvm-stop` / `gvm-start`, then reset the password |
| Start button greyed out for hours | The web interface blocks scans while the CVE database rebuilds | Started the task through the GMP API with `gvm-cli` |

## Skills demonstrated

- Installing, verifying and troubleshooting a full vulnerability-management stack
- Configuring scan targets and tasks (port lists, alive tests, scan configs, quality-of-detection threshold)
- Reading a vulnerability report: CVSS scores, CVEs, QoD, solution types
- Comparing scanner results with an earlier port scan and explaining the differences
- Using an API (GMP) when a web interface is blocked
- Turning raw findings into prioritized remediation
