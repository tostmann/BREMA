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
write its memory, drive its Load State Machines, install its address and
association tables, and change device parameters — and then hand the result to
Home Assistant.

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

## What it will not do

**It does not guess product semantics.** What a ComObject number means, and where
a parameter lives in a device's memory, is in the manufacturer's `.knxprod` — not
in the firmware. The stick gives an agent the mechanism; the product file stays
your input. A tool that invents a mapping here would move somebody's blinds for
the wrong reason.

## Hardware

| | |
|---|---|
| **TUL32** | ESP32-C6 + NCN5130 KNX-TP transceiver, USB-C — [get it at the busware shop](https://shop2.busware.de/product_info.php?products_id=4) |
| Network | Wi-Fi is the normal path; W5500 Ethernet is optional accessory |
| Published build | ESP32-C6 only — the C3 variant compiles but its RAM headroom has not been measured |

The firmware reads the board's factory type marking and starts only on a busware
TUL. On any other ESP32-C6 board it halts with a clear message instead of
running — the KNX pins differ per product, and an image driving foreign GPIOs on
somebody's bus is the worse outcome. The board itself stays freely flashable
with anything else.

---

[← BREMA overview](../README.md) ·
[Web flasher](https://install.busware.de/TUL/) ·
[Firmware page](https://install.busware.de/TUL/mcp/)
