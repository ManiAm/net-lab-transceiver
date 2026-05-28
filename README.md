
# Transceiver

Documentation and tools for pluggable optical/copper transceivers — form factors, management interfaces (CMIS, SFF-8636, SFF-8472), and EEPROM decoding.

## Documentation

- **[The Pluggable Transceiver Model](docs/01_README_module.md):** The three-layer pluggable architecture (port, module, cable), form factor families (SFP through OSFP), the QSFP28 electrical interface, cabling options (DAC, AOC, structured fiber), reach standards, and optical integration architectures (NPO, LPO, CPO).
- **[Transceiver Management Interface](docs/02_README_module_mgmt.md):** How the host identifies, monitors, and controls transceivers over I²C — from static EEPROMs (MSA/INF-8074) through DDM (SFF-8472), multi-lane paging (SFF-8636), to firmware-managed modules (CMIS).
- **[Fiber Optics](docs/03_README_fiber.md):** Fiber mode (MMF vs SMF), grades (OM1–5, OS1–2), bandwidth scaling (WDM vs parallel optics), connectors (LC, MPO), color coding standards, and cable examples.

## Tools

- **[xcvr_decode](eeprom_decode/)** — CLI tool to read and decode transceiver EEPROM.
