# Course Introduction - Network Basics for Hackers

---

## Overview

This guide is an attacker-focused tour through the protocols that run modern networks. Instead of administrative cert-prep, it focuses on why these protocols exist, how packet headers work on the wire, and where design assumptions create exploitable vulnerabilities.

Every device on a network, every connection your browser makes, and every packet passing your router runs on protocols with inherent trade-offs. Most people learn networking to use it; security professionals learn it to understand where it breaks.

---

## What We Cover

| Area | Key Topics | Practical Skills |
|---|---|---|
| **Networking Fundamentals** | TCP/IP, OSI model, IPv4 addressing, CIDR subnetting | Network mapping, subnet math |
| **Traffic & Packet Analysis** | Wireshark, tcpdump, raw socket dissection | Packet decoding, stream inspection |
| **Core Protocol Security** | ARP, DNS, SMB, SMTP, SNMP, HTTP/HTTPS | MITM attacks, cache poisoning, relaying |
| **Wireless & RF** | Wi-Fi 802.11, Bluetooth, SDR / Radio signals | Frame injection, monitor mode, spectrum analysis |
| **Specialized Architectures** | Automobile CAN Bus, SCADA / Industrial ICS | Bus sniffing, ICS simulation, Modbus queries |

---

## Lab Environment & Prerequisites

* **Prerequisites:** Basic comfort with the Linux terminal (navigating directories, inspecting text files, running root commands).
* **Environment:** Kali Linux running in a Virtual Machine (VirtualBox / VMware) or booted from a Live USB.

> 💡 **Lab Hardware Tip:** You don't necessarily need an external USB Wi-Fi dongle to start wireless labs. Booting **Kali Linux from a Live USB** often gives the operating system direct hardware access to laptop integrated wireless cards, enabling monitor mode and packet injection without hypervisor USB pass-through issues.

---

## The Core Approach

1. **Understand Protocol Design:** Why was this protocol built this way?
2. **Inspect Traffic on the Wire:** What does normal communication look like in Wireshark and tcpdump?
3. **Identify Weak Assumptions:** Why does a lack of authentication or encryption lead to exploitation?

---

*Up next: Module 01 - Network Basics (TCP/IP, OSI, Ports & Nmap)*
