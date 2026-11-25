# Task 3: Memory Dump Analysis (Volatility 3)

> **Authorized training exercise.** The evidence is a **training memory image of a Windows 7 machine, supplied by my cybersecurity course** for forensic practice. Analysis was done in my own Kali Linux VM. Kali was given temporary internet access (VirtualBox NAT) only to download the evidence and Volatility's Windows symbol files; the vulnerable lab target stayed isolated. Course names, download links and the lab's planted answer string are **redacted**.

## Why memory forensics?

A computer's RAM holds evidence that never touches the disk: which programs were running, which network connections were open, what was typed or browsed moments ago. When the machine is powered off, all of it disappears. Investigators capture a **memory dump** (a snapshot of RAM) so they can examine that evidence later without changing the original system.

This task treats the image as an investigation and answers five questions.

| # | Investigation question |
|---|---|
| Q1 | Retrieve the process list |
| Q2 | Provide evidence of active network connections |
| Q3 | Extract the Google Chrome browser history |
| Q4 | Provide evidence of suspicious search terms |
| Q5 | Provide evidence of hacking tools (collect the files as-is, without opening them) |

## Environment

| Component | Detail |
|---|---|
| Analyst VM | Kali Linux (VirtualBox) |
| Tool | Volatility 3 Framework 2.28.2 (installed with pipx) |
| Helpers | gdown (download), sqlite3 (read Chrome's database), sha256sum (integrity) |
| Evidence | `memdump.raw`, 2.0 GB raw memory image (renamed from the course filename) |
| Evidence SHA-256 | `97e83bff324df59e371efbe0309cac086cc8e66c746a10e64c87cc9f4c38bf28` |

## Setup and evidence handling

```bash
pipx install volatility3          # Kali's apt repo had no volatility3 package
pipx install gdown                # downloads large Google Drive files from the terminal
gdown <file-id>                   # course-provided evidence (ID redacted)
unzip <evidence>.zip
mv <evidence>.raw memdump.raw     # generic name, keeps the course name out of later output
sha256sum memdump.raw             # fingerprint the evidence BEFORE analysis
vol -f memdump.raw windows.info   # identify the OS; first run downloads Windows symbols
```

**System profile (`windows.info`):** Windows 7 SP1 (build 7601), 32-bit with PAE, 4 CPUs. Memory captured **2022-09-26 14:17:38 UTC**.

---

## Q1: Process list

```bash
vol -f memdump.raw windows.pslist | tee pslist.txt
vol -f memdump.raw windows.pstree | tee pstree.txt
```

**Finding.** The boot chain is normal (`System → smss → csrss/wininit → services → svchost`, then `winlogon → userinit → explorer` at logon). Processes worth noting, all launched by `explorer.exe`:

| Process | PID | Started (UTC) | Assessment |
|---|---|---|---|
| chrome.exe (+ child processes) | 3456 | 14:13:24 | Browser in use (source for Q3/Q4) |
| cmd.exe | 1052 | 14:14:06 | User opened a command prompt shortly before capture |
| MRCv120.exe | 1140 | 14:17:19 | Run from `Z:\`, 19 seconds before capture. Most likely **Magnet RAM Capture**, the tool that created this dump, so it belongs to the investigator. |
| sshd.exe (OpenSSH) | 556 | 13:34:22 | SSH server running as a service on a desktop. Unusual, noted. |

No hacking tool was running at capture time, which pointed the investigation toward files and history (Q3–Q5).

**What I learned.** `pslist` is the official process list; `pstree` shows parent → child relationships, which is how you spot something launched by the wrong parent. A tool missing from the process list doesn't mean it was never used; it may have already exited.

## Q2: Active network connections

```bash
vol -f memdump.raw windows.netscan | tee netscan.txt
```

**Finding.** Host IP was **172.16.0.5**.

- **ESTABLISHED** outbound connections went to Google address ranges (142.250.x.x and 172.217.164.131 on port 443, plus 34.104.35.123 on port 80, a Google download/update server). This matches Chrome being in use.
- An SMB connection (port 445) from the machine to its own IPv6 address, most likely the mapped `Z:` drive the capture tool ran from.
- **LISTENING** ports: 22 (SSH), 445/139 (Windows file sharing), 135 and 1025–1029 (Windows RPC), 5355/1900/3702 (local discovery).
- No inbound SSH session and no unknown foreign IPs at capture time.

**What I learned.** `ESTABLISHED` means an active conversation; `LISTENING` means a service waiting for connections. On Windows 7 images, Volatility often can't attribute TCP connections to a process (owner shows `-`), so you corroborate with other evidence (here, Chrome's own UDP socket and its history).

## Q3: Chrome browser history

```bash
vol -f memdump.raw windows.filescan > filescan.txt
grep -i "chrome" filescan.txt | grep -i "history"
vol -f memdump.raw -o chrome windows.dumpfiles --physaddr 0x7e87ecc8
file chrome/*                                   # → SQLite 3.x database
cp chrome/*History.dat History.db               # work on a copy
sqlite3 -readonly -header -column History.db \
  "SELECT datetime(last_visit_time/1000000-11644473600,'unixepoch') AS visited_utc,
          visit_count AS visits, substr(url,1,100) AS url
   FROM urls ORDER BY last_visit_time;"
```

**Finding.** Chrome's `History` database (196 KB) was recovered intact from memory for user **IEUser**. Timeline (UTC):

| When | Activity |
|---|---|
| 2022-08-01 20:39–20:40 | Searches about the dark web, selling data, and Tor Browser; visit to a Tor Browser download page |
| 2022-08-01 21:06–21:07 | Search "hacking tutorials"; visits to **nmap.org** (Windows download page) and **hashcat.net** |
| 2022-08-17 10:13–10:17 | Search "filezilla" → FileZilla download page; search "mock data" → mockaroo.com |
| 2022-09-23 15:52 | Local file opened in Chrome: `X:/MOCK_DATA.pdf` |
| 2022-09-26 14:13–14:15 | Two searches containing the lab's planted answer *(redacted)* |

**What I learned.** Chrome stores history in SQLite, so a file recovered from RAM can be queried directly. Chrome timestamps are microseconds since 1601-01-01, which is why the query converts them.

## Q4: Suspicious search terms

```bash
sqlite3 -readonly -header -column History.db \
  "SELECT datetime(u.last_visit_time/1000000-11644473600,'unixepoch') AS searched_utc, k.term
   FROM keyword_search_terms k JOIN urls u ON k.url_id=u.id ORDER BY u.last_visit_time;"
```

**Finding.** Suspicious terms:

- *dark web browser*
- *how to sell secret data on the dark web*
- *how much should i sell my data for on the dark web*
- *what to do when you're rich from selling your data on the dark web*
- *www tor browser*
- *hacking tutorials*
- *filezilla*, *mock data*

The other searches (memes, Mandalorian pictures) are ordinary browsing.

**What I learned.** Search terms reveal **intent**, which tool artifacts alone can't. Here they describe a plan: obtain data, move it out, and sell it anonymously. That's an **insider threat** pattern.

## Q5: Hacking tool evidence

```bash
# what Chrome downloaded
sqlite3 -readonly -header -column History.db \
  "SELECT datetime(start_time/1000000-11644473600,'unixepoch') AS started_utc,
          received_bytes AS bytes, target_path FROM downloads ORDER BY start_time;"
# which tool files are present in memory
grep -iE "hashcat|nmap|filezilla|tor ?browser" filescan.txt | tee tools_found.txt
# collect the evidence files as-is (never opened or executed)
vol -f memdump.raw -o tools windows.dumpfiles --physaddr 0x7f339268   # hashcat.exe
vol -f memdump.raw -o tools windows.dumpfiles --physaddr 0x7f4644a0   # nmap.exe
vol -f memdump.raw -o tools windows.dumpfiles --physaddr 0x7f539d08   # nmap Prefetch
sha256sum tools/*
```

**Downloads (Chrome):**

| Started (UTC) | File | Bytes |
|---|---|---|
| 2022-08-01 21:07:00 | `hashcat-6.2.5 (1).7z` (password cracker) | 8,652,614 |
| 2022-08-01 21:07:16 | `nmap-7.92-setup (1).exe` (network scanner) | 28,644,568 |
| 2022-08-17 10:13:52 | `FileZilla_3.60.2_win32-setup.exe` (FTP client) | 12,130,464 |
| 2022-08-17 10:17:24 | `MOCK_DATA.csv` | 31,439 |

**Tool artifacts found in memory:**

| Artifact | Offset | Meaning |
|---|---|---|
| `\Program Files\Nmap\nmap.exe` (+ zenmap, ncat, nping, NSE scripts) | 0x7f4644a0 | Nmap installed |
| `\Users\IEUser\Desktop\Nmap - Zenmap GUI.lnk`, `zenmap.exe.log` | | Shortcut created; Zenmap GUI was opened |
| `\Windows\Prefetch\NMAP.EXE-A4293E59.pf` | 0x7f539d08 | **Windows recorded that nmap.exe was executed** |
| `\Users\IEUser\Downloads\hashcat-6.2.5\hashcat.exe` | 0x7f339268 | Hashcat extracted and ready to use |
| `\Program Files\FileZilla FTP Client\filezilla.exe` | 0x7ef6f4e8 | FileZilla installed |

**Collected evidence (SHA-256):**

| File | SHA-256 |
|---|---|
| hashcat.exe (DataSectionObject) | `2fc806d273cbfb03b5873278f70d1e24ba7846ec35f2e23124ce466056e52feb` |
| nmap.exe (DataSectionObject) | `b7c51b9111f59930abbf3fb414a8ad60f7bcfec49af8f32bdc5786524202f94c` |
| nmap.exe (ImageSectionObject, as loaded to run) | `2174a2342f5d7c45ce0495aa172a6812e614c79e79a5a6f1395a3bba7f61e0aa` |
| NMAP.EXE-A4293E59.pf (partial) | `ac7202c38584019c3cf4c411e7d9e9a51d5a1c309e27e51e7d75903a96d3591e` |

**What I learned.** A **Prefetch** file is Windows' own record that a program ran, which makes it strong proof of execution. Files recovered from memory can be partial (only pages that were in RAM), so you record exactly what you collected with hashes and never open or run suspect binaries.

---

## Conclusion: reconstructed timeline

1. **Aug 1:** user researches selling data on the dark web and Tor Browser, then hacking tutorials; downloads **hashcat** and **nmap** within one minute.
2. **Nmap** is installed and **executed** (Prefetch); **hashcat** is extracted.
3. **Aug 17:** user downloads **FileZilla** (a way to move files off the machine) and a data file.
4. **Sep 26, 14:17:38 UTC:** memory is captured with Magnet RAM Capture.

The evidence is consistent with an **insider preparing to steal and sell data**: reconnaissance and password-cracking tools, an exfiltration tool, and searches showing intent.

## Defensive takeaways

1. **Application allow-listing** (AppLocker, WDAC) would block unapproved tools like nmap and hashcat from running.
2. **Monitor endpoints** (EDR) for hacking-tool installs and executions, and alert on them.
3. **Web filtering / DLP**: block or flag Tor and dark-web activity; watch for FTP uploads leaving the network.
4. **Capture memory early** in an incident: much of this evidence existed only in RAM.

## Troubleshooting log

| Problem | Cause | Fix |
|---|---|---|
| `vol: command not found` | Volatility not preinstalled | Installed it |
| `volatility3 has no installation candidate` | Not in Kali's apt repo at the time | Installed from PyPI with `pipx install volatility3` |
| Kali couldn't download evidence or symbols | Lab VM was host-only (offline) | Added a NAT adapter to Kali only |
| `sSpipx: command not found` | Stray keystrokes before the command | Retyped the command |
| `sha256sum` appeared to hang | Run with no filename, so it waited for keyboard input | Ctrl+C, re-ran with `tools/*` |
| `grep ... tor` returned hundreds of lines | "tor" matches *History*, *Directory*, *Storage* | Narrowed the pattern to `tor ?browser` |
| Prefetch dump: "Error dumping file" | Only part of the file was resident in RAM | Recorded the partial file's hash; its presence in `filescan` is still evidence |

## Skills demonstrated

- Memory acquisition handling: evidence hashing, working on copies, read-only access
- Volatility 3: `windows.info`, `pslist`, `pstree`, `netscan`, `filescan`, `dumpfiles`
- Browser forensics: extracting and querying Chrome's SQLite history
- Windows artifacts: Prefetch, shortcuts, section objects
- Building an incident timeline and recognizing an insider-threat pattern
- Troubleshooting tool installation and documenting root causes

> 📄 The full step-by-step report with screenshots is in this folder: `Task3_Memory_Forensics_Report.pdf`.
