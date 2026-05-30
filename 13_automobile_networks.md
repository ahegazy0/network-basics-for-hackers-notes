# Module 13 - Ghost in the Machine (Automobile Networks)

> *Modern cars run dozens of computers talking to each other constantly. None of them were designed with security in mind.*

---

## Cars Are Networks

A modern vehicle has anywhere from 30 to 100+ Electronic Control Units (ECUs). Each one is a small computer responsible for something specific - engine management, ABS brakes, airbags, transmission, infotainment, climate control, door locks, windows.

All of these need to communicate with each other. The braking system needs to know vehicle speed. The engine controller needs to know gear position. The dashboard needs to read sensor data from everywhere.

The protocol they use to do this is **CAN - Controller Area Network**. Developed by Bosch in the 1980s, it was designed for reliability and speed in a harsh electrical environment. Security was not part of the design brief. At all.

---

## How CAN Works

CAN is a two-wire bus. Every ECU connects to the same two wires - CAN High and CAN Low. When any ECU wants to send a message, it broadcasts to the entire bus. Every other ECU on the bus receives every message. Each ECU decides for itself whether the message is relevant to it.

```
[Engine ECU] ----+
[Brake ECU]  ----|---- CAN Bus (two wires) ---- All ECUs hear everything
[ABS ECU]    ----|
[Dashboard]  ----+
[Door locks] ----+
```

A CAN frame looks like this:

```
| Arbitration ID | DLC | Data (0-8 bytes) |
|   11 or 29 bit |  4b |                  |

Example:
ID: 0x0CF   DLC: 8   Data: 00 00 00 00 00 00 FF 00
```

The **Arbitration ID** identifies what kind of message this is - not which ECU sent it. There's no source address. There's no authentication. There's no encryption. Any ECU on the bus can send any message with any ID.

That last point is the entire security problem.

---

## The OBD-II Port

Every car sold in the US since 1996 has an OBD-II (On-Board Diagnostics) port, usually under the dashboard near the steering column. Mechanics plug diagnostic tools into it to read fault codes and sensor data.

It also connects directly to the CAN bus.

```bash
# Plug a CAN adapter into OBD-II port
# Common adapters: USB2CAN, CANtact, ELM327 (limited)

# Check if Linux sees the CAN interface
ip link show

# Bring up the CAN interface
sudo ip link set can0 up type can bitrate 500000

# Verify it's up
ip link show can0
```

Physical access to the OBD-II port means direct access to the CAN bus. No authentication required.

---

## can-utils

`can-utils` is a collection of Linux command-line tools for working with CAN interfaces.

```bash
# Install
sudo apt install can-utils

# Dump all CAN traffic in real-time
candump can0

# Dump to a file for later analysis
candump -l can0
# Creates a log file like: candump-2024-01-15_143022.log

# Replay a recorded log file
canplayer -I candump-2024-01-15_143022.log

# Send a single CAN frame manually
# cansend <interface> <ID>#<data>
cansend can0 0CF#000000000000FF00

# Live monitor with change highlighting (easier to spot active signals)
cansniffer can0

# Statistics about bus traffic
canbusload can0@500000
```

`cansniffer` is particularly useful during analysis. It highlights bytes that are changing in real-time, which makes it much easier to isolate which CAN IDs respond to physical actions like pressing buttons or turning the steering wheel.

---

## ICSim - Safe Lab Environment

Before touching a real vehicle, use **ICSim** (Instrument Cluster Simulator). It's an open-source simulator that creates a virtual CAN bus with a simulated dashboard - turn signals, speedometer, door locks - all responding to CAN messages.

```bash
# Install ICSim
git clone https://github.com/zombieCraig/ICSim
cd ICSim
make

# Start the virtual CAN interface
sudo modprobe vcan
sudo ip link add dev vcan0 type vcan
sudo ip link set up vcan0

# Run the simulator
./icsim vcan0

# In another terminal, run the controls
./controls vcan0

# Now you can dump traffic from the simulator
candump vcan0
```

The workflow for learning CAN analysis:

```
1. Run ICSim
2. Use candump to capture traffic while using the controls
3. Press a button (lock doors, turn signal, accelerate)
4. Watch which CAN IDs change in the dump
5. Isolate the specific ID and data pattern for that action
6. Try sending that frame manually with cansend
7. See if the simulator responds
```

