# Case 2: FTP Data Theft Investigation (Wireshark)

> **Authorized training exercise.** The packet capture (`suspicious_activity.pcapng`) was **supplied by my cybersecurity course** for analysis practice and examined in Wireshark on my own Kali Linux VM. The stolen file contains **generated, fake personal data**. Even so, I treat it like real personal data: every data row is blurred in the screenshots, and the file itself is **not published here**. Only its column names and SHA-256 fingerprint appear.

## The scenario

A capture of network traffic between two machines was flagged as suspicious. The job: work out who was involved, how they got in, which connection carried data out, and what was taken.

| # | Question | Answer |
|---|---|---|
| 1 | Which two hosts were involved? | **Client 10.0.2.19** (the attacker) and **server 10.0.2.20** (FileZilla FTP server) |
| 2 | What credentials were used? | Username **`anonymous`**, **blank password**. The server allowed anonymous login. |
| 3 | Which port carried the stolen data? | **50082** (not the usual FTP port 21) |
| 4 | What data was taken? | **`employee_database.csv`**: 1,000 employee records with names, emails, birthdays, Social Security numbers, credit card numbers, addresses and phone numbers |

## Environment

| Item | Detail |
|---|---|
| Analyst VM | Kali Linux (VirtualBox) |
| Tool | Wireshark |
| Evidence | `suspicious_activity.pcapng` (161 KB, 337 packets) |
| Recovered file | `employee_database.csv`, SHA-256 `cbe10a41e5215218f82730546f5dfbe02a87d3daab2c1a671ab7e00e38a70319` |

---

## Q1: Identify the two hosts

**How:** *Statistics → Conversations → TCP* lists every pair of machines that talked.

**What it showed:**

- About **190 one-packet connections** from `10.0.2.19` (always from source port 44537) to `10.0.2.20` on a different port each time: 21, 22, 80, 135, 445, 3306, and so on. That pattern is a **port scan**: the attacker knocking on every door to see which ones open.
- Only three connections carried real traffic:

| Stream | From → To | Purpose |
|---|---|---|
| 200 | 10.0.2.19:50080 → 10.0.2.20:**21** | FTP login and commands |
| 201 | 10.0.2.20:49771 → 10.0.2.19:50081 | Folder listing |
| 202 | 10.0.2.20:49772 → 10.0.2.19:**50082** | **The stolen file** |

**Answer:** `10.0.2.19` sent the first `SYN` (the opening "knock") to port 21, so it is the **client**. `10.0.2.20` answered as the **FTP server**.

**What I learned.** Whoever sends the first `SYN` of a TCP connection started it. That's how you tell client from server. A long run of one-packet connections to many ports from one source is a port scan, and it often comes right before an attack.

## Q2: Identify the credentials

**How:** display filter `ftp`, then right-click → *Follow → TCP Stream* on the login connection (stream 200).

**What it showed** (the whole FTP conversation, readable as plain text):

```
220-FileZilla Server 1.5.1
USER anonymous
331 Please, specify the password.
PASS
230 Login successful.
PORT 10,0,2,19,195,161     LIST
PORT 10,0,2,19,195,162     RETR employee_database.csv
PORT 10,0,2,19,195,163     LIST
QUIT
```

**Answer:** username **`anonymous`** with **no password**, and the server accepted it.

**What I learned.** FTP sends logins as plain text, so anyone watching the network can read them. Worse, this server allowed **anonymous FTP**: no real account was needed at all.

## Q3: Which port carried the data out?

FTP uses **two kinds of connections**: one for commands (port 21), and a separate one for each file transfer. The `PORT` command tells the server where to send the data. Its last two numbers make up the port:

```
PORT 10,0,2,19,195,162   →   195 × 256 + 162 = 50082
```

That `PORT` command came right before `RETR employee_database.csv`, and stream 202 runs to port 50082.

**Answer:** **50082.**

**What I learned.** In "active mode" FTP, the **server opens the data connection back to the client**. So the theft traveled server → client on a high, changing port, which a firewall rule that only watches port 21 would miss.

## Q4: What data was taken?

**How:** filter `tcp.stream eq 202` → *Follow → TCP Stream*, plus the **packet bytes** pane on the largest packet (frame 276, 29,756 bytes).

**What it showed:** the contents of `employee_database.csv`. Its first line holds the column names:

```
id,first_name,last_name,email,gender,birthday,social_security,credit_card,street_address,city,state,phone_number
```

…followed by **1,000 rows** of personal data (blurred in the report). About 130 KB in total.

**Saving the evidence.** *File → Export Objects → FTP-DATA* came up empty: Wireshark labeled the data connection as plain TCP because it didn't link port 50082 to the FTP session. Instead I saved it from *Follow TCP Stream* with **Show as: Raw → Save as…**, which keeps every byte exactly, then fingerprinted it:

```bash
sha256sum employee_database.csv   # cbe10a41e521…e38a70319
wc -l employee_database.csv       # 1001 lines = column names + 1,000 records
```

**Answer:** **`employee_database.csv`**, 1,000 employee records including Social Security and credit card numbers.

**What I learned.** Data sent over unencrypted FTP can be rebuilt completely from a capture. Saving it as **Raw** and recording its hash lets you prove exactly what left the network.

---

## Reconstructed timeline (seconds from start of capture)

1. **0–1.3 s:** `10.0.2.19` port-scans `10.0.2.20` and finds FTP open on port 21.
2. **13.19 s:** connects to the FTP server (FileZilla Server 1.5.1).
3. **15.77–16.85 s:** logs in as `anonymous` with no password and is accepted.
4. **17.81 s:** lists the folder contents.
5. **32.33 s:** requests `employee_database.csv`; the server sends it on port 50082 in about 0.01 s.
6. **34.69 s:** lists the folder again, then quits.

**Assessment:** a quick, deliberate theft. Scan, log in through a misconfigured server, look around, take the one valuable file, leave.

## Impact

1,000 people's Social Security numbers, card numbers and contact details were exposed. In a real company this is a reportable data breach (U.S. state breach-notification laws, PCI DSS for the card data).

## How to prevent it

1. **Turn off anonymous FTP.** Require real accounts with strong passwords.
2. **Replace FTP with SFTP or FTPS** so logins and files are encrypted.
3. **Don't keep sensitive files on file-sharing servers,** and encrypt files that hold personal data.
4. **Firewall the FTP server** so only approved machines can reach it.
5. **Alert on port scans and unusual outbound transfers** (IDS/IPS, data loss prevention).

## Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| *Export Objects → FTP-DATA* list was empty | Wireshark treated the port-50082 connection as plain TCP | Saved the file from *Follow TCP Stream* using **Raw** |

## Skills demonstrated

- Reading TCP conversations and handshakes to identify client and server
- Spotting a port scan in traffic
- Recovering FTP logins and commands with *Follow TCP Stream*
- Working out FTP data ports from `PORT` commands
- Rebuilding a stolen file from packets and proving it with a SHA-256 hash
- Handling personal data responsibly in a public report

> 📄 Full step-by-step report with screenshots: `Case2_FTP_Data_Theft_Report.pdf` in this folder.
