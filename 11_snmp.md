# Network Basics for Hackers
## Module 11 - SNMP (Network Management Protocol)

---

## Overview

The Simple Network Management Protocol (SNMP) is widely deployed across routers, switches, servers, and embedded appliances for remote telemetry and configuration. Because legacy versions (SNMPv1 and SNMPv2c) authenticate queries using unencrypted, default "community strings" (such as `public` and `private`), exposed agents provide attackers with extensive intelligence - including routing tables, usernames, running processes, and installed software.

---

## What SNMP Does

SNMP (Simple Network Management Protocol) is how network administrators monitor and manage devices remotely. Routers, switches, servers, printers, UPS units, firewalls - anything networked can potentially run an SNMP agent.

Through SNMP an admin can:
- Check CPU and memory usage on a router
- See how much traffic is flowing through a switch port
- Read the system uptime of a server
- Get a list of installed software on a machine
- In some configurations, change device settings remotely

That last part is important. SNMP isn't just read-only monitoring. With write access, you can modify device configurations. That's a lot of power sitting behind what is often a default password.

Runs on UDP:
```
161/udp  - SNMP agent (where queries go)
162/udp  - SNMP trap (device sends alerts to manager)
```

---

## How SNMP Works

Three components:

- **Manager** - the central system that queries devices and collects data. Could be a monitoring platform like Nagios, Zabbix, or just a script.
- **Agent** - software running on each managed device that responds to queries.
- **MIB (Management Information Base)** - the database of everything a device will report. A structured tree of objects, each identified by an OID (Object Identifier).

```
SNMP Manager                    SNMP Agent (Router, Switch, Server)
     |                                        |
     |-- GET request (what is sysUpTime?) --->|
     |                                        |
     |<-- GET response (14 days, 3 hours) ----|
     |                                        |
     |-- SET request (change hostname) ------>|  (needs write access)
     |                                        |
     |<-- SET response (ok) ------------------|
```

The MIB is essentially a map of everything you can ask about. Different device types have different MIBs. A Cisco router exposes different OIDs than a Windows server. The standard ones are universal - system info, interfaces, uptime. Vendor-specific ones vary.

---

## Community Strings - The Weak Point

In SNMPv1 and SNMPv2c, authentication is handled by a **community string**. It's basically a password that gets sent in cleartext with every SNMP request.

There are two access levels:

| Community String | Access Level |
|-----------------|-------------|
| `public` | Read-only (default on almost everything) |
| `private` | Read-write (default on many devices) |

Those are the actual default values shipped on most network equipment. A huge number of devices never get changed. Scan a corporate network and you'll frequently find SNMP running with `public` as the community string - meaning you can read the full MIB of every device that responds.

With read-only access you get:
- System hostname and description
- OS version and uptime
- Network interfaces and their IP addresses
- Routing tables
- List of running processes
- Installed software
- Connected users
- ARP and MAC tables

That's a complete map of a device's configuration and state handed over with a default password.

With read-write access (`private`) you can change things. Modify routing tables, change configurations, potentially cause outages or redirect traffic.

---

## SNMP Versions

| Version | Authentication | Encryption | Status |
|---------|---------------|-----------|--------|
| SNMPv1 | Community string (cleartext) | None | Obsolete, still common |
| SNMPv2c | Community string (cleartext) | None | Still widely used |
| SNMPv3 | Username + password | Yes (AES/DES) | Current standard |

SNMPv1 sends everything in cleartext - the community string, the query, and the response. Anyone running tcpdump on the same network can capture the community string passively just by watching SNMP traffic.

SNMPv3 adds real authentication and encryption. It's what you should be running. The problem is that SNMPv3 is more complex to configure, so many admins leave v1 or v2c enabled alongside it for "compatibility."

---

## Scanning and Enumeration

```bash
# Scan for SNMP on a network (UDP scan)
ahegazy0@kali:~$ sudo nmap -sU -p 161 192.168.1.0/24

# Check if SNMP is running and get basic info
ahegazy0@kali:~$ sudo nmap -sU -p 161 --script snmp-info <target_IP>

# Enumerate everything nmap can pull via SNMP
ahegazy0@kali:~$ sudo nmap -sU -p 161 --script snmp-* <target_IP>
```

`onesixtyone` is a fast SNMP community string brute-forcer. It's specifically designed for this and is much faster than generic password tools for UDP-based SNMP.

```bash
# Try common community strings against a single host
ahegazy0@kali:~$ onesixtyone 192.168.1.1 public
ahegazy0@kali:~$ onesixtyone 192.168.1.1 private

# Use a wordlist of community strings
ahegazy0@kali:~$ onesixtyone -c /usr/share/doc/onesixtyone/dict.txt 192.168.1.1

# Scan a whole subnet with a community string list
ahegazy0@kali:~$ onesixtyone -c community_strings.txt -i targets.txt
```

