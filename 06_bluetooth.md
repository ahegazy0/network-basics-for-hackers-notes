# Network Basics for Hackers
## Module 06 - Bluetooth Security

---

## Overview

Bluetooth is a ubiquitous short-range wireless protocol operating in the 2.4 GHz ISM band. Despite its limited physical radius, Bluetooth presents a significant attack surface across mobile phones, laptops, automobiles, and IoT sensors. This module explores Bluetooth pairing mechanisms, service discovery via SDP, legacy protocol exploits like bluesnarfing and bluebugging, and modern stack vulnerabilities such as BlueBourne.

---

## Why Bluetooth Gets Ignored

Bluetooth has a reputation for being "safe" because of its short range. People think if someone has to be within 10 meters, it's not a real threat. That thinking is wrong.

Bluetooth is on your phone, laptop, car, smartwatch, earbuds, medical devices, keyboards, and dozens of other things you carry around every day. Most of those devices are always on, always listening for connections, and running firmware that hasn't been updated in years.

The short range isn't much protection either. High-gain directional antennas can extend Bluetooth range to 100 meters or more. In an airport, a mall, or an office building, "short range" covers a lot of people.

---

## How Bluetooth Works

Bluetooth operates on the 2.4 GHz band, the same as Wi-Fi. To avoid interference it uses **frequency hopping** - it jumps between 79 different channels up to 1600 times per second. This makes passive interception harder compared to Wi-Fi, but it doesn't make the protocol attack-proof.

When two Bluetooth devices connect, they form a small network called a **Piconet**. One device is the master, the others are slaves. Up to 7 active devices can be in a piconet at once.

```
         [Master Device]
        /        |        \
  [Slave 1]  [Slave 2]  [Slave 3]
       <- Piconet ->
```

When devices pair for the first time, they exchange a shared secret key. That key is used to encrypt future communications. The security of this pairing process has changed a lot across Bluetooth versions - older versions had serious weaknesses in how keys were generated.

---

## Bluetooth Modes

Before scanning or attacking, it helps to know the different visibility states:

| Mode | What it means |
|------|-------------|
| Discoverable | Device broadcasts its presence, anyone can find it |
| Non-discoverable | Not broadcasting, but still connectable if you know the address |
| Pairable | Accepts new pairing requests |
| Non-pairable | Won't accept new pairings, only talks to already-paired devices |

Most people assume non-discoverable means invisible. It doesn't. If you already know the device's Bluetooth MAC address, you can still connect to it. And tools like BlueBourne don't need discoverability at all.

---

## Basic Recon Commands

```bash
# Check your Bluetooth interface (like ifconfig but for Bluetooth)
ahegazy0@kali:~$ hciconfig

# Bring the interface up
ahegazy0@kali:~$ sudo hciconfig hci0 up

# Scan for discoverable devices in range
ahegazy0@kali:~$ hcitool scan

# Get more info about a specific device
ahegazy0@kali:~$ hcitool info <MAC_ADDRESS>

# Scan for low energy (BLE) devices
ahegazy0@kali:~$ sudo hcitool lescan

# Check what services a device is running (Service Discovery Protocol)
ahegazy0@kali:~$ sdptool browse <MAC_ADDRESS>
```

`hcitool scan` only finds devices in discoverable mode. That's a limited view. Most real-world Bluetooth recon involves additional tools that go beyond basic discovery.

```bash
# btscanner - GUI tool for finding and fingerprinting BT devices
ahegazy0@kali:~$ sudo btscanner

# bluelog - passive Bluetooth scanner, logs everything it finds
ahegazy0@kali:~$ sudo bluelog -i hci0 -o scan_results.txt
```

---

## Attack Types

### Bluejacking
Sending unsolicited messages to a Bluetooth device. More annoying than harmful - nobody can steal anything, but it proves you can reach the device. Old attack, rarely relevant today.

### Bluesnarfing
Actually stealing data from a device without permission and without the owner knowing. Contacts, messages, calendar entries, call logs. This works by exploiting vulnerabilities in the Object Push Profile (OPP) or other Bluetooth profiles that devices expose.

The victim sees nothing. No notification, no indication. The connection happens silently.

### Bluebugging
Goes further than bluesnarfing. Full remote control of the device. Make calls, send messages, access the internet through the victim's phone, listen through the microphone. The device basically becomes a remote tool.

Both bluesnarfing and bluebugging typically require the target to be running old, unpatched firmware. Modern devices with up-to-date software are significantly harder to attack this way.

