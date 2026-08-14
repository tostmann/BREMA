# BREMA for KNX — commission devices without ETS

The KNX incarnation of [BREMA](../README.md), running on the **TUL32**
(ESP32-C6 + NCN5130 TP1 transceiver). Install it from the browser at
**[install.busware.de/TUL/](https://install.busware.de/TUL/)** — pick *“MCP for
TUL”* from the dropdown. Description of the firmware:
[install.busware.de/TUL/mcp/](https://install.busware.de/TUL/mcp/)

## What it does that an ordinary KNX gateway does not

Most gateways relay group telegrams: they talk to devices somebody else
configured with ETS. This one speaks the **connection-oriented management
services ETS itself uses**. It can give a device its individual address, read and
write its memory and properties, drive its Load State Machines, install its
address and association tables, restart it so the change takes effect, and
change device parameters — and then hand the result to Home Assistant.

Installing a table is the dangerous part, so it is **one operation**: the load
session, the read-back-verified chunks, the close and the restart are a single
verb execution that nothing outside can interrupt half-way, and whose close runs
even when a write fails. A device left mid-download resolves *no* group address
at all until its table is loaded again, and a power cycle does not reliably get
it out of there — which is also why the restart belongs to the sequence rather
than to the operator's memory.

It also reads an installation it did not build: a passive survey of everything
it hears, and an active sweep that finds the devices on a line and reads back
the identity that maps each one to its product file.

**No ETS. No knxd. No project file. No vendor cloud.**

## After flashing

1. Provision Wi-Fi in the installer (Improv), in the same USB session —
   credentials are never compiled in.
2. Open the stick's own page at `http://busware-knx-<id>.local/`: it shows its
   mDNS name, both addresses, its hardware identity and the ready-made command.
3. Point your agent at it — nothing to install:

   ```
   claude mcp add --transport http busware-knx http://busware-knx-<id>.local/mcp
   ```

4. Optionally tell it your MQTT broker; everything else works without one.

## What that looks like

> **You** — A blind actuator is on the bus. Its product file is in `~/knxprod/`.
> Put channel 1 into Home Assistant.
>
> **Agent** — From the product file, channel 1 needs four group objects: move
> up/down (DPT 1.008), stop/step (1.007), position (5.001) and the position
> status it sends back (5.001). I'll use 1/1/1 … 1/1/4 unless you have an address
> plan. Dry run first — I read the device and show you the writes without
> performing them.
>
> **You** — Addresses are fine. Go.
>
> **Agent** — Built the address and association tables and wrote them in a load
> session, then restarted the device. Channel 1 answers. Home Assistant now has a
> cover tile — up, down, stop, and a position slider.
>
> **You** — Top to bottom takes 45 seconds.
>
> **Agent** — The product file puts that parameter at a known memory address,
> 16 bit, in units of 100 ms. Written in a load session and the device restarted;
> it now reports intermediate positions. I also exposed it in Home Assistant, so
> next time you type the number there instead of asking me.

This is the real sequence from the session the firmware was developed in, on a
Merten 649802 blind actuator. The wrong turns it also contained are in the
[HowTo](knx-mit-brema.md) — they are the more instructive half.

## Safety model

- **Mutating verbs default to dry run.** A dry run performs every *read* and
  withholds only the write, so its report is truthful about whether the real
  thing would work. Reaching the bus takes an explicit `dry_run=false`.
- **Danger classes** are reported per verb: read-only, local state, changes a
  physical device, changes the stick's own security state.
- **Memory writes are allow-listed, not addressed at runtime.** A wrong
  `A_Memory_Write` does not switch the wrong light — it can leave a device
  needing ETS to recover.
- **The device is asked, not assumed.** Before writing a property, the firmware
  can read that device's own description of it — writable or not, how wide, how
  many elements. Mask data from the manufacturer describes a product family; on
  the bench it marked a property writable that the device then refused, and the
  device's own answer is the one that matched reality.
- **A simulation does not report an outcome.** A dry run reports `would_write`
  where a real run reports `written`.

## What it will not do

**It does not guess product semantics.** What a ComObject number means, and where
a parameter lives in a device's memory, is in the manufacturer's `.knxprod` — not
in the firmware. The stick gives an agent the mechanism; the product file stays
your input. A tool that invents a mapping here would move somebody's blinds for
the wrong reason.

## Hardware

| | |
|---|---|
| **TUL32** | ESP32-C6 + NCN5130 KNX-TP transceiver, USB-C — [get it at the busware shop](https://shop.busware.de/tul) |
| Network | Wi-Fi is the normal path; a **PoE W5500 extension** is available for wired operation |
| Published build | Both chips — but **take the C6**. Measured on the same firmware: 147 KB largest contiguous free block against the C3's 61 KB, and serving the skill document had to be rewritten as a stream before the C3 could deliver it at all. The C3 is also being phased out. |

**Wired, where the installation is wired.** A KNX line ends in a distribution
cabinet, and a cabinet is a poor place for Wi-Fi: coverage is an accident of
sheet metal, and radio credentials end up stored in exactly the box that was
cabled for everything else. The PoE W5500 extension gives the TUL32 wired
networking with power and network over the one cable that reaches the cabinet
anyway. The firmware uses it without configuration: when the module is fitted,
the stick brings Ethernet up alongside Wi-Fi, and its own page shows the wired
address next to the wireless one.

The firmware reads the board's factory type marking and starts only on a busware
TUL. On any other ESP32-C6 board it halts with a clear message instead of
running — the KNX pins differ per product, and an image driving foreign GPIOs on
somebody's bus is the worse outcome. The board itself stays freely flashable
with anything else.

---

[← BREMA overview](../README.md) ·
[Web flasher](https://install.busware.de/TUL/) ·
[Firmware page](https://install.busware.de/TUL/mcp/)
