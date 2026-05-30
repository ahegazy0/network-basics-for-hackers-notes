# Module 10 - The Mailman's Protocol (SMTP)

> *Email feels instant and simple. Under the hood it's a chain of servers, handshakes, and trust relationships - most of which were designed before security was a serious concern.*

---

## How Email Actually Moves

When you hit send on an email, most people imagine it goes straight to the recipient. It doesn't. It travels through a chain of servers, each one handing it off to the next.

Three main components:

- **MUA (Mail User Agent)** - the app you use to write and read email. Outlook, Thunderbird, Gmail in a browser. This is just the interface.
- **MTA (Mail Transfer Agent)** - the actual mail server. Receives your message from the MUA and routes it toward the destination. Exim, Postfix, Sendmail, Microsoft Exchange are all MTAs.
- **MDA (Mail Delivery Agent)** - the final step. Takes the message from the last MTA and drops it into the recipient's mailbox.

The flow looks like this:

```
[Your Email App]        [Your Mail Server]      [Recipient's Mail Server]    [Their Inbox]
  MUA (Outlook)   --->    MTA (Exim)       --->     MTA (Exchange)       --->  MDA/Mailbox

    Port 587                Port 25                    Port 25
 (submission)          (server to server)          (server to server)
```

SMTP handles the MUA to MTA leg and all the MTA to MTA hops. It's a push protocol - it only sends mail forward. Reading email from a mailbox is handled by different protocols (IMAP, POP3).

---

## SMTP Ports

| Port | Use |
|------|-----|
| 25 | Server to server mail relay - the original SMTP port |
| 587 | Mail submission - your email client sends to your mail server here |
| 465 | SMTPS - SMTP over SSL (older, still used) |
| 110 | POP3 - downloading mail from a server |
| 143 | IMAP - accessing mail on a server |

Port 25 is what mail servers use to talk to each other. Your email client typically uses 587. Finding port 25 open on a server during a scan means there's a mail server worth investigating.

---

## The SMTP Conversation

SMTP is a text-based protocol. You can literally telnet to a mail server and have a conversation with it by hand. This is useful for testing and understanding what's exposed.

```bash
# Connect to a mail server on port 25
telnet mail.example.com 25

# The server responds with a banner - often reveals software and version
# 220 mail.example.com ESMTP Exim 4.94.2

# Greet the server
EHLO yourdomain.com

# Server responds with its capabilities
# 250-mail.example.com Hello
# 250-SIZE 52428800
# 250-PIPELINING
# 250-AUTH LOGIN PLAIN
# 250 HELP

# Start a mail transaction
MAIL FROM:<sender@example.com>
RCPT TO:<recipient@example.com>
DATA
Subject: Test

This is a test.
.
QUIT
```

The banner alone is useful. `220 mail.example.com ESMTP Exim 4.94.2` tells you exactly what software and version is running. That's the starting point for looking up known vulnerabilities.

---

## User Enumeration

Some SMTP servers respond differently depending on whether a username exists or not. Two commands expose this:

**VRFY** - asks the server to verify if an address exists
**EXPN** - asks the server to expand a mailing list

```bash
# Manual enumeration via telnet
telnet mail.example.com 25
VRFY john.smith
# 252 2.0.0 john.smith  <- user probably exists
# 550 5.1.1 john.smith  <- user doesn't exist

EXPN admins
# Might return a list of actual email addresses in that group
```

Most hardened servers disable VRFY and EXPN. But many don't, especially internal mail servers.

Nmap has a script that automates this against a wordlist:

```bash
# Enumerate users against a mail server
nmap -p 25 --script smtp-enum-users --script-args smtp-enum-users.methods={VRFY,RCPT,EXPN} <target_IP>

# With a custom user list
nmap -p 25 --script smtp-enum-users --script-args userdb=/usr/share/wordlists/names.txt <target_IP>
```

From an attacker's perspective, a confirmed list of employee email addresses is a prerequisite for targeted phishing. You now know real names, naming conventions (john.smith vs jsmith vs john_smith), and valid addresses to send to.

---

## Open Relays

An open relay is an SMTP server that will forward mail from anyone to anyone, with no authentication. In the early days of email this was normal. Today it's a serious misconfiguration.

An open relay means attackers can send mail that appears to come from your domain without any credentials. Spam campaigns, phishing emails, spoofed executive communications - all using your server's reputation.

```bash
# Test if a server is an open relay
nmap -p 25 --script smtp-open-relay <target_IP>

# Manual test via telnet
telnet mail.target.com 25
EHLO test.com
MAIL FROM:<fake@external.com>
RCPT TO:<anyone@anywhere.com>
# If it accepts this without auth, it's an open relay
```

