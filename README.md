# PLUGS: Proximity Local User Gateway Standard

> Your smart devices should be appliances, not surveillance relationships. A wireless standard as simple as plugging in a USB device, and far more secure.

---

## The Problem: Cloud Hostage Hardware

Modern smart hardware is broken. Consumers buy thermostats, e-bikes, smart deadbolts, and power tools, only to discover they have not bought an appliance. Instead, they have leased a mandatory cloud relationship.

* **Cloud Bricking:** When an IoT startup goes bankrupt or shuts down its servers, perfectly working hardware becomes instant e-waste.
* **App Store Deprecation:** Legacy devices become useless when 32-bit mobile apps are removed from the iOS and Android app stores.
* **Surveillance and Security Risks:** Mandatory cloud connections expose home networks to remote telemetry exfiltration, central data breaches, and unwanted third-party tracking.

---

## The Solution: PLUGS

**PLUGS** (*Proximity Local User Gateway Standard*) is an open, transport-agnostic hardware self-description standard. It shifts smart electronics from cloud-dependent software services back into standalone, local-first appliances.

Devices ship with information, not infrastructure. If the manufacturer vanishes tomorrow, a PLUGS-compliant device continues to work forever.

### Core Principles

1. **Local-First and Air-Gapped by Default:** Devices operate 100% locally over Bluetooth Low Energy (BLE GATT/ATT), local Wi-Fi, or Serial/USB. Zero WAN routing required out of the box.
2. **Self-Describing Schemas:** Devices broadcast their own UI layouts, control capabilities, metrics, and fault diagnostics using open JSON, YAML, or compressed CBOR payloads.
3. **Zero App Store Dependencies:** Render controls instantly in any standard browser via WebBluetooth or flat HTML5/JS served directly from the device's onboard flash memory.
4. **Physical Proximity Security:** Pairing and state changes require physical human intent (chassis button presses or physical toggles), eliminating remote network attacks.
5. **Earth First (Eliminating Software-Deprecated E-Waste):** Software death wipes out billions of dollars in functional consumer electronics every year, forcing millions of tons of working hardware into landfills when cloud servers shut down, APIs change, or manufacturers go out of business. PLUGS ensures hardware utility matches physical component lifespans, independent of corporate software budgets or vendor viability.

---

## Architecture and Data Flow

```text
1. HARDWARE LAYER (ESP32 / nRF52 / Microcontroller)
   Hosts local firmware and self-describing JSON schema.
   |
   +--> Local BLE GATT / Serial / Wi-Fi
   |
2. TRANSPORT LAYER (Zero Cloud Hops)
   Broadcasting structured JSON/YAML or flat HTML5.
   |
   +--> Direct Local Transmission
   |
3. CLIENT / RENDER LAYER (Phone, PC, or Local Hub)
   WebBluetooth / PWA / Home Assistant renders UI directly from schema.
```

### Phone and PC Client Software Layer

PLUGS eliminates proprietary vendor apps on client devices through three open rendering choices:

* **Zero-Install WebBluetooth (Browser Native):** The browser connects directly over BLE, reads the JSON schema, and renders controls on the fly.  
  * *Best for:* Instant, zero-friction access on phones and PCs without installing anything or joining a Wi-Fi network.

* **Embedded Flat HTML5/JS (Device-Served):** The microcontroller hosts a single-file HTML5 interface in flash memory and serves it over local Wi-Fi or USB.  
  * *Best for:* Complete offline access and rich graphical dashboards without relying on browser BLE APIs.

* **Universal Aggregator Integration:** Platforms like Home Assistant, Apple Home, or Google Home ingest the schema and create control tiles automatically.  
  * *Best for:* Multi-device home automation, background scheduling, and unified dashboard control.

---

## Device Schema Standard

A PLUGS-compliant device broadcasts a lightweight, human-readable schema over BLE GATT characteristics or serial text streams.

### Example: Smart Deadbolt Schema (`deadbolt.json`)

