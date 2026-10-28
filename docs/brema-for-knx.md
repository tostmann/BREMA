# BREMA for KNX — commission devices from an AI agent

The KNX incarnation of [BREMA](../README.md), running on the **TUL32**
(ESP32-C6 + NCN5130 TP1 transceiver). Install it from the browser at
**[install.busware.de/TUL/mcp/](https://install.busware.de/TUL/mcp/)** — that
page both describes the firmware and flashes it. It has its own flasher because
it needs one: the page reads the stick's chip identity and writes the firmware
together with a pass issued for that individual device, which the shared
installer under `/TUL/` cannot do.

The TUL32 is a hardware platform for your own projects and ships without
firmware. BREMA is a free firmware you install yourself; busware provides no
support for it.

## What it does beyond relaying group telegrams

Most gateways relay group telegrams: they talk to devices somebody else
configured with ETS. This firmware speaks the **connection-oriented management
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

**No knxd. No project file. No vendor cloud.** The manufacturer's `.knxprod` is
the one input it does need.

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
   Plain `mqtt://` or `ws://`, with user name and password if the broker wants
   them; TLS is not supported.

## Remote access

*ESP32-C6 sticks, from the firmware release after 0.25.2.* Off by default; the
local endpoint above never needs it.

Tick **Remote access (claude.ai)** on the stick's own page. The stick then dials
out to the busware relay — no port forwarding, nothing to open in the router —
and the page shows its connector URL, `https://mcp.bitbridge.eu/t/<id>/mcp`. In
claude.ai: *Settings → Connectors → Add custom connector*, paste the URL.

- **Approval happens on the stick's page.** Each connection attempt gets a
  one-time code that the relay sends to the stick, valid for five minutes. You
  read it on the stick's page and type it on the approval page — so approving
  needs what local access needs anyway: a browser on the stick's network.
- **The relay knows which stick it talks to.** A stick with a hardware key
  proves its identity with it; one without is bound on first use.
- **Same surface as on the LAN** — the same tools, skill and dry-run rules.
  The tools that change the stick's own security state (admin token, broker
  credentials) are refused through the relay and stay local-only.
- Unticked, the stick closes the connection.

When the stick's tool list changes — for example when product-data tools appear
— use *Refresh tool list* on the connector in claude.ai; it keeps the list from
the time you connected.

## Product data

What a ComObject number means, where a parameter lives in a device's memory and
what its values mean is in the manufacturer's product file (`.knxprod`), not in
the device. A separate service,
**[knxprod-mcp](https://github.com/tostmann/knxprod-mcp)**, reads the product
files you put into a folder and answers exactly those questions: which product
an ApplicationId belongs to, its com objects with every variant, its parameters
with addresses and value meanings, and what raw memory bytes decode to.

The stick forwards it (section **Product data (lab)** on its page, *ESP32-C6,
from the release after 0.25.2*):

- Run the service in your network, e.g. `docker run … tostmann/knxprod-mcp` (see
  its README). With host networking it announces itself via DNS-SD and the stick
  **finds it by itself**; otherwise enter its URL on the stick's page.
- The stick takes over the service's **read-only** `prod-*` tools and forwards
  their calls — for a client on your network and through remote access alike.
  One endpoint for the agent; the service itself stays inside your network.
- It passes on the service's own skill as well (an MCP resource,
  `skill://knxprod-mcp/SKILL.md`), so the agent learns the `prod-*` tools from
  the service itself, in the version that is running.
- The page's **upload** button sends a `.knxprod` from your browser straight to
  the service.
- The product files are the manufacturers'. You obtain them yourself and keep
  them on your own server; the service ships no manufacturer data. Check the
  manufacturer's terms — some restrict use to KNX-recognised tools.

This is a lab feature: it works end to end on the bench, the interface may still
change.

## On MQTT

- Every group telegram on the line appears raw on
  `busware/knx/rx/<main>/<middle>/<sub>` (JSON: sender, service, payload in hex),
  with or without an assigned datapoint type.
- A group address with an assigned DPT is also published decoded on
  `busware/knx/state/<group address as four hex digits>/value`, retained
  (11/0/8 → `5808`). Decoded today: DPT 1, 5 and 9.
- Commands go to `busware/knx/cmd/<hex>/value/set` (GroupValueWrite by the
  assigned DPT), `…/read/set` (GroupValueRead) and `…/cover/set`
  (OPEN/CLOSE/STOP). The stick checks no permission on this path; who may
  write there is up to the broker's ACL.
- The prefix is fixed, and these topics carry no stick identity: several sticks
  on one broker share them. A command is carried out by each stick that has a
  DPT assigned to that group address; before 0.25.2, `read` and the cover's
  open/close went out from every stick on the broker. Only
  `busware/knx/avail/<id>`, the Home-Assistant discovery topics and the admin
  plane are per stick.

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
your input, looked up on request through the [product-data service](#product-data)
when you run one. A tool that invents a mapping here would move somebody's
blinds for the wrong reason.

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

The firmware starts only with a pass issued for that individual stick. The
flasher page reads the chip, fetches the pass and writes it together with the
firmware; the stick checks it locally at every boot and, without a valid one,
halts with a clear message instead of running. The pass ties BREMA to busware
hardware, whose sales fund its development. It also keeps an image off a board
it was not built for: the KNX pins differ per product, and an image driving
foreign GPIOs on somebody's bus is the worse outcome. The board itself stays
freely flashable with anything else.

---

[← BREMA overview](../README.md) ·
[Firmware page and web flasher](https://install.busware.de/TUL/mcp/)
