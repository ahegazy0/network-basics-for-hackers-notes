# Module 3 - Network Analysis

> *Stop guessing what's on the wire. Start reading it.*

---

## What's Actually Happening

Every time your machine talks to another, it sends packets. Those packets travel through cables, switches, routers, wireless access points. At every point along the way, the data is physically present on a medium you can tap into.

Network analysis is just that. Tap in, read what's there.

This is how incident responders figure out what happened during a breach. It's also how attackers find credentials and session cookies on networks running cleartext protocols. Same skill, opposite use.

---

## Promiscuous Mode

By default, your network card only picks up packets addressed to your machine. Everything else gets ignored at the hardware level.

Promiscuous mode turns that filter off. Your card captures everything passing by - broadcasts, traffic meant for other machines, all of it.

```bash
# Enable promiscuous mode manually
sudo ip link set eth0 promisc on

# Verify (look for "PROMISC" in the flags)
ip link show eth0
```

Most sniffers like Wireshark and tcpdump enable this automatically when they start. But it's worth knowing what it means - without it you're only seeing your own traffic, which isn't very useful.

One thing to know: on a switched network, promiscuous mode alone only gets you traffic on your local segment. Switches send packets only to the right port. To capture other hosts' traffic on a switch, you need ARP poisoning or a mirror port. That's a later module.

---

## tcpdump

Command-line packet sniffer. No GUI, no setup. Runs on servers, embedded systems, anything with a terminal. This is what you use when Wireshark isn't an option.

```bash
# Basic capture on an interface
sudo tcpdump -i eth0

# Save to a file, open in Wireshark later
sudo tcpdump -i eth0 -w capture.pcap

# Read a saved capture
tcpdump -r capture.pcap

# Only show HTTP traffic
sudo tcpdump -i eth0 port 80

# Filter by host
sudo tcpdump -i eth0 host 192.168.1.50

# Show packet contents in ASCII (great for spotting cleartext)
sudo tcpdump -i eth0 -A port 80

# Combine filters
sudo tcpdump -i eth0 host 192.168.1.1 and port 443

# Hunt for login keywords in cleartext traffic
sudo tcpdump -i eth0 -A | grep -i "pass\|login\|user\|username"
```

The raw output looks confusing at first:

```
14:32:11.482910 IP 192.168.1.5.54231 > 93.184.216.34.80: Flags [S], seq 0, win 64240
```

Left to right: timestamp, source IP.port, destination IP.port, TCP flags, sequence number, window size.

`[S]` = SYN. `[S.]` = SYN/ACK. `[.]` = ACK. `[P.]` = PUSH (actual data). `[F.]` = FIN (closing).

Once you read a few captures, this becomes second nature.

---

## Wireshark

The full visual experience. Same idea as tcpdump but with a GUI that lets you dig into every field of every layer of every packet.

Three panes when you open a capture:

```
+-----------------------------------------------------+
|  Packet List  - one row per packet, high-level info  |
+-----------------------------------------------------+
|  Packet Details  - expandable protocol tree          |
|    > Frame                                           |
|    > Ethernet II                                     |
|    > Internet Protocol (IP)                          |
|    > Transmission Control Protocol (TCP)             |
|    > Hypertext Transfer Protocol (HTTP)              |
+-----------------------------------------------------+
|  Packet Bytes  - raw hex and ASCII side by side      |
+-----------------------------------------------------+
```

The filter bar is where the real power is. Large captures have thousands of packets. You need to cut through the noise fast.

```
# Only HTTP traffic
http

# Traffic to or from a specific IP
ip.addr == 192.168.1.50

# Only DNS queries
dns

# Only new TCP connections (SYN only, no ACK)
tcp.flags.syn == 1 && tcp.flags.ack == 0

# Find packets containing specific text
frame contains "password"
frame contains "login"
```

**Follow TCP Stream** is one of the most useful features. Right-click any packet, Follow, TCP Stream. It reassembles the full conversation between two hosts into one readable window. On unencrypted protocols you can read everything.

The Statistics menu is something beginners often skip. "Protocol Hierarchy" shows a breakdown of everything in the capture by protocol with percentages. If 40% of traffic is some protocol you don't recognize, that's worth looking into.

