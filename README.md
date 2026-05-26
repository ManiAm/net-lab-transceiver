
# Transceiver

Documentation and tools for pluggable optical/copper transceivers — form factors, management interfaces (CMIS, SFF-8636, SFF-8472), and EEPROM decoding.

## Documentation

- **[The Pluggable Transceiver Model](docs/01_README_module.md):** The three-layer pluggable architecture (port, module, cable), form factor families (SFP through OSFP), the QSFP28 electrical interface, fiber optics, and cabling options (DAC, AOC, structured fiber).
- **[Transceiver Management Interface](docs/02_README_module_mgmt.md):** How the host identifies, monitors, and controls transceivers over I²C — from static EEPROMs (MSA/INF-8074) through DDM (SFF-8472), multi-lane paging (SFF-8636), to firmware-managed modules (CMIS).

## Tools

- **[xcvr_decode](eeprom_decode/)** — CLI tool to read and decode transceiver EEPROM.