### BlueBourne
This one is different and worth understanding separately. BlueBourne is a collection of vulnerabilities discovered in 2017 that affected billions of devices across Android, iOS, Windows, and Linux.

What makes it different: it doesn't require the target to be in discoverable mode, doesn't require pairing, and the victim doesn't have to do anything. If your Bluetooth is on, you're potentially vulnerable.

The attack works through vulnerabilities in the Bluetooth stack at the OS level - specifically in how devices handle certain protocol messages before any pairing or authentication happens.

```bash
# BlueBourne scanning and exploitation framework
# https://github.com/ArmisSecurity/blueborne

ahegazy0@kali:~$ python blueborne.py -i hci0 scan
```

BlueBourne is patched on updated devices, but an enormous number of embedded systems, older phones, and IoT devices never received the patches.

---

## Bluetooth Versions and Security

The security of Bluetooth has evolved a lot. Older versions have serious weaknesses.

| Version | Key Security Issue |
|---------|------------------|
| BT 1.x / 2.0 | Weak PIN-based pairing, easily brute-forced |
| BT 2.1 | Added Secure Simple Pairing (SSP), better but still has issues |
| BT 4.x (BLE) | New LE protocol, different attack surface, MITM possible |
| BT 5.x | Improved range and speed, security improvements but still evolving |

BLE (Bluetooth Low Energy) is a different protocol from classic Bluetooth. It runs on the same chip in most modern devices but has its own set of tools and attack methods. Most IoT devices - fitness trackers, smart locks, sensors - use BLE.

---

## SDP - Service Discovery Protocol

This is worth knowing because it's a common way to fingerprint what a Bluetooth device supports.

SDP runs on every Bluetooth device and tells other devices what services are available - audio streaming, file transfer, serial port, etc. You can query it without pairing:

```bash
# Browse all services on a target device
ahegazy0@kali:~$ sdptool browse --tree <MAC_ADDRESS>

# Search for a specific service
ahegazy0@kali:~$ sdptool search --bdaddr <MAC_ADDRESS> <SERVICE_NAME>
```

The service list tells you a lot. A device exposing a OBEX Object Push service is potentially vulnerable to bluesnarfing. A device with a Serial Port Profile might accept AT commands. Each exposed service is a potential attack surface.

---

## Defending Against Bluetooth Attacks

- **Turn Bluetooth off when you don't need it.** Especially in airports, conferences, crowded public places. If it's off, it can't be attacked.
- **Keep firmware updated.** BlueBourne and similar vulnerabilities were patched. Unpatched devices are the ones still at risk.
- **Use non-discoverable mode** when Bluetooth needs to stay on but you're not actively pairing.
- **Don't accept pairing requests from unknown devices.**
- **Avoid Bluetooth in high-risk environments** like security conferences (DEF CON has a long history of Bluetooth attacks in the hallways).

---

## Quick Reference

| Tool | What it does |
|------|-------------|
| `hciconfig` | Manage Bluetooth interface |
| `hcitool scan` | Scan for discoverable devices |
| `hcitool lescan` | Scan for BLE devices |
| `hcitool info <MAC>` | Get info on a specific device |
| `sdptool browse <MAC>` | List services on a device |
| `btscanner` | GUI tool for device discovery and fingerprinting |
| `bluelog` | Passive Bluetooth scanner and logger |

---

## Practice

```bash
# 1. Check your Bluetooth interface
ahegazy0@kali:~$ hciconfig

# 2. Bring interface up if down
ahegazy0@kali:~$ sudo hciconfig hci0 up

# 3. Scan for discoverable devices
ahegazy0@kali:~$ hcitool scan

# 4. If a test device is discovered, inspect exposed SDP profiles
ahegazy0@kali:~$ sdptool browse <MAC_ADDRESS>

# 5. Scan for low energy (BLE) advertisements
ahegazy0@kali:~$ sudo hcitool lescan
```

- [ ] Check local Bluetooth hardware using `hciconfig` and ensure the interface is UP.
- [ ] Perform a discoverable device inquiry with `hcitool scan` against a controlled mobile device.
- [ ] Query SDP service records with `sdptool browse <MAC>` to enumerate supported Bluetooth profiles.
- [ ] Run `sudo hcitool lescan` to observe nearby BLE advertising beacons.
- [ ] Explain why setting a Bluetooth device to non-discoverable mode does not protect it against stack vulnerabilities like BlueBourne if its MAC address is known.

> 💡 *For deeper practice, I also recommend completing the end-of-chapter exercises in the official **Network Basics for Hackers** book.*

---

*Up next: Module 07 - ARP (Address Resolution Protocol)*