---

## What Cleartext Traffic Actually Looks Like

On any HTTP site (no TLS), everything is readable. Literally everything.

Someone submits a login form over HTTP:

```
POST /login HTTP/1.1
Host: example.com
Content-Type: application/x-www-form-urlencoded

username=john&password=hunter2
```

That's not a demo. That's what it looks like in Wireshark's TCP stream view. No cracking, no exploitation. Just reading.

Same thing with cookies. If a session cookie gets sent over HTTP, you can copy it and replay it in your own browser to hijack the session. This is called a session hijack and it works directly from the capture. HTTPS encrypts all of this. You'd see the TLS handshake and then unreadable noise.

---

## netstat - What Is Your Machine Doing Right Now

`netstat` shows active connections and listening ports on your own machine. Good for baselining what's normal and spotting things that shouldn't be there.

```bash
# All connections and listening ports
netstat -a

# Show process names with ports (needs root)
sudo netstat -tulnp

# Only established connections
netstat -an | grep ESTABLISHED

# Only listening ports
netstat -an | grep LISTEN

# Check for web connections
netstat -a | grep :80
netstat -a | grep :443

# On modern Linux, ss is faster
ss -tulnp
```

On a machine you know well, netstat is routine. On a machine you're investigating, it can reveal backdoors - unexpected listening ports, outbound connections to strange IPs, processes that have no business on the network.

---

## Other Useful Commands

```bash
# Your IP addresses and MAC address
ip a
ifconfig       # older systems

# Test if a host is up
ping 8.8.8.8
ping -c 4 192.168.1.1     # send exactly 4 pings

# Trace the route packets take
traceroute google.com     # Linux/macOS
tracert google.com        # Windows

# Show ARP cache (IP to MAC mappings your machine knows)
arp -a

# DNS lookups
nslookup google.com
dig google.com
```

`traceroute` is underrated. It shows every router hop between you and a destination with response times. Useful for understanding network topology and spotting traffic being routed somewhere unexpected.

---

## A Typical Analysis Workflow

```
1. Capture traffic
   -> sudo tcpdump -i eth0 -w session.pcap

2. Open in Wireshark
   -> wireshark session.pcap

3. Orient yourself
   -> Statistics > Protocol Hierarchy (what protocols are here?)
   -> Statistics > Conversations (who is talking to who?)

4. Filter down
   -> http, dns, tcp.flags.syn==1, ip.addr==x.x.x.x

5. Follow interesting streams
   -> Right-click > Follow > TCP Stream

6. Export if needed
   -> File > Export Objects > HTTP (grabs files transferred over HTTP)
```

---

## Quick Reference

| Tool | What It Does |
|------|-------------|
| `tcpdump` | CLI packet capture, fast, works anywhere |
| `wireshark` | GUI capture and deep analysis |
| `netstat` / `ss` | Active connections and listening ports |
| `ip a` / `ifconfig` | Your IP and MAC addresses |
| `ping` | Test if a host responds |
| `traceroute` | Map the route to a destination |
| `arp -a` | Local IP to MAC mappings |

---

## Lab

```bash
# 1. Capture some traffic, browse a site, then stop
sudo tcpdump -i eth0 -w ~/test.pcap
# browse something, ctrl+c to stop
wireshark ~/test.pcap

# 2. In Wireshark, type "http" in the filter bar
#    Find any GET request and click it
#    Expand "Hypertext Transfer Protocol" in the middle pane

# 3. Check what your machine is connected to right now
ss -tulnp
netstat -an | grep ESTABLISHED

# 4. Right-click any TCP packet in Wireshark > Follow > TCP Stream
#    Read what you see
```

**Questions:**

- Why does promiscuous mode matter more on a wireless network than a wired switched one?
- Capture traffic while loading any HTTP site. Can you find the GET request? What headers show up?
- What's the difference between what `netstat` shows you vs what `tcpdump` captures?

---

*Next: ARP - the protocol that maps IPs to MAC addresses, and why it's one of the easiest things to abuse on a local network.*
