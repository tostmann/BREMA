<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="images/busware-logo-for-dark.png">
    <img src="images/busware-logo-for-light.png" alt="busware" height="44">
  </picture>
</p>

# BREMA — home automation you commission by talking to it

A blind actuator sits on the KNX bus, factory-fresh, and does nothing. You tell
an agent:

> *“A blind actuator is on the bus. Its product file is in `~/knxprod/`. Put
> channel 1 into Home Assistant.”*

What follows is what a system integrator would do in ETS — individual address,
group and association tables written in a load session, the travel-time
parameter, the Home-Assistant projection — except nobody opened ETS, installed
knxd, or created a cloud account. At the end there is a cover tile. This
happened on real hardware, in the session the firmware was developed in.

**BREMA** — **B**usware **R**ule **E**ngine **M**odel **A**ssistant — is the
engine inside the stick that makes it possible. One engine, for buses as
different as KNX-TP and 868 MHz radio.

> ### → Start here: **[BREMA for KNX](docs/brema-for-knx.md)**
>
> The first published incarnation. Install it from the browser at
> [install.busware.de/TUL/](https://install.busware.de/TUL/), get the stick at
> the [busware shop](https://shop2.busware.de/product_info.php?products_id=4).

---

## The idea

The real cost of home automation was never the hardware — it is commissioning.
Every ecosystem keeps its own tool and its own expertise: ETS for KNX, a vendor
app per cloud, a gateway per radio. The knowledge of what a device *means* —
these two octets are a temperature, that ComObject is channel 3 — is compiled
into firmware, which is why supporting a new device means waiting for a release,
and why every ecosystem is an island.

BREMA makes two moves.

**What a device means becomes data.** Device knowledge lives in the stick's
filesystem as device-class descriptions; the firmware is the machinery that runs
them. Teaching a stick a new device is authoring a description and loading it —
no rebuild, no reflash, no toolchain. The C core owns the wire, the pairing and
the API; the description layer owns everything that is a matter of
interpretation.

**A model authors that data — a gate decides.** That is the *Model Assistant* in
the name: a large language model writes the descriptions, from structured facts
rather than guesswork, and a deterministic gate decides whether one goes live.
A candidate is dry-run against captured frames before it is promoted. The model
proposes; the gate disposes — nothing invented reaches a bus.

Together the two moves change who can commission a building: the conversation at
the top of this page is the interface.

The long-form treatment — the architecture, the gated onboarding loop, worked
examples on real radio devices, and a reproducible four-model LLM comparison —
is the [Fachartikel](docs/brema-fachartikel.md) (German).

## The stick is its own MCP server

Nothing to install, no proxy, no cloud account. The firmware answers
[MCP](https://modelcontextprotocol.io) over plain HTTP on your LAN:

```
claude mcp add --transport http busware-knx http://busware-knx-<id>.local/mcp
```

The tools are the firmware's own admin verbs, generated from its registry — so
what the agent is offered cannot drift from what the device can do. A skill
resource carries the runbook; any MCP client works.

## Safe enough for a bus

Letting an agent write to a building's bus is exactly the point where good
intentions are not enough. The safety model is in the firmware, where it cannot
be talked around:

- **Mutating verbs default to dry run.** A dry run performs every *read* and
  withholds only the write, so its report is truthful about whether the real
  thing would work. Reaching the bus takes an explicit decision.
- **Danger classes per verb** — read-only, local state, changes a physical
  device, changes the stick's own security state — reported by the stick itself.
- **Delivery verdicts, not fire-and-forget.** Sent, link-acknowledged,
  application-acknowledged, application error, timeout, no bus. Feedback in Home
  Assistant is only as honest as this enum.
- **Memory writes are allow-listed, not addressed at runtime.** A wrong
  `A_Memory_Write` does not switch the wrong light — it can leave a device
  needing ETS to recover. So no address ever travels raw: only what was
  deliberately declared is writable, within declared bounds.

## How it works

```
Rule engine       application semantics: match → extract → publish → actuate
      ↑
ProtocolStack     frame ↔ intent:  cemi · zcl · esp3 · asksin · modbus_rtu · …
      ↑
Transport         bytes + framing: tp1 · 802.15.4 · cc1101 slot · rs485 · uart
```

A **backend** is a bound pair of the two lower layers — `cemi@tp1:0`,
`zcl@802154:0`, `esp3@uart:1` — registered at boot. One stick can carry several;
a device is bound to the backend it was learned on. Everything above that line
is written once, whichever bus is underneath — including one command-plane shape
for the whole family, so a single broker ACL fences actuation across every
backend, and Home-Assistant discovery as *one renderer* over protocol-neutral
device traits, not a second device model.

## Ground truth — the other half, not public yet

A model authoring decoders from an empty page invites plausible invention, and a
plausible invention on a bus moves the wrong blind. So authoring starts at a
**ground-truth service**: curated per-device-type records — classification,
capabilities, pairing procedure, codec facts — each carrying its **provenance**,
from *verified on air* down to *inferred*. When a fact is missing, the answer is
that it is missing. It runs today for the radio backends and is **not openly
available yet**.

For **KNX** the same idea is still ahead of us: the manufacturer's `.knxprod` is
the only place a ComObject number or a parameter address means anything, and
distilling it fresh in every session is work an index should do once. **Index
functions over that catalog are planned** — until then the product file stays
your input, and the firmware says so rather than inventing the mapping.

## What exists today

| Backend | Bus | Status |
|---|---|---|
| **KNX** | KNX-TP (TUL32: ESP32-C6 + NCN5130) | **published** — [BREMA for KNX](docs/brema-for-knx.md) |
| Zigbee | IEEE 802.15.4 | in development, not released |
| EnOcean | 868 MHz ESP3 | in development, not released |
| HomeMatic | BidCoS / HmIP | in development, not released |
| Modbus, M-Bus, DALI | RS485, wired | designed, not built |

Only what has been run on real hardware is offered for download. An entry in a
flasher is a promise about somebody else's device.

**Hardware:** the [TUL KNX stick](https://shop2.busware.de/product_info.php?products_id=4)
(ESP32-C6 + NCN5130, USB-C). Wi-Fi is the normal path; a **PoE W5500 extension**
adds wired networking — power and network over the one cable that reaches the
distribution cabinet anyway, no radio credentials in the box. The firmware uses
it without configuration when fitted.

## Documentation

| | |
|---|---|
| [BREMA for KNX](docs/brema-for-knx.md) | What the KNX firmware does, and how to get it. |
| [Der BREMA-Fachartikel](docs/brema-fachartikel.md) | The long-form article (German): architecture, the gated onboarding loop, worked examples, and a reproducible four-model LLM comparison. |
| [Orchestrating KNX with an LLM](docs/knx-mit-brema.md) | The full HowTo (German): a real commissioning session end to end, including the failures. |
| [install.busware.de/TUL/](https://install.busware.de/TUL/) | Web flasher — all TUL firmware options. |
| [busware.de](https://busware.de/) | The hardware maker. |

---

*Documentation repository — the firmware sources are not part of it. Issues and
corrections are welcome.*
