
# Transceiver Management Interface

As established in the [previous document](01_README_module.md), a pluggable transceiver connects to its host through three distinct interfaces: power, data, and management. While the power interface energizes the module and the data interface carries high-speed network traffic over SerDes lanes, the management interface allows the host to identify, monitor, and control the module over a low-speed serial bus. Because this bus is physically separate from the data lanes, management is independent of traffic flow — the host can discover and configure a module before enabling traffic, and continue monitoring health even if the data path is down.

Without a management interface, a transceiver would be a black box. The host would have no way to determine the module's vendor, speed capabilities, or media type. It could not monitor operating temperatures, detect a failing laser, or disable a transmitter. The management interface transforms a pluggable module from a passive electrical-to-optical converter into an intelligent device that the host can discover, supervise, and configure through software.


## Static vs. Dynamic Data

To understand how management interfaces are structured, it is essential to distinguish the two types of data they handle.

**Static data** represents immutable manufacturing and identity information stored in non-volatile memory. It allows the host to identify the module and apply the correct configuration. Examples: vendor name, part number, serial number, manufacturing date, hardware revision, cable length, and media type.

**Dynamic data** represents real-time operating conditions. The module continuously samples internal sensors, and the host retrieves the latest readings on demand. Examples: module temperature, supply voltage (Vcc), laser bias current, transmit/receive optical power, and DSP metrics such as pre-FEC BER.


## Evolution of the Management Interface

Over the last two decades, the transceiver management interface underwent two parallel transformations:

- **Physical implementation**: Transitioned from standalone passive EEPROM chips to embedded microcontrollers (MCUs) with flash memory and RAM.
- **Software contract**: Evolved from serving static identity bytes to handling live diagnostics, complex paging, and state machines.

