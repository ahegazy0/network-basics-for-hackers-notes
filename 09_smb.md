# Module 9 - Sharing is Risky (SMB)

> *The protocol that powers every shared folder and network drive in Windows - and one of the most exploited things in the history of hacking.*

---

## What SMB Is

SMB (Server Message Block) is the protocol Windows uses to share files, printers, and other resources across a network. When you map a network drive at work, access a shared folder on another machine, or print to a network printer - that's SMB underneath.

It's been part of Windows since the early 90s. That long history is part of the problem. Decades of versions, backward compatibility requirements, and features nobody uses anymore have made it a complex protocol with a lot of surface area to attack.

Port 445 is the one to know. If you see it open during a scan, pay attention.

```
Common SMB ports:
445/tcp  - SMB over TCP/IP (modern, the main one)
139/tcp  - SMB over NetBIOS (older, still sometimes open)
137/udp  - NetBIOS Name Service
138/udp  - NetBIOS Datagram Service
```

---

## SMB Versions - Not All Equal

This matters a lot. SMBv1 is fundamentally broken and should never be running anywhere.

| Version | Status | Notes |
|---------|--------|-------|
| SMB 1.0 / CIFS | Obsolete, dangerous | No encryption, multiple critical vulns. Disable immediately. |
| SMB 2.0 | Acceptable | Introduced in Windows Vista, much cleaner design |
| SMB 2.1 | Acceptable | Minor improvements |
| SMB 3.0 | Current | Added end-to-end encryption, better security overall |
| SMB 3.1.1 | Current | Pre-authentication integrity, stronger encryption |

WannaCry, EternalBlue, and most of the legendary SMB exploits target SMBv1. It's still enabled by default on some older systems and some admins turn it on for compatibility without realizing what they're enabling.

---

## EternalBlue and MS17-010

This is the one everyone needs to know.

MS17-010 is a vulnerability in SMBv1 that allows remote code execution without any credentials. You don't need a username or password. You just send specially crafted packets to port 445 and you get code execution on the target.

The exploit was originally developed by the NSA as a cyberweapon codenamed **EternalBlue**. In April 2017, a group called the Shadow Brokers leaked it publicly. Within weeks it was integrated into attack tools.

In May 2017 the **WannaCry** ransomware used EternalBlue to spread automatically across networks. It didn't need anyone to click a link or open an attachment. It just scanned for port 445, exploited MS17-010, installed itself, encrypted the victim's files, and moved on to the next machine. In a few hours it had infected hundreds of thousands of machines across 150 countries - hospitals, banks, telecom companies, government agencies.

Microsoft had released a patch (MS17-010) two months earlier. The machines that got hit hadn't applied it.

The lesson isn't just "patch your systems" - though it is that. The lesson is that unpatched SMBv1 on a networked machine is essentially an open door.

---

## Reconnaissance - Finding SMB

```bash
# Scan for open SMB ports
nmap -p 445,139 192.168.1.0/24

# Detect SMB version and check for known vulnerabilities
nmap -p 445 --script smb-vuln-ms17-010 192.168.1.1
nmap -p 445 --script smb-security-mode 192.168.1.1

# Full SMB enumeration script set
nmap -p 445 --script smb-enum-shares,smb-enum-users 192.168.1.1

# Check if SMBv1 is enabled
nmap -p 445 --script smb-protocols 192.168.1.1
```

`smbclient` is the Linux tool for interacting with SMB shares directly:

```bash
# List available shares on a target
smbclient -L //192.168.1.10 -N    # -N means no password (try anonymous)
smbclient -L //192.168.1.10 -U username

# Connect to a specific share
smbclient //192.168.1.10/sharename -U username

# Once connected, it works like a basic FTP client
# ls, get, put, cd, etc.
```

---

## Guest Access - The Overlooked Problem

A lot of admins set up SMB shares quickly and leave guest access enabled. This means anyone on the same network can browse and read those shares with no credentials at all.

In a corporate environment this is a serious problem. Sensitive documents, configuration files, backup archives - all potentially readable by any device connected to the Wi-Fi.

```bash
# Check if a share allows anonymous/guest access
smbclient -L //192.168.1.10 -N

# If it connects without asking for a password, guest access is on
# Try connecting to specific shares
smbclient //192.168.1.10/shared -N
```

