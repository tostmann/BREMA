# BREMA — commission KNX devices with an AI agent, without ETS

**BREMA** is the rule engine that runs on busware's building-automation sticks.
This repository holds its **documentation only** — no source. The firmware itself
is installed from the browser, at
[install.busware.de](https://install.busware.de/TUL/).

A normal KNX gateway relays group telegrams: it talks to devices that somebody
else configured with ETS. A stick running BREMA speaks the **connection-oriented
management services ETS itself uses**. It can give a device its individual
address, read and write its memory, drive its Load State Machines, install its
address and association tables, and change device parameters — and then hand the
result to Home Assistant.

**No ETS. No knxd. No project file. No vendor cloud.**

---

## What you can install today

| Stick | Bus | Firmware | Status |
|---|---|---|---|
| **TUL32** (ESP32-C6 + NCN5130) | KNX-TP | [MCP for TUL](https://install.busware.de/TUL/mcp/) | published |
| TUL (ESP32-C3 + NCN5130) | KNX-TP | — | not released: RAM headroom unmeasured |
| other busware sticks | Zigbee, EnOcean, HomeMatic, Modbus | — | in development, not released |

Only the C6 build is offered, and only because it is the one that has been run on
hardware. The rest of the family shares the same engine, but an entry in a
flasher is a promise about somebody else's device, so it waits until it is true.

## The stick is its own MCP server

There is nothing to install on your machine. The firmware answers
[MCP](https://modelcontextprotocol.io) over plain HTTP on your LAN:

```
claude mcp add --transport http busware-knx http://busware-knx-<id>.local/mcp
```

Wi-Fi is provisioned in the browser right after flashing (Improv), in the same
USB session — credentials are never compiled in. The stick then announces itself
over mDNS and serves a page with its own name, addresses and the command above.

Any MCP client works; the endpoint is JSON-RPC 2.0 over HTTP POST. The tools are
the firmware's own admin verbs, so they cannot drift from what the device can
actually do.

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
[HowTo](docs/knx-mit-brema.md) — they are the more instructive half.

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

## Documentation

| | |
|---|---|
| [Orchestrating KNX with an LLM](docs/knx-mit-brema.md) | The full HowTo (German): the real commissioning session end to end, including the failures. |
| [MCP for TUL](https://install.busware.de/TUL/mcp/) | What the firmware is, and what an agent gets. |
| [install.busware.de/TUL/](https://install.busware.de/TUL/) | Web flasher — all TUL firmware options. |
| [busware.de](https://busware.de/) | The hardware. |

---

*Documentation repository. Issues and corrections are welcome; the firmware
sources are not part of it.*