Despite these internal changes, the host has always used the same physical bus: I²C (SDA/SCL), located on the management pins described in the [QSFP28 Transceiver Interface](./01_README_module.md#qsfp28-transceiver-interface) section. Network Operating Systems and diagnostic tools still refer to reading this data as an "EEPROM read" because the interface emulates a memory map even when a modern MCU is responding.

| Specification  | Era   | Typical Form Factors  | Key Advancement |
|----------------|-------|-----------------------|-----------------|
| MSA / INF-8074 | ~2000 | SFP                   | Static identity only (`A0h` EEPROM) |
| SFF-8472       | ~2007 | SFP, SFP+, SFP28      | Digital Optical Monitoring via second address (`A2h`) |
| SFF-8636       | ~2013 | QSFP+, QSFP28         | Multi-lane paged register map with per-lane control |
| CMIS           | ~2017 | QSFP-DD, OSFP, SFP-DD | Firmware state machines, application selection, CDB |

### MSA / INF-8074: Identity Only

The earliest pluggable modules were simple optical assemblies with minimal configurability. Their management interface, defined by INF-8074, required only a passive EEPROM at I²C address `A0h`. The host read this fixed address to retrieve factory-programmed static data such as vendor name and serial number. No processing logic existed between the host and the data.

*Example:* The [Cisco GLC-SX-MM](https://www.telquestintl.com/site/Product%20Manuals/Cisco%20GLC-SX-MM%20data%20sheet.pdf) — a legacy 1G short-reach optic that acts purely as a passive EEPROM, answering only at `A0h` with no diagnostic monitoring.

### SFF-8472: Digital Optical Monitoring

As optics became more sophisticated, network operators needed real-time health monitoring. However, changing the original `A0h` memory map would break backward compatibility. SFF-8472 solved this by adding a second I²C address (`A2h`) dedicated to Digital Diagnostic Monitoring (DDM/DOM). Modules began incorporating small controllers to sample sensors and populate this new address space.

*Example:* The Cisco SFP-10G-SR — a 10G short-reach optic with internal monitoring hardware that provides live temperature, bias current, and optical power readings through the `A2h` address.

### SFF-8636: Multi-Lane Control

The advent of 40G/100G QSFP modules introduced four host lanes, per-lane TX disable, and paged memory. SFF-8636 standardized a new register layout using a single I²C address with page-select, consolidating identity, diagnostics, and control into one address space.

*Example:* The Cisco QSFP-100G-SR4-S — a 100G optic driving four parallel lanes (4×25G) with an internal MCU that exposes per-lane diagnostics and independent transmitter control.

### CMIS: Firmware-Centric Management

With 400G/800G modules came eight host lanes, complex DSPs, multiple operating modes, and field-upgradable firmware. CMIS assumes the presence of an MCU running firmware and defines banked memory, formal state machines, and standardized initialization sequences.

*Example:* The Cisco QDD-400G-DR4-S (QSFP-DD) — a 400G optic that runs live firmware, uses state machines to initialize high-power components, and negotiates operational modes with the host.


---


## SFF-8472

**Full name:** SFF-8472 — Diagnostic Monitoring Interface for Optical Transceivers

**Published by:** SFF Committee (under SNIA — Storage Networking Industry Association)

| Form Factor | Typical Data Rate |
|-------------|-------------------|
| SFP         | 1G                |
| SFP+        | 10G               |
| SFP28       | 25G               |

SFF-8472 is the management interface standard for single-lane SFP-family transceivers. It extends the original MSA/INF-8074 baseline by adding a second I²C address (`A2h`) dedicated to real-time diagnostics, while preserving legacy identification data at the original address (`A0h`). This dual-address design introduced live telemetry without modifying the existing memory layout.

### Memory Model

SFF-8472 presents two independent flat 256-byte memory spaces on the I²C bus:

- **Address `A0h` (device `0x50`)** — Module identification: vendor name, part number, serial number, connector type, compliance codes, wavelength, and link length.
- **Address `A2h` (device `0x51`)** — Diagnostics and control: real-time sensor readings, alarm/warning thresholds, status flags, and control bits.

### How the Host Reads Bytes

The host picks a 7-bit device address, sets a byte offset (0–255), and issues a standard I²C read for one or more consecutive bytes. Multi-byte numeric fields are big-endian (most significant byte first).

- Reading the vendor name: `read(A0h, offset=20, len=16)`.
- Reading module temperature: `read(A2h, offset=96, len=2)`.

On Linux hosts and SONiC, each address appears as its own sysfs node (e.g., `…-0050/eeprom` for A0h and `…-0051/eeprom` for A2h). Software opens the file, seeks to the byte offset, and reads the required length. Identity dumps touch only A0h; monitoring daemons poll A2h repeatedly. Writes are only needed for control operations (e.g., TX Disable at A2h byte 110).

### A0h: Module Identification

The A0h address space contains factory-programmed static data:

| Byte(s) | Field                      | Description        |
|---------|----------------------------|--------------------|
| 0       | Identifier                 | Module type (e.g., `0x03` = SFP/SFP+/SFP28). |
| 2       | Connector Type             | Physical connector (e.g., LC, copper pigtail). |
| 3–10    | Compliance Codes           | Supported standards (Ethernet, Fibre Channel, SONET/SDH). |
| 12–19   | Link Length                | Maximum reach for each supported fiber type and wavelength. |
| 20–35   | Vendor Name                | 16-byte ASCII vendor identification. |
| 40–55   | Vendor Part Number         | 16-byte ASCII part number. |
| 60–61   | Wavelength                 | Laser wavelength in nm (or copper attenuation for passive cables). |
| 63      | CC_BASE                    | Checksum over bytes 0–62 for data integrity verification. |
| 68–83   | Vendor Serial Number       | 16-byte ASCII serial number. |
| 84–91   | Date Code                  | Manufacturing date in YYMMDD format plus vendor lot code. |
| 92      | Diagnostic Monitoring Type | DDM capabilities: internal/external calibration, power measurement type. |
| 95      | CC_EXT                     | Checksum over bytes 64–94. |

Byte 92 is critical for host software. It indicates whether the module supports DDM, whether diagnostic values at A2h are internally calibrated (directly usable) or externally calibrated (requiring host-side computation), and whether received power is measured as average power or OMA (Optical Modulation Amplitude).

### A2h: Digital Diagnostic Monitoring (DDM)

The A2h address space is organized into three functional regions.

**Alarm and warning thresholds (bytes 0–39):** Factory-set limits for each monitored parameter. Each of the five parameters has four thresholds — high alarm, low alarm, high warning, low warning — stored as 2 bytes each (5 × 4 × 2 = 40 bytes).

**Real-time diagnostic values (bytes 96–105):**

| Byte(s) | Parameter            | Format          | Resolution |
|---------|----------------------|-----------------|------------|
| 96–97   | Temperature          | Signed 16-bit   | 1/256 °C |
| 98–99   | Supply Voltage (Vcc) | Unsigned 16-bit | 100 µV |
| 100–101 | TX Bias Current      | Unsigned 16-bit | 2 µA |
| 102–103 | TX Output Power      | Unsigned 16-bit | 0.1 µW |
| 104–105 | RX Received Power    | Unsigned 16-bit | 0.1 µW |

**Status and control (bytes 110–117):**

| Byte(s) | Function |
|---------|----------|
| 110     | TX Disable, TX Fault, RX LOS (Loss of Signal), and Data Ready bits. |
| 112–113 | Alarm flags — one bit per parameter per direction indicating threshold exceeded. |
| 116–117 | Warning flags — same structure for warning-level thresholds. |

### Calibration

SFF-8472 defines two calibration modes, indicated by bits in A0h byte 92:

- **Internal calibration:** The module converts raw sensor readings into calibrated values before writing them to A2h. The host reads final values directly. This is the standard mode for modern SFP+ and SFP28 modules.

- **External calibration:** The module writes raw ADC counts to A2h and provides calibration constants (slope and offset) in A2h bytes 56–91. The host must apply these constants to compute meaningful values. Found in some legacy or low-cost modules.

### Limitations

SFF-8472 carries constraints that made it unsuitable for the multi-lane era:

- **Single lane only** — No per-lane monitoring or per-lane TX disable.
- **Flat memory** — Each address space is a fixed 256-byte block with no paging. All data must fit within 512 total bytes.
- **No state machine** — No standardized initialization sequence or data path transition management.
- **Dual-address consumption** — Using two I²C addresses per module reduces available address space on shared buses.

These constraints drove the development of SFF-8636.


---


## SFF-8636

**Full name:** SFF-8636 — Management Interface for Cabled Environments

**Published by:** SFF Committee (under SNIA — Storage Networking Industry Association)

| Form Factor | Typical Data Rate |
|-------------|-------------------|
| QSFP+       | 40G               |
| QSFP28      | 100G              |
| Some QSFP56 | 200G              |

SFF-8636 is the management interface standard for multi-lane QSFP-family transceivers. Where SFF-8472 used two I²C addresses for a single-lane module, SFF-8636 consolidates identification, diagnostics, and control into a single I²C address (`A0h`) with a **page-based** memory model. This redesign was necessary because QSFP modules have four electrical lanes, each requiring independent monitoring and control — data that could not fit within SFF-8472's flat 512-byte layout.

### Memory Model

SFF-8636 uses a single I²C address (`A0h`, device `0x50`) with a 256-byte address window divided into two halves:

- **Lower Page (bytes 0–127):** Always accessible regardless of page selection. Contains real-time status, interrupt flags, diagnostic values, and control registers.
- **Upper Memory (bytes 128–255):** A sliding view controlled by the **Page Select** byte at address 127 (0x7F). The host writes a page number to select which 128-byte block appears in this region.

The lower page provides latency-free access to time-critical data, while upper pages store less frequently accessed information.

**Flat Memory vs. Paged Memory:** Bit 2 of byte 2 indicates the memory mode:
- **Flat memory (bit = 1):** All data fits within the lower page and upper page 00h. No page switching needed. Common in passive copper assemblies (DACs).
- **Paged memory (bit = 0):** Multiple upper pages are available. The host must set the page-select byte before accessing data beyond page 00h. Active optical modules use this mode.

### How the Host Reads Bytes

Everything is accessed through one I²C device at A0h (`0x50`). The module always presents a 256-byte address window (offsets 0–255).

**Lower page (bytes 0–127)** is always visible. Routine monitoring is a direct read — for example, bytes 22–23 for temperature, 34–57 for per-lane optical power, and bytes 3–14 for alarm flags.

**Upper page (bytes 128–255)** requires page selection. Before reading, the host writes the desired page number to byte 127, then reads offsets 128–255. For identity data: write `0x00` to byte 127, then read 128–255. For alarm thresholds: write `0x03` to byte 127, then read 128–255 again. Lower-page bytes are unaffected by page switches.

On SONiC and other Linux systems, this is typically a single sysfs file (e.g., `/sys/bus/i2c/devices/i2c-30/30-0050/eeprom`). Lower-page reads work with read-only access; upper pages require write permission because page select is a write to byte 127.

### Lower Page (Bytes 0–127): Status, Diagnostics, and Control

**Identification and status (bytes 0–2):**

| Byte | Field      | Description |
|------|------------|-------------|
| 0    | Identifier | Module type (`0x0D` = QSFP+, `0x11` = QSFP28). |
| 1    | Revision   | SFF-8636 revision level supported by the module. |
| 2    | Status     | Flat memory indicator (bit 2), interrupt status (bit 1), Data Not Ready (bit 0). |

**Interrupt and flag registers (bytes 3–21):** Each flag is a single bit indicating whether the corresponding parameter has exceeded its threshold.

| Byte(s) | Function |
|---------|----------|
| 3       | Per-lane RX LOS and TX LOS flags. |
| 4       | Per-lane TX Fault flags. |
| 6       | Module-level temperature alarm and warning flags. |
| 7       | Module-level Vcc alarm and warning flags. |
| 9–10    | Per-lane RX power alarm and warning flags. |
| 11–12   | Per-lane TX bias current alarm and warning flags. |
| 13–14   | Per-lane TX power alarm and warning flags. |

**Module-level diagnostics (bytes 22–27):**

| Byte(s) | Parameter |
|---------|-----------|
| 22–23   | Module temperature (signed 16-bit, 1/256 °C). |
| 26–27   | Supply voltage (unsigned 16-bit, 100 µV). |

Temperature and supply voltage are module-wide values because the entire module shares a single thermal sensor and power rail.

**Per-lane diagnostics (bytes 34–57):**

| Byte(s) | Parameter |
|---------|-----------|
| 34–41   | RX power, lanes 1–4 (4 × 2 bytes, unsigned 16-bit, 0.1 µW). |
| 42–49   | TX bias current, lanes 1–4 (4 × 2 bytes, unsigned 16-bit, 2 µA). |
| 50–57   | TX power, lanes 1–4 (4 × 2 bytes, unsigned 16-bit, 0.1 µW; availability varies). |

Per-lane monitoring is a key improvement over SFF-8472. A failing lane can be identified without shutting down the entire port.

**Control registers:**

| Byte | Field            | Description |
|------|------------------|-------------|
| 86   | TX Disable       | Bits 0–3 map to lanes 1–4. Setting a bit disables that lane's transmitter. |
| 87   | RX Rate Select   | Per-lane RX rate selection (for rate-selectable modules). |
| 88   | TX Rate Select   | Per-lane TX rate selection (for rate-selectable modules). |
| 93   | Power Control    | High-power class enable and power override. |
| 127  | Page Select      | Selects the active upper page (0x00–0x03). |

### Upper Page Structure (Bytes 128–255)

SFF-8636 defines four upper pages:

| Page    | Function       | Content |
|---------|----------------|---------|
| **00h** | Identification | Vendor name, part number, serial number, date code, connector type, compliance codes, wavelength, cable length. |
| **01h** | Optional / AST | Application Select Table in newer revisions; reserved in many implementations. |
| **02h** | User EEPROM    | User-writable non-volatile memory for CLEI codes, internal part numbers, deployment metadata. |
| **03h** | Thresholds     | High/low alarm and warning limits for temperature, Vcc, TX bias, TX power, and RX power. |

### Upper Page 00h: Module Identification

Upper Page 00h contains factory-programmed static data with fields for multi-lane capabilities:

| Byte(s)  | Field                              | Description |
|----------|--------------------------------------|-------------|
| 128      | Identifier                           | Module type (e.g., `0x0D` = QSFP+, `0x11` = QSFP28). |
| 129      | Extended Identifier                  | Power class (1–7), CLEI support, CDR presence for TX/RX. |
| 130      | Connector Type                       | Physical connector (e.g., LC, MPO 1×12, copper pigtail). |
| 131      | 10/40G Ethernet Compliance           | Supported Ethernet standards. Value `Extended` means see byte 192. |
| 132      | SONET Compliance                     | SONET/SDH reach and rate codes. |
| 133      | SAS/SATA Compliance                  | SAS and SATA speed codes. |
| 134      | Gigabit Ethernet Compliance          | 1000BASE-T, -SX, -LX, -CX support flags. |
| 135–136  | Fibre Channel Link Length / Tech     | FC distance and transmitter technology codes. |
| 137      | Fibre Channel Transmission Media     | Media type codes for Fibre Channel. |
| 138      | Fibre Channel Speed                  | Supported FC data rates. |
| 139      | Encoding                             | Line coding (e.g., NRZ, PAM4). |
| 140      | Nominal Bit Rate                     | In units of 100 Mb/s. Value `255` means see byte 192. |
| 141      | Extended Rate Select Compliance      | Rate select version, if supported. |
| 142      | Length (SMF, km)                      | Maximum reach over single-mode fiber in km. |
| 143      | Length (OM3, 2 m)                     | Maximum reach over OM3 fiber in units of 2 m. |
| 144      | Length (OM2, 1 m)                     | Maximum reach over OM2 fiber in meters. |
| 145      | Length (OM1, 1 m)                     | Maximum reach over OM1 fiber in meters. |
| 146      | Length (Copper/OM4, 1 m)             | Maximum reach over copper or OM4 fiber in meters. |
| 148–163  | Vendor Name                          | 16-byte ASCII vendor identification. |
| 164      | Extended Module Codes                | InfiniBand speed codes (HDR, EDR, FDR, QDR, DDR). |
| 165–167  | Vendor OUI                           | 3-byte IEEE vendor identifier. |
| 168–183  | Vendor Part Number                   | 16-byte ASCII part number. |
| 184–185  | Vendor Revision                      | 2-byte ASCII hardware revision. |
| 186–187  | Wavelength                           | Laser wavelength in units of 0.05 nm. |
| 188–189  | Wavelength Tolerance                 | Tolerance in units of 0.005 nm. |
| 190      | Max Case Temperature                 | Maximum operating case temperature in °C. |
| 191      | CC_BASE                              | Checksum over bytes 128–190. |
| 192      | Extended Specification Compliance    | Actual application when byte 131 indicates `Extended` (e.g., `100GBASE-SR4`, `100GBASE-LR4`). |
| 195      | Options                              | Optional feature flags (TX Disable, TX Fault, LOS, warning support). |
| 196–211  | Vendor Serial Number                 | 16-byte ASCII serial number. |
| 212–219  | Date Code                            | Manufacturing date in YYMMDD format plus vendor lot code. |
| 220      | Diagnostic Monitoring Type           | DDM capability flags. |
| 223      | CC_EXT                               | Checksum over bytes 192–222. |
| 224–255  | Vendor Specific                      | 32 bytes reserved for vendor-proprietary data. |

**Extended Specification Compliance (byte 192):** For 100G-class modules, byte 131 typically contains the value `Extended`, indicating that the actual application identity is at byte 192. This indirection exists because the original compliance code space was designed for 10G/40G standards and ran out of room for 100G applications.

**Extended Identifier (byte 129):** Encodes the module's power class (determining how much current the host must supply) and whether the module contains CDR circuits for TX and/or RX. Power class is critical for host power budgeting — a Class 4 module draws up to 3.5W, while higher classes can exceed 5W.

### Upper Page 03h: Alarm and Warning Thresholds

Page 03h defines the threshold values corresponding to the alarm/warning flags in the lower page (bytes 3–14). Each monitored parameter has four thresholds:

| Byte(s)  | Parameter                    | Format |
|----------|------------------------------|--------|
| 128–129  | Temperature high alarm       | Signed 16-bit, 1/256 °C |
| 130–131  | Temperature low alarm        | Signed 16-bit, 1/256 °C |
| 132–133  | Temperature high warning     | Signed 16-bit, 1/256 °C |
| 134–135  | Temperature low warning      | Signed 16-bit, 1/256 °C |
| 144–145  | Vcc high alarm               | Unsigned 16-bit, 100 µV |
| 146–147  | Vcc low alarm                | Unsigned 16-bit, 100 µV |
| 148–149  | Vcc high warning             | Unsigned 16-bit, 100 µV |
| 150–151  | Vcc low warning              | Unsigned 16-bit, 100 µV |
| 176–177  | RX power high alarm          | Unsigned 16-bit, 0.1 µW |
| 178–179  | RX power low alarm           | Unsigned 16-bit, 0.1 µW |
| 180–181  | RX power high warning        | Unsigned 16-bit, 0.1 µW |
| 182–183  | RX power low warning         | Unsigned 16-bit, 0.1 µW |
| 184–185  | TX bias high alarm           | Unsigned 16-bit, 2 µA |
| 186–187  | TX bias low alarm            | Unsigned 16-bit, 2 µA |
| 188–189  | TX bias high warning         | Unsigned 16-bit, 2 µA |
| 190–191  | TX bias low warning          | Unsigned 16-bit, 2 µA |
| 192–193  | TX power high alarm          | Unsigned 16-bit, 0.1 µW |
| 194–195  | TX power low alarm           | Unsigned 16-bit, 0.1 µW |
| 196–197  | TX power high warning        | Unsigned 16-bit, 0.1 µW |
| 198–199  | TX power low warning         | Unsigned 16-bit, 0.1 µW |

When a live diagnostic value crosses one of these thresholds, the module sets the corresponding flag bit. The host polls these flags to detect out-of-range conditions without reading every diagnostic register individually.

### Initialization

SFF-8636 does not define a formal module state machine. Initialization follows an implicit sequence:

1. The module is inserted and powered via the cage's Vcc pins.
2. The host detects the module through `ModPrsL` (Module Present) going low.
3. The host deasserts `ResetL` to release the module from reset.
4. The host polls the "Data Not Ready" bit (byte 2, bit 0) until the module indicates readiness.
5. The host reads identity data from upper page 00h, then begins monitoring diagnostics in the lower page.

This implicit approach works for simple VCSEL-based optics but creates ambiguity for firmware-driven modules where readiness depends on internal initialization, DSP calibration, or thermal stabilization.

### Limitations

- **Fixed at four lanes** — The register layout assumes four electrical lanes. Eight-lane modules do not fit this model.
- **No application selection** — The host cannot choose between operating modes (e.g., 1×100G vs. 4×25G).
- **No formal state machine** — Implicit initialization creates interoperability risks.
- **No standardized firmware update** — Field upgrades require vendor-specific tools.
- **Limited page space** — Only four upper pages (00h–03h).
- **No banking** — No mechanism to replicate page structures across lane groups.

These limitations drove the development of CMIS.


---


## CMIS

**Full name:** Common Management Interface Specification

**Published by:** QSFP-DD MSA Group and OIF (Optical Internetworking Forum)

| Form Factor | Typical Data Rate  |
|-------------|--------------------|
| QSFP-DD     | 200G, 400G, 800G   |
| OSFP        | 400G, 800G         |
| SFP-DD      | 50G, 100G          |
| QSFP112     | 400G               |

CMIS addresses all SFF-8636 limitations. Rather than extending the four-lane register layout, it defines a new register architecture for higher lane counts, flexible data path configurations, and firmware-managed modules. The official [specification PDF](https://www.oiforum.com/wp-content/uploads/OIF-CMIS-05.2.pdf) is available from the OIF website.

Key characteristics:

- **Extended page model** — Lower memory (bytes 0–127) plus a large set of upper pages (0x00–0x4F and vendor-specific), selected via page and bank registers.
- **Module state machine** — Defined states (`ModuleLowPwr`, `ModuleReady`, `ModuleFault`) with explicit transitions triggered by host writes.
- **Application descriptors** — The module advertises supported configurations and the host selects one per data path.
- **Data path state machine** — Each logical data path has its own state, independent of the module state.
- **CDB (Command Data Block)** — A mailbox mechanism for firmware download, diagnostics, and vendor-specific commands.
- **VDM (Versatile Diagnostic Monitoring)** — Rich diagnostic counters beyond simple power and temperature (e.g., SNR, pre-/post-FEC BER).

### Memory Model: Paging and Banking

CMIS retains the SFF-8636 concept of a 256-byte I²C window but expands the page space significantly:

- **Lower Memory (bytes 0–127):** Fixed region, always accessible. Holds latency-sensitive status, global DOM, and the bank/page select registers.
- **Upper Memory (bytes 128–255):** A sliding view into a much larger internal memory space, controlled by page and bank selection.

<img src="../pics/paging.png" width="1000"/>

**Paging** organizes different types of data into separate 128-byte pages. **Banking** adds a higher-level hierarchy that groups lanes into sets (typically 8 per bank) and replicates the same page structure for each group. This is necessary because a single 128-byte page cannot hold per-lane data for 16 or 32 lanes.

The full addressing hierarchy is three layers deep:

| Address | Register        | Purpose |
|---------|-----------------|---------|
| 0x7E    | **Bank Select** | Selects the active lane group. |
| 0x7F    | **Page Select** | Selects the functional page within the current bank. |
| 0x80–0xFF | **Byte Address** | The specific register within the selected page/bank. |

For a 16-lane module with lanes 1–8 in Bank 0 and lanes 9–16 in Bank 1, reading per-lane data for the second group requires selecting Bank 1 before choosing the appropriate page.

### How the Host Reads Bytes

CMIS uses the same physical pattern as SFF-8636: one I²C target (usually `0x50`), a fixed lower window, and a paged upper window.

**Lower memory reads** are direct — module identifier, module state, global temperature/Vcc, and interrupt flags require no setup.

**Upper memory reads** follow a bank → page → byte sequence:
1. Write the bank number to byte 0x7E (if applicable).
2. Write the page number to byte 0x7F.
3. Read or write bytes 128–255.

Block reads of the full upper half (128 bytes) after page select are common in NOS and lab tools. Control operations use the same path — select bank/page, then write the control register.

Operations that do not fit in one byte (firmware download, CDB commands) write a command block into designated upper pages and poll status in lower memory.

### Lower Memory (Bytes 0–127)

| Address           | Register              | Description |
|-------------------|-----------------------|-------------|
| 0 (0x00)          | **Identifier**        | Module type (e.g., `0x19` = OSFP, `0x1E` = QSFP-DD). |
| 1–2 (0x01–0x02)   | **Revision & Status** | CMIS revision level, status bits, Flat Memory indication. |
| 3 (0x03)          | **Module State**      | Module State Machine status (e.g., `0x01` = Low Power, `0x03` = Ready). |
| 14–17 (0x0E–0x11) | **Monitors (DOM)**    | Real-time temperature and supply voltage. |
| 26 (0x1A)         | **Module Control**    | Low Power Mode (Force LowPwr) and Software Reset. |
| 85 (0x55)         | **Media Type**        | Media classification (optical fiber, passive copper, active copper). |
| 126 (0x7E)        | **Bank Select**       | Active memory bank for multi-lane modules. |
| 127 (0x7F)        | **Page Select**       | Active upper memory page within the selected bank. |

### Upper Memory Pages

| Page        | Function                              | Content |
|-------------|---------------------------------------|---------|
| **00h**     | Inventory                             | Vendor Name, Part Number, Serial Number, hardware revision, date code. |
| **01h**     | Configuration                         | Supported applications, host lane configuration, speeds, modulation. |
| **02h**     | Monitoring                            | Telemetry and thresholds: temperature, Vcc, TX/RX power, alarm/warning limits. |
| **03h**     | User EEPROM                           | User-reserved non-volatile memory for custom tags and deployment notes. |
| **04h**     | Laser Capabilities                    | Tunable range, grid spacing, and laser properties. |
| **10h**     | Data Path Control                     | Per-lane TX Disable, TX Squelch, output tuning, equalization. |
| **11h**     | Data Path Status                      | Per-lane LOS, LOL, TX Fault, and related flags. |
| **20h–2Fh** | VDM                                   | Advanced DSP telemetry: SNR, pre-/post-FEC BER, signal quality. |
| **30h–3Fh** | Coherent & DSP Control                | Tunable frequency, DSP parameters, grid settings. |
| **9Fh**     | CDB Command                          | Command mailbox for firmware updates and diagnostics. |
| **A0h–AFh** | CDB Payload                          | Bulk data region for CDB commands (e.g., firmware image transfer). |
| **B0h–FEh** | Vendor Custom Pages                   | Vendor-reserved for proprietary features and debugging. |
| **FFh**     | Vendor-Specific                       | Extended diagnostics and vendor-defined counters. |

### Operational Examples

**Example 1 — Reading Lane 12 TX Bias Current (16-lane module)**

Lanes 1–8 are in Bank 0; lanes 9–16 are in Bank 1.

1. Write `0x01` to byte 0x7E (select Bank 1).
2. Write `0x02` to byte 0x7F (select monitoring page).
3. Read the byte offset corresponding to Lane 12's TX bias current.

The MCU retrieves the latest sensor reading and returns the real-time value.

**Example 2 — Disabling Transmitter for Lane 12**

1. Write `0x01` to byte 0x7E (Bank 1).
2. Write `0x10` to byte 0x7F (Data Path Control page).
3. Read the TX Disable register, set the bit for Lane 12, write back.

The MCU interprets the command and disables the laser driver for that lane.

### Functional Data Categories

CMIS organizes module management into three categories reflecting the lifecycle of host–module interaction.

#### Identity — Discovery and Capabilities

The first host–module interaction is identification. The module exposes vendor name, part number, serial number, hardware revision, media type, supported operating modes, and power class. For cable assemblies (DAC/AOC), cable length is also included. This data is factory-programmed and immutable.

The host reads this information to verify compatibility, determine supported speeds and lane configurations, and confirm power requirements.

*Where it lives:* Lower Memory (module identifier, media type), Page 00h (inventory), portions of Page 01h (application advertising).

#### Health — Monitoring and Alarms

Once active, the host continuously monitors operational health through real-time DOM metrics: temperature, Vcc, TX bias current, TX power, and RX power. These values are updated internally by the module's MCU and DSP.

CMIS defines threshold values and associated alarm/warning flags. When a parameter exceeds a limit, the module sets a status bit. The host can respond by adjusting cooling, reducing power mode, logging events, or shutting down the port.

*Where it lives:* Lower Memory (global status, interrupt flags), Page 02h (thresholds and per-lane telemetry), Page 11h (per-lane LOS, LOL, TX Fault), banked pages for high lane-count modules.

#### Control — Configuration and Commands

CMIS significantly expands host-side configuration beyond basic TX Disable. The host can enable/disable high-power mode, select application profiles (APPSEL), issue software resets, and disable individual transmit lanes.

For complex operations (firmware upgrades, bulk configuration, advanced diagnostics), CMIS defines the **CDB mechanism**. The host writes a multi-byte command payload into Page 9Fh, triggers execution, and the module processes the request asynchronously. Status and results are retrieved later.

As module complexity grows — particularly for coherent optics (ZR/ZR+) — control extends to DSP parameter tuning, optical frequency selection, modulation formats, and FEC modes. CMIS accommodates this without changing the physical management interface.

*Where it lives:* Lower Memory (power mode, reset), Page 01h (application selection), Page 10h (TX Disable, lane controls), Pages 30h–3Fh (coherent optics configuration), Page 9Fh (CDB mailbox), banked pages for per-lane configuration.