This process - capture, analyze, isolate, replay - is the foundation of CAN bus research.

---

## Replay Attacks

A replay attack records a valid signal and plays it back later to trigger the same action.

The classic example is a key fob. When you press unlock, your fob transmits a radio signal. If that signal is static (same signal every time), recording it and replaying it unlocks the car. Modern key fobs use rolling codes to prevent this - each press generates a different code. But older vehicles and some aftermarket systems still use static codes.

On the CAN bus itself, replay attacks work by recording the frames associated with a specific action and replaying them:

```bash
# Record all traffic while triggering an action
candump -l vcan0

# Replay the recorded traffic
canplayer -I logfile.log vcan0

# Or send a specific frame repeatedly
while true; do cansend vcan0 19B#000000000000; sleep 0.1; done
```

In ICSim this is how you practice - find the ID for an action, replay it, confirm it works. On a real vehicle this is where things get dangerous, which is why the simulator exists.

---

## Attack Surface Beyond the OBD-II Port

Physical OBD-II access is the obvious entry point, but modern cars have other network interfaces that connect back to the CAN bus:

- **Bluetooth** - for hands-free calling, audio streaming. If the infotainment unit has a Bluetooth stack vulnerability, it's a path to the bus.
- **Wi-Fi** - some vehicles offer hotspots or use Wi-Fi for diagnostic purposes.
- **Cellular** - connected car features, remote start apps, OTA updates. A remote attacker who compromises the telematics unit has CAN access.
- **USB ports** - infotainment systems process USB drives. Malicious firmware or media files have been used to compromise head units.
- **TPMS** - tire pressure monitoring sensors broadcast wirelessly. Researchers have demonstrated attacks through TPMS receivers.

The 2015 Jeep Cherokee remote hack by Charlie Miller and Chris Valasek is the well-known public example. They accessed the vehicle over cellular through the Uconnect infotainment system and were able to send CAN messages affecting steering and brakes. That research led directly to a 1.4 million vehicle recall.

---

## Why CAN Has No Security

CAN was designed in 1983 for use inside a single vehicle. The assumption was that the bus was physically isolated - nobody hostile could get access to it. That was a reasonable assumption in 1983.

Cars are now connected devices. The isolation assumption no longer holds. But CAN is so deeply embedded in automotive hardware and so much vehicle validation testing is tied to it that replacing it wholesale is a decade-long process. CAN FD and newer protocols like Automotive Ethernet are being added, but CAN isn't going away quickly.

The result is that the physical network inside a vehicle is one of the least secured networks you'll encounter.

---

## Quick Reference

| Tool | What it does |
|------|-------------|
| `candump` | Capture and log CAN traffic |
| `canplayer` | Replay a captured log |
| `cansend` | Send a single CAN frame |
| `cansniffer` | Live monitor with change highlighting |
| `canbusload` | Show bus utilization statistics |

| Concept | Detail |
|---------|--------|
| OBD-II | Physical diagnostic port, connects directly to CAN bus |
| CAN Frame | Arbitration ID + data, no source address, no auth |
| ICSim | Safe virtual CAN simulator for learning |
| Replay attack | Record a valid signal, play it back to repeat the action |
| ECU | Individual computer controlling one vehicle subsystem |

---

## Lab (ICSim Only)

```bash
# 1. Set up the virtual CAN interface
sudo modprobe vcan
sudo ip link add dev vcan0 type vcan
sudo ip link set up vcan0

# 2. Start ICSim in one terminal
./icsim vcan0

# 3. Start the controls in another terminal
./controls vcan0

# 4. Start capturing in a third terminal
candump vcan0 | tee can_capture.log

# 5. Use the controls - press lock/unlock, use turn signals
# Watch which CAN IDs appear or change in the dump

# 6. Try cansniffer to see changes more clearly
cansniffer vcan0

# 7. Once you identify a CAN ID for an action, try sending it
cansend vcan0 <ID>#<data>
# Does the simulator respond?
```

**Things to think about:**

- CAN has no source addresses. Why does that make authentication impossible without redesigning the protocol?
- A car's infotainment system gets a software update via cellular. What's the worst-case attack scenario if that update mechanism isn't secured properly?
- Rolling codes on key fobs prevent basic replay attacks. What would an attacker need to do to defeat rolling codes?

---

*Next: SCADA and industrial control systems - the same design philosophy as CAN but running power grids, water treatment plants, and factories.*
