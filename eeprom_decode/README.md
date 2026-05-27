# xcvr_decode

Read and decode transceiver EEPROM — supports **CMIS** (QSFP-DD / OSFP), **SFF-8636** (QSFP/QSFP28), and **SFF-8472** (SFP).

## Requirements

- Python 3.8+
- One of:
  - Linux sysfs EEPROM access (`optoe` / `at24` at **0x50**)
  - SONiC platform SDK (Nvidia/Mellanox SN-series, others)
- On SONiC, run with **`sudo`**: upper pages need **write** access for page select (byte 127).

## Usage

On SONiC (DX010 and similar), use a port with a module present (`show interfaces transceiver presence`):

```bash
# SONiC port (auto-detects platform and selects I2C or SDK backend)
sudo python3 xcvr_decode.py Ethernet16

# JSON (structured decode) or raw hex only
sudo python3 xcvr_decode.py Ethernet16 --json
sudo python3 xcvr_decode.py Ethernet16 --raw
```

Skip optional upper pages:

```bash
sudo python3 xcvr_decode.py Ethernet16 --no-page1 --no-page2
```

Direct path or bus number (any Linux host with sysfs EEPROM):

```bash
sudo python3 xcvr_decode.py --eeprom-path /sys/bus/i2c/devices/i2c-30/30-0050/eeprom
sudo python3 xcvr_decode.py --i2c-bus 30
```

Also works as a Python module:

```bash
sudo python3 -m xcvr_decode Ethernet16
```

If reads time out on an empty cage, the tool exits with a short message (no traceback). Stop `xcvrd` only if you still cannot read a port that shows **present**:

```bash
sudo docker exec pmon supervisorctl stop xcvrd
sudo python3 xcvr_decode.py Ethernet16
sudo docker exec pmon supervisorctl start xcvrd
```

## How it works

1. **Detect platform** — reads `/etc/sonic/device_metadata.json` or `sonic-cfggen`
2. **Resolve access** — looks up `xcvr_decode.yaml` for backend type (I2C sysfs vs SDK)
3. **Read EEPROM** — lower page (0–127) + upper pages (128–255, page-select via byte 127)
4. **Auto-detect standard** — identifier byte 0 determines CMIS vs SFF-8636 vs SFF-8472
5. **Decode & format** — human-readable byte-map output with live telemetry

## Platform configuration

Edit `xcvr_decode.yaml` (next to the script, or `/etc/xcvr_decode.yaml`):

```yaml
platforms:
  x86_64-cel_seastone-r0:
    i2c_start: 26
    port_index: "eth//4"
    eeprom_template: "/sys/bus/i2c/devices/i2c-{bus}/{bus}-0050/eeprom"

  x86_64-nvidia_sn5600-r0:
    backend: sdk
    port_index: "eth//8"
    sfp_index_offset: 1
```

Resolution order:

1. `--eeprom-path` (explicit file path)
2. `--i2c-bus` (builds sysfs path)
3. `XCVR_DECODE_EEPROM_PATH` env var
4. `xcvr_decode.yaml` + auto-detected SONiC platform + `EthernetN`

## What is decoded

| Standard | Identifiers | Output |
|----------|-------------|--------|
| **CMIS 4.x/5.x** | QSFP-DD (0x18), OSFP-8X (0x19), OSFP (0x1E) | Module info, live temp/voltage/per-lane DOM, firmware, app advertising |
| **SFF-8636** | QSFP (0x0C), QSFP+ (0x0D), QSFP28 (0x11) | Byte-map + per-lane DOM table + thresholds (page 03h) |
| **SFF-8472** | SFP (0x03) | A0h identification fields |
| Unknown | Any other | Hex dump + note |

## Design principles

- **Independent** — does not rely on SONiC `xcvrd`, `sfpshow`, or vendor decoder libraries
- **Standard-driven** — follows CMIS 5.2, SFF-8636, SFF-8472, and SFF-8024 specifications
- **Modular** — each standard has its own decoder + formatter; shared code is minimal
- **Graceful degradation** — passive copper DACs (flat memory) are handled without errors

## Sample output

```bash
sudo python3 xcvr_decode.py Ethernet16
```

<details>
<summary>Sample output</summary>

