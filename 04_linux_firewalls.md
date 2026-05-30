# Module 4 - Building the Wall (Linux Firewalls)

> *Every packet that touches your machine goes through this. Worth understanding properly.*

---

## What a Firewall Does

A firewall sits between your machine and the network and decides what traffic gets through. Every packet trying to enter or leave gets checked against a list of rules. Match a rule, get handled. No match, the default policy decides.

On Linux that's `iptables`. It's been in the kernel for a long time. Newer tools like `ufw` and `firewalld` exist, but they're just wrappers around `iptables`. Understanding the real thing means you'll never be confused by what those tools are doing underneath.

The bouncer analogy is accurate. There's a list. Every packet that shows up gets checked top to bottom. First matching rule wins. You either get in (ACCEPT) or get turned away (DROP). Not on the list at all? Default policy decides.

---

## Tables and Chains

iptables organizes rules into **tables**. For basic firewall work, the only one you really need is the `filter` table. That's the default and it handles all allow/deny decisions.

Inside the filter table there are three **chains**:

```
INPUT    ->  traffic coming INTO your machine
OUTPUT   ->  traffic LEAVING your machine
FORWARD  ->  traffic passing THROUGH (only if you're routing)
```

Each chain is its own list of rules. Three separate bouncers - one at the entrance, one at the exit, one for people passing through. A packet hits the right chain and gets walked down the list until something matches.

---

## Basic Syntax

```bash
sudo iptables -A INPUT -s <source_IP> -j DROP
```

Breaking it down:

- `-A INPUT` - **A**ppend this rule to the INPUT chain
- `-s <IP>` - match packets from this **s**ource IP
- `-j DROP` - **j**ump to this action when the rule matches

Three actions you'll use constantly:

| Action | What happens |
|--------|-------------|
| `ACCEPT` | Packet gets through |
| `DROP` | Silently discarded. Sender gets no response. |
| `REJECT` | Discarded, but sender gets an error back |

DROP vs REJECT matters. DROP makes your machine look like it doesn't exist. No reply, nothing. An attacker scanning gets silence, which is less information for them. REJECT at least confirms something is alive at that address. For inbound blocking, DROP is almost always better.

---

## Rule Order - This Is Where People Get Burned

iptables reads rules **top to bottom and stops at the first match**. The order you add rules is the order they run. This trips up beginners constantly.

Classic mistake:

```bash
# Drop everything
sudo iptables -A INPUT -j DROP

# Allow SSH - this will NEVER fire
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
```

That SSH rule is dead. Every packet hits DROP first and gets discarded before iptables even checks the second rule. You've locked yourself out.

Correct order - specific rules first, broad rules last:

```bash
# Allow SSH first
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow replies to connections you started
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Now drop everything else
sudo iptables -A INPUT -j DROP
```

That `ESTABLISHED,RELATED` rule is easy to forget and it breaks everything when you do. Without it, outbound connections work but replies get blocked coming back in. Websites stop loading, updates fail, everything that talks to the internet breaks.

---

## Default Policy

Every chain has a **default policy** - what happens when a packet reaches the end without matching anything.

```bash
# Drop anything not explicitly allowed
sudo iptables -P INPUT DROP

# Allow anything not explicitly blocked
sudo iptables -P INPUT ACCEPT
```

DROP as default = allowlist. You decide what's allowed, everything else is silently discarded. Secure, but more work to set up.

ACCEPT as default = blocklist. Everything gets through unless you block it. Easy to set up, easy to miss things.

For any machine you care about, default DROP on INPUT is the right call. You're forced to think about every service you're intentionally exposing.

---

## Common Commands

```bash
# See all current rules
sudo iptables -L
sudo iptables -L -v               # verbose, shows packet/byte counters
sudo iptables -L --line-numbers   # shows rule numbers, needed for deleting

# Block incoming traffic from a specific IP
sudo iptables -A INPUT -s 10.10.10.10 -j DROP

# Allow a specific port inbound
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT

# Block outbound traffic to a specific IP
sudo iptables -A OUTPUT -d 93.184.216.34 -j DROP

# Delete a rule by line number
sudo iptables -D INPUT 3

# Flush all rules - wipes everything, fresh start
sudo iptables -F

# Set default policy
sudo iptables -P INPUT DROP
sudo iptables -P OUTPUT ACCEPT
```

The `-F` flush is your panic button. Misconfigured rules and locked yourself out of a remote machine? `-F` wipes everything and restores default behavior. Know this command before you start experimenting on anything important.

---

## A Simple Hardened Ruleset

This is what a minimal secure server looks like as a starting point:

```bash
# Allow loopback (machine talking to itself - never block this)
sudo iptables -A INPUT -i lo -j ACCEPT

# Allow established connections (replies to your outbound traffic)
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow SSH
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow web traffic if it's a web server
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Drop everything else
sudo iptables -A INPUT -j DROP
```

Each rule has a reason. Loopback lets the OS talk to itself - blocking it breaks things in weird ways. ESTABLISHED lets replies in. The rest is just whitelisting what you actually need open.

---

## Quick Reference

| Flag | What it does |
|------|-------------|
| `-A` | Append a rule to a chain |
| `-D` | Delete a rule |
| `-L` | List all current rules |
| `-F` | Flush (clear) all rules |
| `-P` | Set default policy for a chain |
| `-s` | Match by source IP |
| `-d` | Match by destination IP |
| `-p` | Match by protocol (tcp, udp, icmp) |
| `--dport` | Match by destination port |
| `-j` | Jump to action (ACCEPT / DROP / REJECT) |

---

## Lab

```bash
# 1. Check your current rules
sudo iptables -L --line-numbers

# 2. Block your own machine from a specific port
sudo iptables -A INPUT -s 127.0.0.1 -p tcp --dport 80 -j DROP

# 3. List rules and confirm yours appeared
sudo iptables -L --line-numbers

# 4. Delete the rule you added
sudo iptables -D INPUT 1    # use the actual line number from step 3

# 5. Flush everything and verify
sudo iptables -F
sudo iptables -L
```

**Things to think about:**

- If you set `-P INPUT DROP` as default policy then run `iptables -F` to flush rules - what happens to your SSH session? Why?
- From an attacker's port scan, what's the difference between a port that REJECTs vs one that silently DROPs?
- Why does `ESTABLISHED,RELATED` need to come before the final DROP rule?

---

*Next: how traffic gets routed between networks, and where that process gets abused to redirect or intercept packets.*
