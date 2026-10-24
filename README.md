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

PLUGS eliminates proprietary vendor apps on client devices through open rendering choices:

* **Zero-Install WebBluetooth (Browser Native):** The browser connects directly over BLE, reads the JSON schema, and renders controls on the fly.  
  * *Best for:* Instant, zero-friction access on phones and PCs without installing anything or joining a Wi-Fi network.

* **Embedded Flat HTML5/JS (Device-Served):** The microcontroller hosts a single-file HTML5 interface in flash memory and serves it over local Wi-Fi or USB.  
  * *Best for:* Complete offline access and rich graphical dashboards without relying on browser BLE APIs.
 
* **Host-Side Value-Add Extensions:** PLUGS natively supports host-side application layers, enabling manufacturers to build custom value-add features like contact-named passcodes, calendar scheduling, or analytics on phone and desktop software. This extension architecture gives companies full freedom for commercial software differentiation while ensuring the physical device remains 100% accessible through open, unconstrained local endpoints.
  * *Best for:* Proprietary software features, advanced local management, and commercial product differentiation.

* **Universal Aggregator Integration:** Platforms like Home Assistant, Apple Home, or Google Home ingest the schema and create control tiles automatically.  
  * *Best for:* Multi-device home automation, background scheduling, and unified dashboard control.

---

### Schema Architectural Preface: The Death of Rigid Device Classes

Traditional IoT standards (such as Matter, Zigbee, and legacy Bluetooth profiles) force hardware into rigid, pre-defined "device classes" like Door Locks, Thermostats, or Moisture Meters. When a manufacturer creates hybrid hardware or introduces novel features, these rigid committee categories break, forcing vendors to write proprietary clusters or custom mobile apps.

#### The 2010 Memory Bottleneck
This rigid taxonomy was originally an engineering necessity. In 2010, Bluetooth Low Energy microcontrollers operated under extreme physical constraints, often limited to 8 kilobytes of RAM and 20-byte wireless packet caps. Because these tiny chips could not store or transmit human-readable text schemas, manufacturers hardcoded numeric byte arrays (e.g., `0x01` = locked) and were forced to place the translation logic (the UI "Rosetta Stone") on proprietary cloud servers or single-purpose phone apps.

#### The Web-Like PLUGS Paradigm
Today, inexpensive $1.50 microcontrollers (such as the ESP32) feature megabytes of flash storage, hundreds of kilobytes of RAM, and high-speed BLE pipelines. The memory bottleneck that forced cloud dependence no longer exists.

PLUGS abandons rigid device taxonomies entirely, adopting the dynamic design model of the World Wide Web:
* **Web Browser Analogy:** Just as a web browser does not need prior knowledge of whether a site is a bank or a blog to render its buttons and text, a PLUGS viewer does not need prior knowledge of a device type to render its control panel.
* **Atomic UI Primitives:** Devices simply broadcast their available capabilities using open JSON, YAML, or CBOR arrays containing basic UI primitives (toggles, sliders, status badges, gauges, and text inputs).
* **Instant Hybrid Compatibility:** Whether hardware is a single-purpose moisture sensor or a complex hybrid generator with integrated BMS and deadbolt controls, the device self-describes its physical feature set. Generic PLUGS viewers render the interface on the fly without requiring custom drivers, vendor apps, or schema negotiation.

---

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

## Commercial and Industry Resilience

In commercial real estate, industrial facilities, and fleet management, cloud dependencies create massive financial and operational risks. Commercial hardware—such as HVAC dampers, backup generators, sub-meters, and access gates—is engineered to last 15 to 30 years, whereas cloud startups and SaaS platforms rarely survive a decade.

* **Immunity to SaaS Sunsetting & Price Hikes:** If a commercial vendor goes bankrupt, shuts down its cloud infrastructure, or imposes predatory subscription hikes, installed hardware continues operating locally without requiring a costly "rip-and-replace" project.
* **Zero-Downtime Platform Migration:** Property managers and fleet operators can transition an entire deployment of PLUGS sensors to a new software dashboard, local server, or open-source hub instantly—without dispatching technicians to re-key or swap physical devices.
* **Air-Gapped Operational Continuity:** Multi-tenant access systems, utility sub-metering, and critical machinery monitoring remain 100% operational during internet service provider outages, cloud server downtime, or WAN network failures.

---

## Supply Chain and Manufacturing Advantages

PLUGS lowers engineering and regulatory barriers for small hardware businesses, independent makers, and overseas ODMs:

* **Zero AWS and Cloud Server Liabilities:** Manufacturers burn an open PLUGS firmware module onto an inexpensive ESP32 chip (\$1.50) and ship the product.
* **No Mobile App Maintenance:** Eliminates the ongoing software debt of designing, publishing, and updating custom iOS and Android apps.
* **Streamlined Import Compliance:** Air-gapped local appliances enter import markets as standalone equipment (similar to a digital multimeter), minimizing network security liabilities and cross-border data privacy overhead.

---

### Legacy Hardware Retrofits

PLUGS bridges legacy and proprietary hardware through two distinct retrofit mechanisms depending on where the compatibility gap exists:

1. **Device-Side Hardware Bridges (For Wired / Dumb Machines):** Low-cost ($2 ESP32) inline hardware adapters attached directly to non-wireless or wired legacy equipment (via UART, RS-485, screw terminals, or USB). The bridge translates proprietary serial telemetry into open wireless PLUGS endpoints.

2. **Host-Side Software Shims (For Proprietary Wireless Devices):** Pure software translation engines running locally on the user's phone, PC, or local hub. The client software uses the host's native wireless radio to snoop and translate proprietary BLE payloads into clean PLUGS schemas in real time with zero extra hardware.

*Note: For host PCs or local servers lacking native Bluetooth hardware, a standard off-the-shelf USB Bluetooth dongle provides the necessary wireless link.*

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
