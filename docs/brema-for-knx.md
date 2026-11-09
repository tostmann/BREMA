# BREMA for KNX — commission devices from an AI agent

The KNX incarnation of [BREMA](../README.md), published as the firmware
**MCP4KNX**, running on the **TUL32** (ESP32-C6 + NCN5130 TP1 transceiver).
Install it from the browser at
**[install.busware.de/TUL/mcp/](https://install.busware.de/TUL/mcp/)** — that
page both describes the firmware and flashes it. It has its own flasher because
it needs one: the page reads the stick's chip identity and writes the firmware
together with a pass issued for that individual device, which the shared
installer under `/TUL/` cannot do.

Two services you run in your own network complete it:
[knxprod-mcp](#product-data) for the manufacturers' product data and
[knxproject-mcp](#project) for the installation's record. Together they do the
jobs you would otherwise open ETS for —
[which ones, and where ETS is still the tool](#do-you-still-need-ets).

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

## Do you still need ETS?

In a KNX installation, ETS does four jobs: it holds the manufacturers' product
data, keeps the project, programs the devices and helps diagnose the bus. Here
they are spread over the firmware and two services you run yourself, and none
of them needs ETS:

| Job | Without ETS, here |
|---|---|
| Product data | [knxprod-mcp](#product-data) reads the `.knxprod` files you supply: which product an ApplicationId is, its com objects, its parameters with memory address and value meanings, and what raw memory bytes decode to. |
| Individual address | The stick checks that the address is free and that exactly one device is in programming mode, writes the address, verifies it and takes the device out of programming mode. |
| Group addresses | The address and association tables are built from the links and installed as one operation: load session, chunks verified by read-back, close, restart. |
| Parameters | The load procedure comes from the device's product file; the parameter is written inside a load session, verified by read-back, and the device restarted. |
| Project | [knxproject-mcp](#project) keeps the installation's record between conversations: topology, devices, group addresses, links, parameters and the owner's decisions, each with its source. |
| Diagnostics | A passive survey of every telegram on the line, an active sweep for the devices on a line, and each device's identity, properties and memory read back. |

The whole chain — address, tables, parameter, restart — is documented end to end
on a Merten 649802 blind actuator: below under
[What that looks like](#what-that-looks-like), and with every wrong turn in the
[HowTo](knx-mit-brema.md) (German). A device with another load procedure is a
new case: let the agent run it as a dry run first.

Where ETS is still the tool:

- **KNX Secure.** The stick does not speak it; a device that is to run secured
  is commissioned with ETS.
- **System B devices.** Devices with mask 07B0, OpenKNX devices among them,
  have their tables allocated at load time and in a format of their own. The
  stick so far writes tables only into fixed memory segments, so it cannot load
  them yet.
- **An ETS project to hand over.** The record cannot write a `.knxproj`, and
  cannot read one yet. It imports ETS group-address exports (ESF), ETS3 project
  databases, and Home Assistant and FHEM configurations.
- **Recovery.** A device left unusable by a wrong memory write may need ETS to
  get back.
- **Licence terms.** Some manufacturers allow their product files in
  KNX-recognised tools only. Check the terms before you load one.

## After flashing

1. Provision Wi-Fi in the installer (Improv), in the same USB session —
   credentials are never compiled in.
2. Press **Visit device** in the installer to open the stick's own page,
   `http://busware-knx-<id>.local/`: it shows its mDNS name, both addresses, its
   hardware identity and the ready-made command.
3. Point your agent at it — nothing to install:

   ```
   claude mcp add --transport http busware-knx http://busware-knx-<id>.local/mcp
   ```

4. Optionally tell it your MQTT broker; everything else works without one.
   Plain `mqtt://` or `ws://`, with user name and password if the broker wants
   them; TLS is not supported.
5. Optionally set an [admin password](#admin-password) on the stick's page.

## Admin password

*Optional, since 0.26.0.* The stick works without a password, and then anyone on
your network can use it; its page says so and offers to set one. It then answers
only under its IP address or its `.local` name, and refuses requests from other
web sites.

Once a password is set, the stick's pages and its MCP endpoint on your network
need it. Browsers sign in and hold a session; scripts and agents send
`Authorization: Bearer <password>`.

- **Forgotten:** plug the stick in again, open the installer over USB within 30
  seconds and press *Visit device*. Holding the stick is the proof; the old
  password is not needed, and wrong guesses on the network cannot lock this way
  out.
- **TULX (no USB for its user):** if the password is forgotten, the way back is
  the recovery system's factory reset, which also erases the application and its
  settings.

The stick's web server speaks plain HTTP, so a password crosses your network
unencrypted: use the stick on a network you trust. Remote access has its own
approval and does not ask for the password.

## Remote access

*ESP32-C6 only, since 0.26.0.* Off by default; the local endpoint above never
needs it.

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
since 0.26.0*):

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

## Project

What an installation *is* — which device sits where, which group address does
what, what its owner decided, what is still to do — is in no single device, and
a chat assistant forgets it between conversations. A second service,
**[knxproject-mcp](https://github.com/tostmann/knxproject-mcp)**, keeps that
record: devices, group addresses, links and parameters, each with its source (an
ETS export, the Home Assistant configuration, what the device itself holds),
plus the notes, findings and rules of the house.

The stick forwards it the way it forwards product data (section **Project
(lab)** on its page, shown once a service is found or entered; *ESP32-C6, since
0.26.0*):

- The stick takes over the service's `proj-*` tools and passes on its skill
  (`skill://knxproject-mcp/SKILL.md`), locally and through remote access. The
  agent starts with `proj-brief`.
- Unlike `prod-*`, some of these tools **write**: `proj-note` and `proj-set`
  record notes and the owner's decisions. They change **the record only, never
  the bus**.
- `proj-read-device` is the one tool with bus traffic: it reads one device
  through the stick — found by its name `busware-knx-<stick_id>.local` — and
  records what the device holds. It only reads, and only when asked to.
- Imports: an ETS group-address export (ESF), the `knx:` block of a Home
  Assistant configuration, FHEM configuration files, an ETS3 project database,
  and the replies of the stick's survey tools. **ETS5/6 project files
  (`.knxproj`) cannot be imported yet.**
- The service has no login. Anyone on your network can read and change the
  record — directly, or through a stick that has no admin password. Run it on a
  network you trust.

This is a lab feature as well.

## Running the services

Both services run as containers on a Linux machine with Docker in your network,
next to the stick, not on it — tested on a Raspberry Pi 5 (arm64) and on an
x86-64 server. One Compose file starts both:
[knx-services.compose.yml](knx-services.compose.yml).

```sh
mkdir -p knxprod knxprod-index import
curl -fLO https://raw.githubusercontent.com/tostmann/BREMA/main/docs/knx-services.compose.yml
cp ~/Downloads/*.knxprod knxprod/
KNXPROD_UID=$(id -u) KNXPROD_GID=$(id -g) docker compose -f knx-services.compose.yml up -d
```

- **Host networking** lets the stick find the services by itself: they announce
  themselves via DNS-SD (`_knxprod-mcp._tcp`, `_knxproject-mcp._tcp`), and the
  stick looks for them on its own. Tested next to avahi-daemon and Home
  Assistant's own mDNS on the same host.
  mDNS stays within one network segment, so stick and host must share it — no
  router or VLAN in between.
- **Without host networking** (Docker Desktop, for example): publish ports 8095
  and 8096 instead, enter `http://<host>:8095/mcp` and `http://<host>:8096/mcp`
  on the stick's page, and give the project service the stick's URL in
  `KNXPROJECT_STICK_URL`. The Compose file says where.
- A firewall on the host must let the stick reach ports 8095 and 8096.
- The record starts as the installation `home`. A stick with an admin password
  needs it in `KNXPROJECT_STICK_TOKEN` before `proj-read-device` can read through
  it.
- Run one service of each kind per network. With two, the stick keeps the one it
  uses while that one answers, and takes the first answer when it has to choose
  — or enter the URL on its page.

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
| **TULX** | ESP32-C6 + NCN5130, powered by the bus, no USB for its user. Takes this firmware from its recovery system: [install.busware.de/tulx/](https://install.busware.de/tulx/) |
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
