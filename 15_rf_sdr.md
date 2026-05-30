# Module 15 - Signals from the Sky (RF and SDR)

> *Everything wireless is just radio waves. With the right hardware and software, you can see the invisible.*

---

## What RF Hacking Actually Is

Every wireless communication you can think of is radio frequency energy moving through the air. Wi-Fi, Bluetooth, cellular, GPS, garage doors, key fobs, aircraft transponders, ship tracking, weather satellites, pagers, baby monitors - all of it is just RF signals on different frequencies.

Traditionally, you needed dedicated hardware for each signal type. A police scanner for one thing, an FM radio for another, a separate receiver for aircraft. SDR changes that completely.

**Software Defined Radio (SDR)** replaces most of the hardware components of a radio with software running on a computer. One piece of hardware captures a wide chunk of the radio spectrum as raw data. Software then does the signal processing - tuning, filtering, demodulation - in real time. Switch the software configuration and you're listening to a completely different signal type on a completely different frequency.

---

## The Hardware

You don't need much to get started.

**RTL-SDR** is the entry point. Originally a cheap DVB-T TV tuner dongle, someone discovered it could be put in a raw capture mode that makes it a wideband receiver. Around $35, covers roughly 500 kHz to 1.75 GHz. Receive only - can't transmit. Enough to get started with almost everything in this module.

**HackRF One** is the step up. Covers 1 MHz to 6 GHz, can both receive and transmit (half-duplex - one at a time). Around $300-350. This is what you use when you need to generate signals for research purposes.

**BladeRF** and **USRP** are higher-end platforms used in professional research. Full-duplex (transmit and receive simultaneously), higher dynamic range, used in academic and commercial RF research.

| Hardware | Frequency Range | TX/RX | Price |
|----------|----------------|-------|-------|
| RTL-SDR | ~500 kHz - 1.75 GHz | RX only | ~$35 |
| HackRF One | 1 MHz - 6 GHz | TX + RX (half) | ~$340 |
| BladeRF | 300 MHz - 3.8 GHz | TX + RX (full) | ~$650 |

For learning, start with RTL-SDR. You can do a surprising amount with receive-only hardware, and it costs almost nothing.

---

## Software

```bash
# Install core SDR tools on Linux
sudo apt install rtl-sdr gqrx-sdr

# Test your RTL-SDR dongle
rtl_test

# Check what SDR devices are connected
rtl_eeprom

# Command-line FM radio receiver
rtl_fm -f 100.1M -M wbfm -s 200000 -r 48000 - | aplay -r 48000 -f S16_LE
```

**GQRX** is the main GUI for general SDR exploration on Linux. It shows a live waterfall display - frequency on the horizontal axis, time going downward, signal strength shown as color. You can see every signal in your range at once.

**SDR#** (SDRSharp) is the Windows equivalent, very beginner friendly.

**GNU Radio** is the serious platform for building custom signal processing pipelines. Steeper learning curve but it's what researchers use for complex work.

---

## Waterfall Display

The waterfall is the fundamental SDR visualization and worth understanding:

```
Frequency (Hz) -->
900MHz    920MHz    940MHz    960MHz    980MHz
  |         |         |         |         |
  |   GSM   |         |   GSM   |         |  <- time 0
  |  |||||| |         |  |||||| |         |
  |  |||||| |         |  |||||| |         |  <- time 1
  |         |    FM   |         |  pager  |
  |         |  ~~~~~  |         |  |||||  |  <- time 2
```

Bright colors mean strong signal. Dark means nothing. Signals appear as vertical streaks (continuous transmission) or scattered bursts (intermittent). Once you recognize the visual patterns, you can identify signal types by shape before you even demodulate them.

---

## ADS-B - Tracking Aircraft

ADS-B (Automatic Dependent Surveillance-Broadcast) is the system modern aircraft use to broadcast their position, altitude, speed, heading, and flight number. They transmit on 1090 MHz every second or so. No encryption. Completely open.

`dump1090` decodes these signals and can display aircraft positions on a map in your browser.

```bash
# Install dump1090
sudo apt install dump1090-mutability

# Or build from source
git clone https://github.com/antirez/dump1090
cd dump1090
make

# Start receiving (RTL-SDR plugged in)
./dump1090 --interactive --net

# Open browser to see aircraft on a map
# http://localhost:8080
```

In a populated area you'll typically see dozens of aircraft at any time. Each one is broadcasting:
- ICAO address (unique aircraft identifier)
- Flight number
- GPS position (latitude, longitude)
- Altitude
- Ground speed
- Vertical rate

This is legal to receive everywhere - it's publicly broadcast aviation safety data. FlightAware and Flightradar24 are built on crowdsourced ADS-B receivers like this.

---

## Other Signals Worth Receiving

With an RTL-SDR you can receive a lot:

**ACARS** - text messages between aircraft and ground stations. Flight plans, weather, maintenance messages.
```bash
acarsdec -r 0 131.550
```

**AIS** - ship tracking. Same concept as ADS-B but for marine vessels on 161.975 MHz and 162.025 MHz.
```bash
rtl_fm -f 161.975M | aisdecoder
```

**NOAA Weather Satellites** - the NOAA polar-orbiting weather satellites transmit images on 137 MHz. You can receive actual satellite images of cloud cover.
```bash
# Receive and decode NOAA APT weather images
rtl_fm -f 137.620M -s 60000 | sox -t raw -r 60000 -e signed -b 16 -c 1 - output.wav
# Then decode the audio file with noaa-apt or WXtoImg
```

