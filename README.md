# PLUGS — Proximity Local User Gateway Standard

> **Your smart devices should be appliances, not surveillance relationships.**  
> *A wireless standard as simple as plugging in a USB device, and far more secure.*

---

## The Problem: Cloud Hostage Hardware

Modern smart hardware is broken. Consumers buy thermostats, e-bikes, smart deadbolts, and power tools, only to discover they haven't bought an appliance—they've leased a mandatory cloud relationship.

- **Cloud Bricking:** When an IoT startup goes bankrupt or shuts down its AWS servers, perfectly working hardware becomes instant e-waste.
- **App Store Deprecation:** Legacy devices become useless when 32-bit mobile apps are removed from the iOS/Android app stores.
- **Surveillance & Security Risks:** Mandatory cloud connections expose home networks to remote telemetry exfiltration, central data breaches, and unwanted third-party tracking.

---

## The Solution: PLUGS

**PLUGS** (*Proximity Local User Gateway Standard*) is an open, transport-agnostic hardware self-description standard. It shifts smart electronics from cloud-dependent software services back into **standalone, local-first appliances**.

Devices ship with **information, not infrastructure**. If the manufacturer vanishes tomorrow, a PLUGS-compliant device continues to work forever.

### Core Principles
1. **Local-First & Air-Gapped by Default:** Devices operate 100% locally over Bluetooth Low Energy (BLE GATT/ATT), local Wi-Fi, or Serial/USB. Zero WAN routing required out of the box.
2. **Self-Describing Schemas:** Devices broadcast their own UI layouts, control capabilities, metrics, and fault diagnostics using open JSON, YAML, or compressed CBOR payloads.
3. **Zero App Store Dependencies:** Render controls instantly in any standard browser via **WebBluetooth** or flat HTML5/JS served directly from the device's onboard flash memory.
4. **Physical Proximity Security:** Pairing and state changes require physical human intent (chassis button presses or physical toggles)—eliminating remote network attacks.

---