Once you have a valid community string, `snmpcheck` dumps everything available:

```bash
# Full enumeration of a device
ahegazy0@kali:~$ snmpcheck -t 192.168.1.1 -c public

# Specific sections
ahegazy0@kali:~$ snmpcheck -t 192.168.1.1 -c public -e users    # user accounts
ahegazy0@kali:~$ snmpcheck -t 192.168.1.1 -c public -e process  # running processes
ahegazy0@kali:~$ snmpcheck -t 192.168.1.1 -c public -e software # installed software
ahegazy0@kali:~$ snmpcheck -t 192.168.1.1 -c public -e network  # network interfaces
```

Or use `snmpwalk` to walk the entire MIB tree:

```bash
# Walk the full MIB
ahegazy0@kali:~$ snmpwalk -v2c -c public 192.168.1.1

# Walk a specific OID
ahegazy0@kali:~$ snmpwalk -v2c -c public 192.168.1.1 1.3.6.1.2.1.1   # system info
ahegazy0@kali:~$ snmpwalk -v2c -c public 192.168.1.1 1.3.6.1.2.1.25.4 # running processes
```

---

## What You Can Learn From SNMP

This is why it matters during recon. A single successful SNMP query against one device can give you:

```
System info:     Hostname, OS version, uptime, contact, location
Interfaces:      All network interfaces, IP addresses, MAC addresses
Routing:         Full routing table showing network topology
ARP table:       IP to MAC mappings for connected devices
Users:           Local user accounts on the system
Processes:       Every running process with PID
Software:        Complete list of installed applications and versions
TCP connections: Active connections and their state
```

The installed software list is particularly useful for picking exploits. If SNMP tells you a server is running Apache 2.2.15, you know exactly what CVEs apply to it. You didn't have to scan port 80 or interact with the web server at all.

---

## SNMP Traps

SNMP also works in the other direction. Devices can be configured to send unsolicited alerts to the manager when something happens - this is called a **trap**.

```
[Device] -- trap --> [SNMP Manager]
"CPU just hit 95%"
"Interface went down"
"Authentication failure"
```

Authentication failure traps are interesting from a security monitoring perspective. If your SNMP manager is logging traps and you see a flood of authentication failures, someone is probably brute-forcing your community strings.

---

## Defense

- **Disable SNMP if you don't need it.** A lot of devices have it enabled by default without anyone actively using it. Just turn it off.
- **Use SNMPv3 only.** If you do need SNMP, v3 with proper authentication and encryption is the only version worth running.
- **Change default community strings.** Never leave `public` and `private` as-is. Use long random strings.
- **Firewall port 161/udp.** SNMP should only be reachable from your management network, never from the general LAN or internet.
- **Read-only where possible.** Most monitoring use cases only need read access. Don't enable write access unless absolutely necessary.

---

## Quick Reference

| Topic | Detail |
|-------|--------|
| Port | 161/udp (queries), 162/udp (traps) |
| Default read string | `public` |
| Default write string | `private` |
| Weak versions | SNMPv1, SNMPv2c (cleartext) |
| Strong version | SNMPv3 (encrypted) |
| Brute-force tool | onesixtyone |
| Enumeration tool | snmpcheck, snmpwalk |
| Key database | MIB (Management Information Base) |

---

## Practice

```bash
# 1. Scan your local network for SNMP agents via UDP
ahegazy0@kali:~$ sudo nmap -sU -p 161 192.168.1.0/24

# 2. Test common community strings against responsive targets
ahegazy0@kali:~$ onesixtyone 192.168.1.1 public
ahegazy0@kali:~$ onesixtyone 192.168.1.1 private

# 3. Pull full telemetry report with snmpcheck
ahegazy0@kali:~$ snmpcheck -t 192.168.1.1 -c public

# 4. Walk the system information MIB subtree
ahegazy0@kali:~$ snmpwalk -v2c -c public 192.168.1.1 1.3.6.1.2.1.1
```

- [ ] Scan your lab network for UDP port 161 using `sudo nmap -sU -p 161`.
- [ ] Fast-brute community strings with `onesixtyone <target> public private`.
- [ ] Walk the MIB tree of a responsive target using `snmpwalk -v2c -c public <target>` and examine device description OIDs.
- [ ] Use `snmpcheck` to extract running processes and network interfaces from a test agent.
- [ ] Explain why SNMPv1/v2c cleartext community strings expose sensitive network telemetry to passive sniffing.

> 💡 *For deeper practice, I also recommend completing the end-of-chapter exercises in the official **Network Basics for Hackers** book.*

---

*Up next: Module 12 - HTTP & HTTPS (The Web's Language)*
