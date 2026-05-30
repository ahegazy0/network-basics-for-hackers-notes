# Module 11 - The Network Manager (SNMP)

> *A protocol designed to give admins visibility into every device on a network. Which means it also gives attackers visibility into every device on a network.*

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
# Scan for SNMP on a network
nmap -sU -p 161 192.168.1.0/24

# Check if SNMP is running and get basic info
nmap -sU -p 161 --script snmp-info <target_IP>

# Enumerate everything nmap can pull via SNMP
nmap -sU -p 161 --script snmp-* <target_IP>
```

`onesixtyone` is a fast SNMP community string brute-forcer. It's specifically designed for this and is much faster than generic password tools for UDP-based SNMP.

```bash
# Try common community strings against a single host
onesixtyone 192.168.1.1 public
onesixtyone 192.168.1.1 private

# Use a wordlist of community strings
onesixtyone -c /usr/share/doc/onesixtyone/dict.txt 192.168.1.1

# Scan a whole subnet with a community string list
onesixtyone -c community_strings.txt -i targets.txt
```

Once you have a valid community string, `snmpcheck` dumps everything available:

```bash
# Full enumeration of a device
snmpcheck -t 192.168.1.1 -c public

# Specific sections
snmpcheck -t 192.168.1.1 -c public -e users    # user accounts
snmpcheck -t 192.168.1.1 -c public -e process  # running processes
snmpcheck -t 192.168.1.1 -c public -e software # installed software
snmpcheck -t 192.168.1.1 -c public -e network  # network interfaces
```

Or use `snmpwalk` to walk the entire MIB tree:

```bash
# Walk the full MIB
snmpwalk -v2c -c public 192.168.1.1

# Walk a specific OID
snmpwalk -v2c -c public 192.168.1.1 1.3.6.1.2.1.1   # system info
snmpwalk -v2c -c public 192.168.1.1 1.3.6.1.2.1.25.4 # running processes
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

## Lab

```bash
# 1. Scan your local network for SNMP
sudo nmap -sU -p 161 192.168.1.0/24

# 2. Try the default community string against any hosts that respond
onesixtyone 192.168.1.1 public
onesixtyone 192.168.1.1 private

# 3. If something responds, pull the full info
snmpcheck -t 192.168.1.1 -c public

# 4. Walk the system info OID manually
snmpwalk -v2c -c public 192.168.1.1 1.3.6.1.2.1.1

# 5. Check if any device is running SNMPv1 (cleartext)
nmap -sU -p 161 --script snmp-info 192.168.1.1
```

**Things to think about:**

- If SNMP runs on UDP and has no connection setup, how does that affect detection compared to TCP-based protocols?
- You find a server with SNMP running and `public` works. The installed software list shows an unpatched version of OpenSSH. What's your next step?
- Why do admins leave default community strings unchanged? What organizational failure does that represent?

---

*Next: databases over the network - how SQL servers get exposed, and why finding an open database port is often the end of the engagement.*
