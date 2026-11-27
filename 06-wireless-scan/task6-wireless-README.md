# Task 6: Wireless Network Scan (airodump-ng)

> **Passive scan only.** This task only *listened* to WiFi signals already in the air, to list the networks within range. Nothing was connected to, captured, attacked or disrupted. Network names (ESSIDs) and hardware addresses (BSSIDs) of nearby networks are **blurred**: they belong to my neighbors, and some names would reveal the location.

## Objective

Use a USB WiFi adapter in **monitor mode** with **airodump-ng** to survey the wireless networks within range, and read what the survey shows about their security.

## Equipment

| Item | Detail |
|---|---|
| Adapter | Panda Wireless PAU04 (USB, 2.4 GHz, 802.11n) |
| Chipset | **Ralink RT5370** (USB ID `148f:5370`), Linux driver `rt2800usb`, built into Kali |
| Host | Kali Linux in VirtualBox, adapter passed through with *Devices → USB* |
| Tools | `lsusb`, `iw`, `airmon-ng`, `airodump-ng` (Aircrack-ng suite) |

**Why an external adapter?** A laptop's built-in WiFi can't be put into monitor mode from inside a VM: VirtualBox gives the VM a virtual *wired* card. A USB adapter can be handed to Kali directly, and this chipset supports monitor mode.

## Method

```bash
lsusb
```
Lists USB devices. Confirmed Kali sees the adapter: `Ralink Technology, Corp. RT5370 Wireless Adapter`.

```bash
iw dev
```
Lists wireless interfaces and their mode. Showed `wlan0` in `managed` (normal) mode.

```bash
sudo airmon-ng check kill
```
Stops background programs that would pull the card out of monitor mode. Here it stopped `wpa_supplicant`, the program that logs in to WiFi networks.

```bash
sudo airmon-ng start wlan0
```
Switches the card into **monitor mode**, where it listens to all WiFi traffic instead of joining one network. Created `wlan0mon`.

```bash
sudo airodump-ng --band bg -w ~/wifiscan --output-format csv wlan0mon
```
Hops across the 2.4 GHz channels (`--band bg`), shows a live table of networks, and saves a log (`wifiscan-01.csv`). Ran for about one minute.

```bash
sudo airmon-ng stop wlan0mon
sudo systemctl start NetworkManager
```
Returns the card to normal mode and restores Kali's network connection.

## Results

**38 access points** were seen in one minute.

| Security | Count | What it means |
|---|---|---|
| WPA2, PSK | 19 | Typical home WiFi: one shared password for everyone |
| WPA2, MGT | 8 | **Enterprise** WiFi: each user signs in with their own account (802.1X / RADIUS) |
| WPA3, SAE | 8 | Newest home WiFi security; resists offline password guessing |
| **Open (OPN)** | **3** | **No encryption**: traffic not otherwise protected (e.g. by HTTPS) can be read by anyone nearby |

- **11 networks hid their name** (`<length: 0>`). Hiding the name doesn't hide the network: a monitor-mode scan still sees it, and its name is revealed as soon as a device connects.
- **Channels:** most networks use **1, 6 and 11**, the only three 2.4 GHz channels that don't overlap. Six use **2, 5, 7 or 9**, which overlap their neighbors and cause interference for everyone nearby.
- **Signal strength (PWR)** ranged from about −33 dBm (very close) to −81 dBm (barely in range).

## Analysis

- **Open networks are the biggest risk** in this survey. Users on them are exposed to eavesdropping and fake look-alike hotspots ("evil twins"). Over open WiFi, rely on HTTPS or a VPN.
- **WPA3 adoption is growing** (8 of 38), but WPA2 with a shared password is still the most common. Its strength depends entirely on how long and random that password is.
- **Enterprise networks (MGT)** are the right choice for organizations: individual logins can be revoked one at a time, unlike a shared password.
- **Hidden names are not a security control.** They add a little obscurity but no protection.
- **Badly chosen channels** (2, 5, 7, 9) are a common cause of "slow WiFi" complaints. Moving to 1, 6 or 11 is a quick help-desk fix.

## Recommendations (for a network owner)

1. Use **WPA3**, or WPA2/WPA3 mixed mode, with a long random password. Disable WPS.
2. Use **WPA2/WPA3-Enterprise (802.1X)** for business networks.
3. Avoid open networks, or isolate guests and require a captive portal plus HTTPS.
4. Put 2.4 GHz access points on channel **1, 6 or 11**, and prefer 5 GHz where possible.
5. Don't count on a hidden network name for security.

## Limitations

- The adapter is **2.4 GHz only**, so 5 GHz networks (common today) weren't seen.
- One minute from one spot is a snapshot. Networks with weak signals or few beacons may have been missed.

## Skills demonstrated

- Passing USB hardware through to a virtual machine
- Identifying a wireless chipset and its Linux driver
- Monitor mode vs. managed mode
- Surveying networks with airodump-ng and reading the results (encryption, authentication, channel, signal)
- Applying WiFi security concepts (WPA2/WPA3, PSK vs. Enterprise, open networks) to real data
- Protecting third-party information in a public report

> 📄 Step-by-step report with screenshots: `Task6_Wireless_Scan_Report.pdf` in this folder.