Legitimate mail servers should reject the RCPT TO if you're trying to relay to an external domain without authenticating first.

---

## Email Spoofing

SMTP has no built-in sender verification. The `MAIL FROM` in the SMTP transaction and the `From:` header in the email itself can be set to anything. This is how phishing emails appear to come from your bank, your CEO, or any trusted address.

Three DNS-based mechanisms were created to address this:

**SPF (Sender Policy Framework)** - a TXT record listing which mail servers are authorized to send email for your domain. Receiving servers can check if the sending server is on the list.

**DKIM (DomainKeys Identified Mail)** - the sending server signs outgoing emails with a private key. The public key is published in DNS. Receiving servers verify the signature.

**DMARC (Domain-based Message Authentication)** - builds on SPF and DKIM, lets domain owners specify what to do with mail that fails checks (quarantine, reject, or nothing).

```bash
# Check if a domain has SPF configured
dig TXT example.com | grep spf

# Check DMARC record
dig TXT _dmarc.example.com

# Check DKIM selector (you need to know the selector name)
dig TXT selector1._domainkey.example.com
```

Missing or misconfigured SPF/DKIM/DMARC means it's trivially easy to send email that appears to come from that domain.

---

## Reconnaissance on Mail Servers

Before any targeted attack, the mail server gives you information:

```bash
# Scan for mail-related ports
nmap -p 25,465,587,110,143 <target_IP>

# Grab the banner and check software version
nmap -p 25 -sV <target_IP>

# Run all SMTP-related NSE scripts
nmap -p 25 --script smtp-* <target_IP>

# Check for common vulnerabilities
nmap -p 25 --script smtp-vuln-cve2010-4344 <target_IP>

# Get MX records to find the mail servers for a domain
dig MX target.com
```

The software version from the banner maps directly to CVE databases. Exim has had multiple critical remote code execution vulnerabilities. Exchange has had several high-profile ones. Knowing the exact version tells you what's potentially exploitable.

---

## Exim and the Microsoft Exchange Breach

**Exim** is one of the most common MTAs on Linux systems. It's the default on Debian-based systems. It has also had several critical vulnerabilities over the years - most notably a series of heap buffer overflow bugs that allow remote code execution as root. If you find an Exim server running an old version, check the CVE list.

```bash
# Install Exim on Linux
sudo apt install exim4

# Configure it
sudo dpkg-reconfigure exim4-config

# Check version
exim --version

# View the mail queue
mailq

# Main config
/etc/exim4/exim4.conf
```

The **2021 Microsoft Exchange** breaches (ProxyLogon, ProxyShell) were a different level. State-sponsored attackers exploited a chain of vulnerabilities in Exchange Server that allowed unauthenticated remote code execution. Tens of thousands of organizations were compromised before patches were widely applied. The attackers installed web shells that persisted for months.

Exchange runs web services alongside SMTP, which extended the attack surface significantly beyond just the mail protocol itself. The lesson from that incident: mail servers are high-value targets because they contain sensitive communications and often have privileged network positions.

---

## Quick Reference

| Component | What it does |
|-----------|-------------|
| MUA | Email client (Outlook, Thunderbird) |
| MTA | Mail server (Exim, Postfix, Exchange) |
| MDA | Final delivery to mailbox |
| Port 25 | Server to server SMTP |
| Port 587 | Client to server mail submission |
| VRFY / EXPN | User enumeration commands |
| SPF | Authorizes sending servers via DNS |
| DKIM | Cryptographic email signing |
| DMARC | Policy enforcement for SPF/DKIM |

---

## Lab

```bash
# 1. Find mail servers for a domain
dig MX gmail.com
dig MX protonmail.com

# 2. Check SPF and DMARC on a domain
dig TXT gmail.com | grep spf
dig TXT _dmarc.gmail.com

# 3. Scan a mail server
nmap -p 25,587,465 -sV <target_IP>

# 4. Try a manual SMTP conversation (use your own test server)
telnet localhost 25
EHLO test.local
# Read the capabilities the server advertises
QUIT

# 5. Test for open relay (on your own test server only)
nmap -p 25 --script smtp-open-relay 127.0.0.1
```

**Things to think about:**

- An email arrives from `ceo@yourcompany.com` asking you to wire money. The domain has no SPF or DMARC records. What does that tell you about whether the email is real?
- Why would an attacker want a list of valid email addresses from VRFY enumeration before sending a phishing campaign vs just guessing addresses?
- Exim is the default MTA on Debian systems. A lot of servers get set up and never touched again. Why does this make mail servers a particularly good target for old CVEs?

---

*Next: HTTP and HTTPS - the foundation of the web, and where the majority of application-layer attacks actually happen.*