```text
========================================================================
========================== ACCESS / I2C PATH ===========================
========================================================================
  interface: Ethernet16
  eeprom_path: /sys/bus/i2c/devices/i2c-30/30-0050/eeprom
  backend: config:x86_64-cel_seastone-r0
  eth: 16
  qsfp_index: 4
  i2c_bus: 30
========================================================================
======================== SFF-8636 EEPROM DECODE ========================
========================================================================
  Byte addresses are offsets on I2C A0h (0x50). Lower page 0–127; upper 128–255 per page select.
  Unmapped bytes appear as ** Reserved ** with hex.

========================================================================
======================= LOWER PAGE  BYTES 0–127 ========================
========================================================================

  ── Module identity ──
  [  0]       Identifier: QSFP28 or later
  [  1]       Revision compliance: SFF-8636 Rev 2.8–2.10

  ── Status ──
  [  2]       Status: 0x02 — DNR=False IntL_asserted=True flat/page3bit=False

  ── Alarm & warning flags ──
  [  3]       Channel status / LOS (byte 3): no LOS flags
  [  4]       TX fault (byte 4): no TX fault
  [  5]       Reserved / vendor (byte 5): 00
  [  6]       Module alarm/warning — temperature (byte 6): 0x00 (none)
  [  7]       Module alarm/warning — Vcc (byte 7): 0x00 (none)
  [  8]       Vendor specific (byte 8): 00
  [  9]       Channel alarm/warning — RX power lanes 1–2: none
  [ 10]       Channel alarm/warning — RX power lanes 3–4: none
  [ 11]       Channel alarm/warning — TX bias lanes 1–2: none
  [ 12]       Channel alarm/warning — TX bias lanes 3–4: none
  [ 13]       Channel alarm/warning — TX power lanes 1–2: none
  [ 14]       Channel alarm/warning — TX power lanes 3–4: none
  [ 15- 21]  Reserved / vendor (bytes 15–21): 00000000000000

  ── Module monitors (live DOM) ──
  [ 22- 23]  Module temperature: 34.3125 °C
  [ 24- 25]  Reserved (bytes 24–25): 0000
  [ 26- 27]  Supply voltage Vcc: 3.2938 V

  ── Channel monitors (per lane) ──
  [ 28- 33]  Reserved (bytes 28–33): 000000000000
  [ 34- 35]  RX1 power: 1035.70 µW (0.152 dBm)
  [ 36- 37]  RX2 power: 1173.30 µW (0.694 dBm)
  [ 38- 39]  RX3 power: 1210.20 µW (0.829 dBm)
  [ 40- 41]  RX4 power: 1144.50 µW (0.586 dBm)
  [ 42- 43]  TX1 bias: 8.0060 mA
  [ 44- 45]  TX2 bias: 8.0140 mA
  [ 46- 47]  TX3 bias: 8.0100 mA
  [ 48- 49]  TX4 bias: 8.0260 mA
  [ 50- 51]  TX1 power: 1250.90 µW (0.972 dBm)
  [ 52- 53]  TX2 power: 1383.00 µW (1.408 dBm)
  [ 54- 55]  TX3 power: 1229.40 µW (0.897 dBm)
  [ 56- 57]  TX4 power: 1236.40 µW (0.922 dBm)

    Lane   RX (dBm)   TX (dBm)   TX bias (mA)   RX (µW)   TX (µW)
    ----   --------   --------   ------------   -------   -------
    1         0.152      0.972         8.0060   1035.700   1250.900
    2         0.694      1.408         8.0140   1173.300   1383.000
    3         0.829      0.897         8.0100   1210.200   1229.400
    4         0.586      0.922         8.0260   1144.500   1236.400

  ── Controls ──
  [ 58- 85]  Reserved (bytes 58–85): 00000000000000000000000000000000000000000000000000000000
  [ 86]       TX Disable: tx1_disabled=False, tx2_disabled=False, tx3_disabled=False, tx4_disabled=False
  [ 87]       RX Rate Select: lane1=default, lane2=default, lane3=default, lane4=default
  [ 88]       TX Rate Select: lane1=default, lane2=default, lane3=default, lane4=default
  [ 89- 92]  Reserved (bytes 89–92): 00000000
  [ 93]       Power Control: power_override=False, power_set=False, high_power_class_enable=False

  ── Page select & reserved ──
  [ 94-126]  Reserved (bytes 94–126): 00000000ff000000000000011f000000000000000000000000ffffffffffffffff
  [127]       Page Select: 0x00 (0)

========================================================================
========== UPPER PAGE 00H  BYTES 128–255  (PAGE SELECT = 00H) ==========
========================================================================

  ── Module type & connector ──
  [128]       Identifier: QSFP28 or later
  [129]       Extended identifier: 0xDC — class 4, CLEI=True, CDR_TX=True, CDR_RX=True
  [130]       Connector type: MPO 1x12

  ── Specification compliance ──
  [131]       10/40G Ethernet compliance: Extended
  [132]       SONET compliance: None
  [133]       SAS/SATA compliance: None
  [134]       Gigabit Ethernet compliance: None
  [135-136]  Fibre Channel link length / transmitter tech (bytes 135–136): 0000
  [137]       Fibre Channel transmission media: None
  [138]       Fibre Channel speed: None

  ── Reach & device technology ──
  [139]       Encoding: NRZ
  [140]       Nominal bit rate (100 Mb/s): 255 (255=see ext. compliance)
  [141]       Extended rate select compliance: Unspecified
  [142]       Length (SMF, km): 0
  [143]       Length (OM3, 2m): 35
  [144]       Length (OM2, 1m): 0
  [145]       Length (OM1, 1m): 0
  [146]       Length (copper or OM4, 1m): 50
  [147]       Device technology: 00

  ── Vendor identification ──
  [148-163]  Vendor name (ASCII): OEM
  [164]       Extended module codes (byte 164): N/A
  [165-167]  Vendor OUI: 000000
  [168-183]  Vendor part number (ASCII): QSFP-100G-SR4-S
  [184-185]  Vendor revision (ASCII): 04
  [186-187]  Wavelength: 850.00 nm
  [188-189]  Wavelength tolerance: 10.000 nm
  [190]       Max case temperature: 70 °C
  [191]       CC_BASE checksum: 61
  [192]       Extended specification compliance (byte 192): 100GBASE-SR4 or 25GBASE-SR
  [193]       Options (byte 193): 0f
  [194]       Options (byte 194): ff
  [195]       Options (byte 195): de

  ── Options & diagnostic capabilities ──
  [196-211]  Vendor serial number (ASCII): CS100PB0188
  [212-219]  Vendor date code (ASCII): 251108 → 2025-11-08
  [220]       Diagnostic monitoring type: 0x0C — temp=False vcc=False OMA=True tx_pwr=True tx_bias=False rx_pwr=False
  [221-222]  Enhanced options / reserved: 1068
  [223]       CC_EXT checksum: 0d

  ── Vendor-specific ──
  [224-255]  Vendor specific (bytes 224–255): 000011f32deba6c6c4b8519f8eb930eeb04e84000000000000000000ca9714f3

========================================================================
========== UPPER PAGE 01H  BYTES 128–255  (PAGE SELECT = 01H) ==========
========================================================================
  [128-143]  hex 00000000000000000000000000000000  |................|
  [144-159]  hex 00000000000000000000000000000000  |................|
  [160-175]  hex 00000000000000000000000000000000  |................|
  [176-191]  hex 00000000000000000000000000000000  |................|
  [192-207]  hex 00000000000000000000000000000000  |................|
  [208-223]  hex 00000000000000000000000000000000  |................|
  [224-239]  hex 00000000000000000000000000000000  |................|
  [240-255]  hex 00000000000000000000000000000000  |................|

  ── Decoded summary ──
    status: Empty — no application select table (all 0x00 / 0xFF)
    note: Application Select Table (optional; format is SFF-8636 rev-dependent)

========================================================================
========== UPPER PAGE 02H  BYTES 128–255  (PAGE SELECT = 02H) ==========
========================================================================
  [128-143]  hex 434d5549414c3843414131302d333134  |CMUIAL8CAA10-314|
  [144-159]  hex 322d3031563031200100000000000000  |2-01V01 ........|
  [160-175]  hex 00760000000000000000000000000000  |.v..............|
  [176-191]  hex 0000000000000000000000000000aaaa  |................|
  [192-207]  hex 515346502d313030472d5352342d5320  |QSFP-100G-SR4-S |
  [208-223]  hex 20202020000000000000000000000065  |    ...........e|
  [224-239]  hex 313333393937303731d8000000000000  |133997071.......|
  [240-255]  hex 00000000000000000000000000000000  |................|

  ── Decoded summary ──
    clei_code: CMUIAL8CAA10-3142-01V01
    part_number: QSFP-100G-SR4-S
    serial_number: 133997071
    note: User EEPROM / CLEI / vendor fields (layout is vendor-specific)

========================================================================
========== UPPER PAGE 03H  BYTES 128–255  (PAGE SELECT = 03H) ==========
========================================================================

  ── Temperature thresholds ──
  [128-129]  Temp high alarm: 85.0000 °C
  [130-131]  Temp low alarm: -10.0000 °C
  [132-133]  Temp high warning: 80.0000 °C
  [134-135]  Temp low warning: -5.0000 °C
  [136-143]  Reserved: 0000000000000000

  ── Vcc thresholds ──
  [144-145]  Vcc high alarm: 3.6000 V
  [146-147]  Vcc low alarm: 2.9000 V
  [148-149]  Vcc high warning: 3.5000 V
  [150-151]  Vcc low warning: 3.1000 V
  [152-175]  Reserved: 00000000000000000000000000000000000018300e6115b7

  ── RX power thresholds ──
  [176-177]  RX power high alarm: 4.400 dBm (raw=27542)
  [178-179]  RX power low alarm: -14.306 dBm (raw=371)
  [180-181]  RX power high warning: 3.400 dBm (raw=21877)
  [182-183]  RX power low warning: -13.298 dBm (raw=468)

  ── TX bias thresholds ──
  [184-185]  TX bias high alarm: 15.0000 mA
  [186-187]  TX bias low alarm: 0.0000 mA
  [188-189]  TX bias high warning: 12.0000 mA
  [190-191]  TX bias low warning: 2.0000 mA

  ── TX power thresholds ──
  [192-193]  TX power high alarm: 4.400 dBm (raw=27542)
  [194-195]  TX power low alarm: -10.205 dBm (raw=954)
  [196-197]  TX power high warning: 3.400 dBm (raw=21877)
  [198-199]  TX power low warning: -9.201 dBm (raw=1202)
  [200-255]  Reserved / vendor-specific: 0000000000000000000000000000000000000000000000009900000000000000000033333333222200000000000000000000000000000000
```

</details>
