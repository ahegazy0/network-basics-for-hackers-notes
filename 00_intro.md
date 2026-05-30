# Network Hacking Protocols: Analysis and Exploitation

> *Based on **Network Basics for Hackers** by OccupyTheWeb (Master OTW) - InfoSec Press, 2023*

---

## Welcome

This is a hacker's tour through the protocols that run the internet. Not the "networking for sysadmins" kind. The kind where you learn why these protocols exist, how they work under the hood, and exactly where they break.

No exam prep. No corporate fluff. Just real networking from an attacker's perspective.

---

## What This Is About

Every device on a network, every email you send, every DNS lookup your browser makes quietly in the background - all of it runs on protocols. And every single one of those protocols has weaknesses.

Most people learn networking to *use* it. We're here to learn how to *break* it.

OTW wrote this book the same way he wrote *Linux Basics for Hackers* - start with the fundamentals, build real stuff on Linux, then go through the vulnerabilities of each protocol. That's the structure we're following.

---

## What We'll Cover

| Area | Topics |
|------|--------|
| Networking Fundamentals | TCP/IP, OSI model, IP addressing, subnetting |
| Packet Analysis | Wireshark, tcpdump, reading raw traffic |
| Core Protocols | DNS, ARP, SMTP, HTTP and their attack surfaces |
| Wireless | Wi-Fi (802.11), Bluetooth |
| Advanced Networks | SCADA/ICS, Car Networks, Radio Frequency, Mobile |
| Linux Applications | Building BIND (DNS), EXIM (mail) servers from scratch |

---

## Who This Is For

Beginner to intermediate. If you've read *Linux Basics for Hackers* or you're comfortable in a Linux terminal, you're in the right place. No deep networking background needed.

If you don't know what a packet is yet - that's fine. You will.

---

## The Approach

Most networking courses teach you to pass a cert. This teaches you to think like an attacker. That means:

- Understanding **why** a protocol was designed the way it was
- Knowing **what assumptions** designers made, and why those assumptions become vulnerabilities
- Being able to **read traffic**, spot anomalies, and understand what an exploit looks like on the wire

Theory first. Hands-on Linux second. Exploitation third.

---

*Let's get into it.*
