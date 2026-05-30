# Module 2 - Subnetting & CIDR

> *One big network is a mess. Subnetting is how you clean it up - and limit the damage when something goes wrong.*

---

## Why Subnetting Exists

Early internet design handed out IP ranges like they were unlimited. Class A, B, C blocks went to organizations whether they needed 10 hosts or 10 million. That burned through addresses fast, and huge flat networks became slow and hard to manage.

Subnetting fixes two things:

1. **Efficiency** - split a large block into smaller chunks that match your actual needs
2. **Security** - isolate parts of the network so a breach in one area doesn't reach everything else

The hotel analogy works well here. One giant open floor where everyone can hear everything is chaos. Building walls creates separate rooms. You can lock certain rooms, monitor who enters, and stop problems from spreading.

---

## IP Address Structure

An IPv4 address is 32 bits, written as four 8-bit octets in decimal:

```
192      .  168      .   1       .   50
11000000    10101000    00000001    00110010
```

Every IP has two parts:

- **Network ID** - which network this address belongs to
- **Host ID** - which specific device inside that network

The subnet mask tells you where the line is between those two parts.

---

## Subnet Masks

A subnet mask is also 32 bits. The `1` bits mark the network portion, the `0` bits mark the host portion.

```
IP address:    192.168.1.50   ->  11000000.10101000.00000001.00110010
Subnet mask:   255.255.255.0  ->  11111111.11111111.11111111.00000000
                                  |------network-------|----host----|
```

Everything under the `1` bits is the network. Everything under the `0` bits is the host.

So `192.168.1.50` with mask `255.255.255.0` is on network `192.168.1.0`, host `.50`.

---

## CIDR Notation

Writing `255.255.255.0` every time is annoying. CIDR is the shorthand. Count the `1` bits in the mask and put that number after a slash.

`255.255.255.0` has 24 ones -> `/24`
`255.255.0.0` has 16 ones -> `/16`
`255.0.0.0` has 8 ones -> `/8`

| CIDR | Subnet Mask | Total Addresses | Usable Hosts |
|------|-------------|-----------------|-------------|
| /8 | 255.0.0.0 | 16,777,216 | 16,777,214 |
| /16 | 255.255.0.0 | 65,536 | 65,534 |
| /24 | 255.255.255.0 | 256 | 254 |
| /25 | 255.255.255.128 | 128 | 126 |
| /26 | 255.255.255.192 | 64 | 62 |
| /27 | 255.255.255.224 | 32 | 30 |
| /28 | 255.255.255.240 | 16 | 14 |
| /30 | 255.255.255.252 | 4 | 2 |

You always lose 2 addresses per subnet. One is the **network address** (all host bits = 0) and one is the **broadcast address** (all host bits = 1). Neither can be assigned to a device.

For a `/24`: 256 total - 2 = **254 usable hosts**.

---

## How Routers Decide Where to Send Traffic

When your machine sends a packet, it needs to know: is the destination on my local network, or does it go to the router?

It uses a bitwise AND operation. Compares your IP and the destination IP against the subnet mask.

```
Your IP:        192.168.1.50   ->  11000000.10101000.00000001.00110010
Subnet mask:    255.255.255.0  ->  11111111.11111111.11111111.00000000
AND result:     192.168.1.0    ->  11000000.10101000.00000001.00000000
                                   <- your network ID ->
```

Do the same AND with the destination IP. Result matches your network ID? Same subnet, send directly. Result is different? Goes to the router.

You don't need to do this math by hand. But it explains why `192.168.1.x` and `192.168.2.x` can't talk directly without going through a router.

---

## Subnetting a Network - Worked Example

You have `192.168.10.0/24` and need to split it into 4 subnets.

You need 4 subnets, so borrow 2 bits from the host portion (2 squared = 4).
New mask: `/26` (24 + 2 = 26) = `255.255.255.192`

Each `/26` has 64 addresses, 62 usable:

```
Subnet 1:  192.168.10.0   - 192.168.10.63    (usable: .1 - .62)
Subnet 2:  192.168.10.64  - 192.168.10.127   (usable: .65 - .126)
Subnet 3:  192.168.10.128 - 192.168.10.191   (usable: .129 - .190)
Subnet 4:  192.168.10.192 - 192.168.10.255   (usable: .193 - .254)
```

Clean, predictable, easy to reference in firewall rules.

---

## Security Use Case - Broadcast Domains

This is the part that actually matters for security.

When a device sends a **broadcast** (a packet addressed to everyone), it reaches every device in the same subnet and stops at the router. It doesn't cross into other subnets.

That boundary is called a broadcast domain. Subnets define them.

```
[Subnet A: Finance]    [Subnet B: HR]       [Subnet C: Engineering]
  192.168.1.0/24         192.168.2.0/24       192.168.3.0/24
  +--------------+       +--------------+     +--------------+
  | workstation  |       | workstation  |     | workstation  |
  | file server  |       | HR database  |     | dev servers  |
  +------+-------+       +------+-------+     +------+-------+
         +-------------- Router +--------------------------+
```

If an attacker gets into Subnet C, their discovery scans and ARP broadcasts stay inside that subnet. They can't just reach the HR database without crossing the router, which is where your firewall rules sit.

Flat network with no subnetting? One compromised machine can see everything.

---

## Reading CIDR in the Wild

You'll see CIDR constantly in firewall rules, cloud security groups, routing tables, nmap output, VPN configs.

```bash
# This specific host only
192.168.1.50/32

# The entire 192.168.1.x range
192.168.1.0/24

# All IPv4 addresses (default route)
0.0.0.0/0
```

`/32` is one host. `/0` is the entire internet. Everything between is a range.

---

## Quick Reference

| CIDR | Usable Hosts | Common Use |
|------|-------------|-----------|
| /30 | 2 | Point-to-point router links |
| /28 | 14 | Small office segment |
| /27 | 30 | Small team |
| /26 | 62 | Medium segment |
| /24 | 254 | Standard LAN |
| /16 | 65,534 | Large campus network |
| /8 | 16,777,214 | ISP / huge enterprise |

---

## Lab

```bash
# Check your current network and subnet mask
ip a                   # Linux - look for "inet x.x.x.x/xx"
ifconfig               # older Linux / macOS
ipconfig /all          # Windows

# Check your routing table
ip route
route -n               # alternative

# Scan a full subnet to see what's alive
sudo nmap -sn 192.168.1.0/24
```

**Questions:**

- Your machine is `10.0.0.5/22`. What's the network address? How many hosts are in that subnet?
- Why does `/30` only give 2 usable addresses if it has 4 total?
- Finance and HR are on the same `/24`. What are the security risks vs putting them on separate `/26` subnets?

---

*Next: how devices find each other inside a subnet - ARP, MAC addresses, and why the local network is a great place to intercept traffic.*
