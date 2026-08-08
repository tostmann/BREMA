# BREMA — one rule engine for KNX, Zigbee, EnOcean and HomeMatic

**BREMA** is the rule engine that runs inside busware's building-automation
sticks. This repository holds its **documentation only** — no source.

> ### → Start here: **[BREMA for KNX](docs/brema-for-knx.md)**
>
> Commission KNX devices with an AI agent — individual addresses, group and
> association tables, device parameters, Home Assistant — **without ETS and
> without knxd**. The first published incarnation, and the one you can install
> today from the browser.

---

## The name

**B**usware **R**ule **E**ngine **M**odel **A**ssistant.

The first three words are the machinery: device knowledge lives as rules in the
stick's filesystem instead of compiled into a binary. The last two are the point
of it — a **large language model authors** those rules, from structured ground
truth rather than from guesswork, and a deterministic gate decides whether one is
allowed to go live. The model proposes; the gate disposes: a candidate decoder is
dry-run against captured frames before it is promoted, so nothing invented ever
reaches a bus.

## The principle: what a device *means* is data, not firmware

Every bus has the same shape of problem. A frame arrives; somewhere there is
knowledge that says these two octets are a temperature, that endpoint is
channel 3, this button press means "off". Conventionally that knowledge is
compiled into a gateway, which is why supporting a new device means a new
firmware release — and why every ecosystem ends up with its own island of
half-supported hardware.

In BREMA that knowledge lives in the stick's filesystem as **device-class
descriptions**, and the firmware is the machinery that runs them. Teaching a
stick a new device is authoring a description and loading it — no rebuild, no
reflash, no toolchain. The C core owns the wire, the pairing and the API; the
device-class layer owns everything that is a matter of *interpretation*.

That split is the whole design, and it is what makes one engine serve buses as
different as KNX-TP and 868 MHz radio.

## Three layers

```
Rule engine       application semantics: match → extract → publish → actuate
      ↑
ProtocolStack     frame ↔ intent:  cemi · zcl · esp3 · asksin · modbus_rtu · …
      ↑
Transport         bytes + framing: tp1 · 802.15.4 · cc1101 slot · rs485 · uart
```

A **backend** is a bound pair of the two lower layers — `cemi@tp1:0`,
`zcl@802154:0`, `esp3@uart:1` — registered at boot. One stick can carry several,
and a device is bound to the backend it was learned on. Everything above that
line is written once.

## One core API, whichever bus is underneath

- **Admin verbs with capability introspection.** A consumer asks the stick what
  it can do and gets an answer generated from the firmware's own registry, so it
  cannot drift from reality.
- **Danger classes and a dry-run default.** Every verb reports whether it is
  read-only, changes local state, changes a physical device, or changes the
  stick's own security state. Mutating verbs default to dry run: they perform
  every *read* and withhold only the write.
- **Delivery verdicts, not fire-and-forget.** "Did it arrive" means something
  different on every bus, so the answer is explicit: sent, acknowledged at link
  level, acknowledged by the application, application error, timeout, no bus.
  Feedback in Home Assistant is only as honest as this enum.
- **One command-plane shape for the whole family**, so a single broker ACL
  fences actuation across every backend at once.
- **Home Assistant discovery is one renderer, not the model.** Devices are
  described in protocol-neutral traits; the HA projection is an adapter over
  them. A second consumer does not mean a second device model.

## Built for agents as much as for people

Each stick is its own **[MCP](https://modelcontextprotocol.io) server** over
plain HTTP on your LAN — nothing to install on your machine, no proxy, no cloud
account:

```
claude mcp add --transport http busware-knx http://busware-knx-<id>.local/mcp
```

The tools are the firmware's admin verbs; a skill resource carries the runbook.
An agent can therefore *commission* an installation, not just switch things in
one somebody else configured — which is the interesting part, and the part that
needs the safety model above rather than good intentions.

## Ground truth — the other half, not public yet

Authoring a device decoder from an empty page invites a plausible invention, and
a plausible invention on a bus moves the wrong blind. So the authoring loop does
not start empty: it starts at a **ground-truth service** that holds curated
per-device-type records — classification, capabilities, pairing procedure, codec
facts — each carrying its **provenance**, from *verified on air* down to
*inferred*. An agent queries it before it classifies a device it has just caught,
and when a fact is missing, the answer is that it is missing. That is the
difference between a generated decoder and a guessed one.

It runs today for the radio backends (HomeMatic, Zigbee, EnOcean) and is **not
openly available yet**.

For **KNX** the same idea takes a different shape and is still ahead of us. The
manufacturer's product data is the only place where a ComObject number or a
parameter address means anything, and distilling it again in every session is
work an index should do once. **Index functions over that catalog are planned**,
so an agent can look a device up instead of parsing it — under the same
provenance discipline. Until then the product file stays your input, and the
firmware says so rather than inventing the mapping.

## Backends

| Backend | Bus | Status |
|---|---|---|
| **KNX** | KNX-TP (TUL32: ESP32-C6 + NCN5130) | **published** — [BREMA for KNX](docs/brema-for-knx.md) |
| Zigbee | IEEE 802.15.4 | in development, not released |
| EnOcean | 868 MHz ESP3 | in development, not released |
| HomeMatic | BidCoS / HmIP | in development, not released |
| Modbus, M-Bus, DALI | RS485, wired | designed, not built |

Only what has been run on real hardware is offered for download. An entry in a
flasher is a promise about somebody else's device.

## Documentation

| | |
|---|---|
| [BREMA for KNX](docs/brema-for-knx.md) | What the KNX firmware does, and how to get it. |
| [Orchestrating KNX with an LLM](docs/knx-mit-brema.md) | The full HowTo (German): a real commissioning session end to end, including the failures. |
| [install.busware.de/TUL/](https://install.busware.de/TUL/) | Web flasher — all TUL firmware options. |
| [busware.de](https://busware.de/) | The hardware. |

---

*Documentation repository. Issues and corrections are welcome; the firmware
sources are not part of it.*