**Pagers** - still used widely in hospitals and industrial facilities. Transmit on frequencies around 152-170 MHz and 929-932 MHz. Completely unencrypted.
```bash
multimon-ng -t raw -a POCSAG512 -a POCSAG1200 -a FLEX <(rtl_fm -f 152.240M -s 22050 -)
```

**TPMS** - tire pressure sensors on cars broadcasting their readings.
```bash
rtl_433 -f 315M
```

`rtl_433` is a general-purpose decoder for the 433 MHz ISM band where a huge number of IoT sensors, weather stations, and consumer devices transmit.

---

## Transmitting - The Legal Boundary

Everything above is receive-only. Transmitting is a completely different situation.

In most countries, transmitting on radio frequencies requires a license. The specific frequencies you can use, the power levels, and the permitted uses are regulated. Transmitting without authorization on:

- Aviation frequencies - interferes with aircraft safety systems. Federal crime.
- GPS frequencies - illegal signal jamming/spoofing. Serious federal offense in most countries.
- Cellular frequencies - illegal interference with licensed services.
- Emergency services - illegal, potentially dangerous.

Ham radio licenses (Technician, General, Amateur Extra in the US) give you legal access to specific amateur bands for experimentation and communication. If you want to transmit legally for research purposes, getting a ham license is the correct path. The Technician exam is not difficult and covers exactly the regulatory knowledge you need.

For research that involves transmitting in controlled conditions, the standard approach is a **shielded enclosure (Faraday cage)**. Signals stay inside, nothing leaks out, no interference with real systems.

```
A Faraday cage for RF research:
- Metal box or room with conductive mesh walls
- All signals stay inside
- You can transmit at full power with no external interference
- Used in legitimate RF security labs everywhere
```

---

## GPS Spoofing - Concept and Implications

GPS receivers work by listening to signals from satellites and calculating position based on timing differences. The signals are extremely weak by the time they reach Earth - about 20 watts transmitted from 20,000 km up, arriving at around -130 dBm. A locally generated signal at even very low power can overpower the real satellite signals.

GPS spoofing means transmitting fake GPS signals that make a receiver calculate a false position. The receiver has no way to distinguish a spoofed signal from a real one - GPS was designed with no authentication.

This is a documented real-world problem. Researchers have demonstrated it against drones, ships, and phones. GPS spoofing near airports, harbors, and in conflict zones is a documented phenomenon affecting navigation systems globally. In 2019, ships in the Black Sea reported GPS positions placing them inland, miles from their actual location.

The implications for security research: any system that trusts GPS-derived location for authentication, geofencing, or navigation is potentially vulnerable to position falsification. This matters for drone delivery systems, autonomous vehicles, financial trading systems that timestamp transactions with GPS, and any location-dependent access control.

Transmitting GPS spoof signals without authorization is illegal in virtually every jurisdiction. The research value is in understanding that GPS position cannot be trusted as a ground truth for security-sensitive applications.

---

## Quick Reference

| Tool | What it does |
|------|-------------|
| `rtl_test` | Test RTL-SDR hardware |
| `gqrx` | GUI SDR receiver with waterfall display |
| `dump1090` | Decode ADS-B aircraft transponders |
| `rtl_fm` | Command-line FM/AM receiver |
| `rtl_433` | Decode 433 MHz ISM band devices |
| `multimon-ng` | Decode pager traffic and other digital modes |
| `acarsdec` | Decode aircraft ACARS messages |
| GNU Radio | Build custom signal processing pipelines |

| Signal | Frequency | Notes |
|--------|-----------|-------|
| FM Radio | 88-108 MHz | Easy first target |
| Aircraft ADS-B | 1090 MHz | Aircraft position broadcasts |
| NOAA Weather Sat | 137 MHz | Satellite image downloads |
| AIS (Ships) | 161-162 MHz | Marine vessel tracking |
| Pagers | 152-170 MHz | Often unencrypted text |
| GPS | 1575.42 MHz | L1 civilian band |

---

## Lab

```bash
# 1. Plug in RTL-SDR dongle and test it
rtl_test -t

# 2. Open GQRX
gqrx
# Tune to a local FM station (88-108 MHz)
# Switch to WFM (wide FM) demodulation
# You should hear audio

# 3. Track aircraft
dump1090 --interactive --net
# Open http://localhost:8080 in your browser
# How many aircraft can you see? What altitudes and speeds?

# 4. Decode general 433 MHz devices
rtl_433
# Leave it running - you'll likely see weather sensors, car remotes, other devices

# 5. If near a coast or major river, try AIS ship tracking
rtl_fm -f 161.975M -s 48000 | multimon-ng -t raw -a AIS -
```

**Things to think about:**

- ADS-B has no authentication. An aircraft just broadcasts its own position. What attack does that enable against air traffic management systems that trust ADS-B data?
- GPS has been used as a timestamp source for financial trading systems. If GPS position can be spoofed, what else can be spoofed along with it?
- Pager traffic in hospitals is often unencrypted. What kind of information flows over hospital paging systems, and what are the privacy implications?

---

*That's the last module. From TCP/IP fundamentals all the way to the radio frequency spectrum - you now have a solid map of the attack surface that runs the modern world.*