This is one of the first things to check during an internal network assessment. You'd be surprised how often it comes up.

---

## Samba - SMB on Linux

Samba is the Linux implementation of SMB. It lets Linux machines share files with Windows machines and vice versa. On a mixed network, Samba is what makes the two talk to each other.

```bash
# Install Samba
sudo apt install samba

# Main config file
sudo nano /etc/samba/smb.conf

# A basic share config looks like this:
[shared]
   path = /home/user/shared
   browseable = yes
   read only = no
   guest ok = no
   valid users = @smbgroup

# Restart after changes
sudo systemctl restart smbd

# Add a Samba user (separate from Linux user accounts)
sudo smbpasswd -a username

# Check your config for syntax errors
testparm
```

Running Samba yourself and configuring it from scratch gives you a good understanding of where misconfigurations happen. The `smb.conf` file has a lot of options, and the insecure defaults are easy to accidentally leave in.

---

## Enum4linux - SMB Enumeration

`enum4linux` is a tool specifically for pulling information from Windows and Samba systems via SMB and related protocols.

```bash
# Full enumeration
enum4linux -a 192.168.1.10

# Just users
enum4linux -U 192.168.1.10

# Just shares
enum4linux -S 192.168.1.10

# Just OS info
enum4linux -o 192.168.1.10
```

On a vulnerable or misconfigured target this can return usernames, group memberships, share names, OS version, domain information - all without any credentials.

---

## MS08-067 - The Earlier One

Before EternalBlue there was MS08-067. Another SMB vulnerability, this time in the Server service on Windows XP and Windows Server 2003. Also remote code execution, also without credentials, also wormable.

Conficker used MS08-067 to spread in 2008 and ended up on an estimated 9-15 million machines. It was so widespread it infected hospital equipment, military computers, and government systems in multiple countries.

Both MS08-067 and MS17-010 follow the same pattern: critical SMB vuln, slow patching, catastrophic worm. The history repeats because the underlying problem (legacy protocol + delayed patching + internet-exposed port 445) keeps showing up.

---

## Defense

- **Disable SMBv1** - there's no good reason to have it enabled in 2024. On Windows: `Set-SmbServerConfiguration -EnableSMB1Protocol $false`
- **Patch immediately** - EternalBlue had a patch two months before WannaCry. Machines that were patched weren't affected.
- **Block port 445 at the perimeter** - SMB should never be reachable from the internet. It's an internal protocol.
- **Disable guest access** - require authentication on all shares.
- **Use SMB signing** - prevents relay attacks where an attacker captures and replays authentication.
- **Segment the network** - subnets and VLANs limit how far a worm can travel even if it gets in.

---

## Quick Reference

| Topic | Detail |
|-------|--------|
| Main port | 445/tcp |
| Legacy port | 139/tcp |
| Dangerous version | SMBv1 (disable it) |
| Safe versions | SMB 3.0, 3.1.1 |
| Famous exploits | EternalBlue (MS17-010), MS08-067 |
| Linux implementation | Samba |
| Enum tool | enum4linux, smbclient, nmap scripts |

---

## Lab

```bash
# 1. Scan your local network for SMB
nmap -p 445,139 192.168.1.0/24

# 2. Check if SMBv1 is running on any hosts
nmap -p 445 --script smb-protocols <target_IP>

# 3. Try listing shares anonymously
smbclient -L //<target_IP> -N

# 4. Set up a basic Samba share on your Linux VM
sudo apt install samba
sudo nano /etc/samba/smb.conf
# Add a [testshare] section pointing to a folder
sudo systemctl restart smbd
# Try connecting from another machine or from localhost
smbclient //127.0.0.1/testshare -U yourusername
```

**Things to think about:**

- WannaCry had a patch available before it hit. What organizational failures led to hundreds of thousands of machines still being unpatched?
- Why is SMBv1 still found on networks today if it's known to be dangerous?
- If SMB should never be exposed to the internet, why do so many port scans find it open on public IPs?

---

*Next: HTTP and HTTPS - the protocols your browser uses for everything, and where a huge portion of real-world web attacks happen.*