```json
{
  "plugs_version": "1.0",
  "device": {
    "name": "Front Door Lock",
    "type": "smart_deadbolt",
    "model": "PLUGS-LOCK-V1",
    "firmware": "1.0.4"
  },
  "controls": [
    {
      "id": "lock_state",
      "label": "Door Lock",
      "type": "toggle",
      "options": ["LOCKED", "UNLOCKED"],
      "current": "LOCKED"
    }
  ],
  "telemetry": [
    {
      "id": "battery_level",
      "label": "Battery",
      "type": "percentage",
      "value": 88
    },
    {
      "id": "tamper_status",
      "label": "Tamper Sensor",
      "type": "status",
      "value": "NORMAL"
    }
  ],
  "security": {
    "pairing_method": "PHYSICAL_BUTTON_PRESS",
    "air_gapped": true
  }
}
```

---

## Security Architecture and Air-Gap Standards

Physical USB ports expose host machines to electrical destruction, BadUSB keystroke injection, and Direct Memory Access (DMA) exploits. PLUGS delivers the plug-and-play simplicity of USB with superior security insulation:

* **Zero Copper Contact:** All proximity communication occurs over sandboxed wireless airwaves (BLE or local Wi-Fi). High-voltage spikes from malicious hardware cannot travel across radio waves to damage a phone or computer motherboard.
* **BadUSB and DMA Immunity:** PLUGS devices never mount as Human Interface Devices (keyboards/mice) or system controllers. The host OS parses plain, sandboxed text payloads (JSON/YAML/HTML), preventing hardware-level command injection.
* **Physical Intent Gatekeeping:** Administrative functions and new client pairings require an explicit physical action on the device (a button press or chassis toggle), blocking unauthorized remote access.

---

## Commercial Fleet and Multi-Tenant Resilience

In commercial environments (such as apartment buildings, fleets of diesel generators, or HVAC systems), cloud dependency creates severe financial and operational risks.

* **Single-Point-of-Failure Elimination:** If a SaaS vendor hosting 10,000 generator or sub-meter sensors goes bankrupt, those devices do not become landfill.
* **Seamless Platform Re-Housing:** Property managers can transition 10,000 existing PLUGS sensors to a new management platform or open-source hub in minutes without replacing a single piece of installed hardware.

---

## Supply Chain and Manufacturing Advantages

PLUGS lowers engineering and regulatory barriers for small hardware businesses, independent makers, and overseas ODMs:

* **Zero AWS and Cloud Server Liabilities:** Manufacturers burn an open PLUGS firmware module onto an inexpensive ESP32 chip (\$1.50) and ship the product.
* **No Mobile App Maintenance:** Eliminates the ongoing software debt of designing, publishing, and updating custom iOS and Android apps.
* **Streamlined Import Compliance:** Air-gapped local appliances enter import markets as standalone equipment (similar to a digital multimeter), minimizing network security liabilities and cross-border data privacy overhead.

---

## Legacy Retrofit Engine

PLUGS supports legacy hardware through two retrofit mechanisms:

* **Software Shims:** Software utilities utilize BLE snoop logging and schema synthesis to reverse-engineer proprietary Bluetooth devices into open PLUGS JSON profiles.
* **Hardware Shims:** Inexpensive (\$2) inline ESP32 dongles attach to legacy serial ports or diagnostic headers to expose standard PLUGS GATT characteristics.

---

## Repository Layout

```text
.
├── README.md              <-- Specification overview and manifesto
├── SPECIFICATION.md       <-- Complete technical protocol specification
├── LICENSE                <-- MIT Open Source License
├── schemas/               <-- Reference JSON/YAML device definition schemas
│   ├── deadbolt.json
│   ├── generator_bms.json
│   └── thermostat.json
├── viewers/               <-- WebBluetooth PWA reference implementation
└── firmware/              <-- ESP32/Arduino C++ helper libraries
```

---

## Development Roadmap

* **Phase 1: Protocol and Schema Specification** (Core protocol and schema definition freeze)
* **Phase 2: Universal WebBluetooth Reference PWA** (Browser-based PWA rendering engine)
* **Phase 3: Firmware Helper Libraries** (ESP32, C++, and MicroPython reference code)
* **Phase 4: Home Assistant Integration Module** (Native component for open-source home automation)

---

## Contributing and License

We welcome contributions from embedded software engineers, hardware designers, right-to-repair champions, and cybersecurity researchers.

* **Full Specification:** See [`SPECIFICATION.md`](./SPECIFICATION.md) for full protocol definitions.
* **License:** PLUGS is released as an open standard under the **MIT License**.
