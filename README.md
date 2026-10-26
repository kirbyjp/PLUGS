# PLUGS: Proximity Local User Gateway Standard

> Your smart devices should be appliances, not surveillance relationships.

PLUGS is a software gateway in your pocket, allowing devices to explain themselves without proprietary apps, accounts, or cloud dependencies.

---

## The Problem: Cloud Hostage Hardware

Modern smart hardware is broken. Consumers buy thermostats, e-bikes, smart deadbolts, and power tools, only to discover they have not bought an appliance. Instead, they have leased a mandatory cloud relationship.

* **Cloud Bricking:** When an IoT startup goes bankrupt or shuts down its servers, perfectly working hardware becomes instant e-waste.
* **App Store Deprecation:** Legacy devices become useless when 32-bit mobile apps are removed from the iOS and Android app stores.
* **Surveillance and Security Risks:** Mandatory cloud connections expose home networks to remote telemetry exfiltration, central data breaches, and unwanted third-party tracking.

---

## The Solution: PLUGS

**PLUGS starts from a different assumption: the phone should become useful because it understands devices, not because every manufacturer wrote a separate app.**

**PLUGS** (*Proximity Local User Gateway Standard*) is an open, transport-agnostic hardware self-description standard. It shifts smart electronics from cloud-dependent software services back into standalone, local-first appliances.

Devices ship with information, not infrastructure. If the manufacturer vanishes tomorrow, a PLUGS-compliant device continues to work forever.

### Core Principles

1. **Local-First and Air-Gapped by Default:** Devices operate 100% locally over Bluetooth Low Energy (BLE GATT/ATT), local Wi-Fi, or Serial/USB. Zero WAN routing required out of the box.
2. **Self-Describing Schemas:** Devices describe themselves once, in a **Manifest**, using open JSON, YAML, or compressed CBOR: their controls, readings, units, and ranges. Readings and settings then travel separately as small **Values**. A device can also report plain-text fault messages in its own words.
3. **No Per-Device App Dependencies:** No manufacturer has to write, publish, or maintain an app for each product, and no owner has to install one. A single generic Viewer, in a browser (WebBluetooth where supported), on a phone or PC, or in a hub, renders any PLUGS device from its Manifest. Devices may also serve optional flat HTML5/JS from onboard flash as a presentation asset. It is never required.
4. **Physical Proximity Security:** Pairing and sensitive state changes require physical human intent (a chassis button press, a physical toggle, or, for simple low-risk devices, physically applying power), enforced by the device's own firmware, eliminating remote network attacks.
5. **Earth First (Eliminating Software-Deprecated E-Waste):** Software death wipes out billions of dollars in functional consumer electronics every year, forcing millions of tons of working hardware into landfills when cloud servers shut down, APIs change, or manufacturers go out of business. PLUGS ensures hardware utility matches physical component lifespans, independent of corporate software budgets or vendor viability.

---

## The Consumer Guarantees of Device Purchase and Ownership

When a consumer or business purchases a physical tool, that transaction represents a permanent transfer of ownership, not a conditional lease. 

### The Hammer Rule
Consider a steel hammer manufactured in 1974. If the original toolmaker goes out of business or gets acquired, that hammer sitting at a garage sale today loses zero utility. It hits a nail with the exact same reliability as a hammer manufactured yesterday. 

Modern smart hardware breaks this basic principle of physics and commerce. When a corporate cloud server is shut down, a $300 smart deadbolt, solar charge controller, or environmental sensor turns into electronic waste. 

PLUGS restores **The Hammer Rule** to digital electronics. If the physical copper, silicon, and enclosure still function, the device retains 100% of its core utility for the first, second, or tenth owner:

1. **Internet-Independent Core Functionality:** A PLUGS device shall perform all primary telemetry, monitoring, and control functions locally. Internet connectivity may provide supplemental features, data feeds, software updates, or remote access, but loss of internet access shall never prevent the device from performing its core purpose.
2. **Manufacturer Survivability:** A PLUGS device shall remain 100% operational even if the original manufacturer ceases operations, shuts down servers, or goes bankrupt.
3. **Local Self-Description:** A PLUGS device shall carry its own Manifest (control definitions, metric descriptors, and units) on internal flash memory, answering local client devices directly.
4. **Owner Wireless Control:** The device owner retains absolute authority over radio broadcast states (BLE, Wi-Fi, pairing modes) without remote cloud overrides.
5. **Frictionless Second-Owner Transfer:** Physical hardware reset procedures allow seamless ownership transfer at garage sales, estate sales, or secondary markets without vendor account unlocking or original owner authorization.
6. **Zero Subscription Prerequisites:** Core hardware utility shall never be gated behind recurring fees, cloud software tiers, or vendor accounts.

---

## The Core Idea: Manifest, Values, Controls

A USB device carries a small, static descriptor that tells the computer what it is, and the computer's generic driver does the rest. PLUGS works the same way over wireless and serial links.

```text
Firmware   = what is allowed         (the device is always the final authority)
Manifest   = what exists             (static, sent once, never holds live readings)
Values     = what the numbers are now (small, changing, keyed by id)
Controls   = what the user wants     (one change at a time, validated by the device)
Viewer     = how it looks            (any developer's creation)
```

Take an e-bike. The Manifest describes each item once:

```json
{ "id": "speed", "label": "Speed", "type": "digital_readout", "unit": "km/h" }
```

After that, all the device ever sends is the numbers:

```json
{"speed": 18.4, "battery_level": 76, "distance": 12.3}
```

The device isn't smart, and the phone is. Today's Bluetooth devices already work this way, with one difference: the meaning of each byte is hardcoded inside a different app for every product. PLUGS puts that meaning in the Manifest, so one generic Viewer can understand them all.

**For Viewer developers:** the Manifest is raw material. It gives you each item's type, unit, range, role, and order. What you do with them is yours. A speed reading can become a dial, a battery percentage can become an icon with a color, a thermostat setpoint can become a slider. A small stream of data can be rendered beautifully by a creative Viewer.

---

## Supply Chain and Manufacturing Advantages

PLUGS lowers engineering, bill-of-materials (BOM), and regulatory barriers for small hardware businesses, independent makers, and overseas ODMs:

* **Ship a Bluetooth Chip Instead of a Remote, a Receiver, and an App:** An LED light strip, a fan, or a TV can drop the plastic remote, the IR or RF receiver, and the custom phone app. A cheap Bluetooth chip, a Manifest, and a small amount of firmware let any PLUGS Viewer control the whole device: power, volume, channels, inputs, brightness, colors. A lost remote stops being a problem, and a manufacturer who wants to stop shipping one can do so once Viewers are common. A product can ship both at first. (A single physical control, such as a pairing/reset button or the act of powering the device up, remains for security.)
* **BOM & Assembly Reduction via Headless Architecture:** Manufacturers eliminate cheap LCDs, 7-segment screens, and fragile rubber navigation buttons. This removes physical chassis cutouts, acrylic display windows, waterproofing gaskets, and pin-heavy button harnesses—slashing component costs, tooling expenses, and manual assembly labor while eliminating the primary physical hardware failure points (water/dust ingress, cracked screens, and worn-out buttons).
* **Zero AWS and Cloud Server Liabilities:** Manufacturers burn an open PLUGS firmware module onto an inexpensive ESP32 chip ($1.50) and ship the product with zero ongoing cloud hosting or backend server overhead.
* **No Mobile App Maintenance:** Eliminates the continuous software debt of designing, publishing, and maintaining custom iOS and Android apps across breaking OS updates.
* **Streamlined Import Compliance:** Air-gapped local appliances enter import markets purely as standalone local equipment (similar to a digital multimeter or microwave), minimizing network security liabilities and cross-border data privacy overhead.
* **Onboard Documentation & Zero Web Dependencies:** Manufacturers can store vector PDF user manuals, wiring schematics, and offline troubleshooting guides directly on the device's cheap onboard flash memory (e.g., 8MB/16MB SPIFFS/LittleFS) on devices that serve files over local Wi-Fi. This eliminates broken 404 links, website redesign casualties, and customer support overhead when product documentation vanishes from corporate web servers over time.

---

## Target Ecosystems and Market Opportunities

Modern cloud-dependent smart hardware fails hardest where internet access is non-existent, unstable, or where hardware operational lifespan must outlast corporate startups. PLUGS expands outward from immediate off-grid champions into massive global hardware markets.

### Early Adopter Champions (Immediate Off-Grid Value)
These sectors represent industries where off-grid local reliability is mandatory, cloud dependencies are actively rejected, and hardware must survive for decades:

1. **Marine Equipment & Boating:** Navigation telemetry, bilge pump monitors, lithium battery banks, and engine diagnostics operating miles offshore with zero cell service.
2. **RVs & Off-Grid Solar:** Solar charge controllers, inverter/chargers, freshwater/graywater tank sensors, and battery management systems (BMS) in mobile and off-grid builds.
3. **Agriculture & Water Management:** Automated irrigation valves, well pump controllers, soil moisture telemetry, and livestock climate systems deployed across rural farmland.
4. **Commercial HVAC & Refrigeration:** Walk-in freezers, commercial draft systems, and HVAC dampers where local diagnostics reduce downtime and simplify maintenance.
5. **Field Operations & Critical Infrastructure:** Utility crew diagnostic tools, disaster recovery gear, and emergency communications equipment operating in grid-down scenarios.
6. **Amateur (Ham) Radio & Field Operations:** Remote antenna rotators, weather stations, and field power switches used in off-grid emergency communication deployments.

### Mass Market Applications (High-Volume Expansion)
Where PLUGS eliminates massive software liabilities, app maintenance costs, and cloud hosting overhead for high-volume manufacturers:

7. **Universal Battery Systems (BMS):** Tool batteries, solar storage banks, marine lithium packs, and e-bike batteries sharing universal telemetry primitives (SOC, Voltage, Current, Temperature, Cell Balance, Cycle Count, and Faults).
8. **Automotive & Transportation:** Auxiliary battery monitors, trailer brake controllers, tire pressure (TPMS) gateways, winches, recovery gear, and OBD diagnostic monitors.
9. **Power Tools & Shop Equipment:** Battery chargers, portable welders, air compressors, job-site generators, and dust collection systems.
10. **Recreation & Micromobility:** E-bikes, e-scooters, electric skateboards, golf carts, and ATV telemetry.
11. **Consumer Electronics & Precision Tools:** TVs, LED lighting, DSLR cameras, field audio recorders, telescopes, 3D printers, and network UPS systems.

---

## Architecture and Data Flow

```text
1. HARDWARE LAYER (ESP32 / nRF52 / Microcontroller)
   Firmware holds the Manifest and the current readings,
   and validates every control change.
   |
   +--> Local BLE GATT / Serial / Wi-Fi
   |
2. TRANSPORT LAYER (Zero Cloud Hops)
   Three operations, each using the transport's own verbs:
   read Manifest, read Values (poll, or push where offered), write one Control.
   |
   +--> Direct Local Transmission
   |
3. CLIENT / RENDER LAYER (Phone, PC, or Local Hub)
   A PLUGS Viewer builds the interface from the Manifest
   and keeps it live from Values.
   (Optional: device-served HTML opens separately in the host browser.)
```

### Phone and PC Client Software Layer

When engineers and hardware OEMs read the specification, seeing clean, generic primitives (like toggle, slider, percentage_gauge, and status_indicator) proves that the standard is genuinely domain-agnostic. The exact same protocol engine runs on an e-bike, a smart deadbolt, or a commercial HVAC damper without needing special-casing or custom drivers. PLUGS eliminates proprietary per-device apps through open rendering choices:

* **Zero-Install WebBluetooth (Browser Native):** The browser connects directly over BLE, reads the Manifest, and renders controls on the fly.  
  * *Best for:* Instant, zero-friction access on phones and PCs without installing anything or joining a Wi-Fi network.
  * *Platform note:* WebBluetooth availability varies across browser engines and host operating systems. Viewers must not assume it and should offer local Wi-Fi or USB as a fallback.

* **Viewer-Generated Interface:** A Viewer, native or web, may draw the Manifest however it likes, including by generating a page of its own. Because the Manifest is plain data, creative Viewers can all render the same device differently, and the device doesn't change.

* **Device-Served HTML/JS (Optional Presentation Asset):** A device with Wi-Fi can host a single-file HTML5 page in flash memory for a branded layout, custom charts, or service diagrams.  
  * *Best for:* A manufacturer's own look, offline access in any browser, including where browser BLE APIs are unavailable.
  * *How it works:* The page opens as an ordinary page in the host's standard browser, on the device's own local address. It is never run inside the PLUGS Viewer, so the Viewer's sandbox rules stay intact.
  * *Rules:* The HTML is never the only way to reach a reading or control, it is never authoritative, and if it ever disagrees with the Manifest, the Manifest wins. The device's firmware validates every command regardless of who sent it.
 
* **Host-Side Value-Add Extensions:** PLUGS natively supports host-side application layers, enabling manufacturers to build custom value-add features like contact-named passcodes, calendar scheduling, or analytics on phone and desktop software. This extension architecture gives companies full freedom for commercial software differentiation while ensuring the physical device remains 100% accessible through open, unconstrained local endpoints.
  * *Best for:* Proprietary software features, advanced local management, and commercial product differentiation.

* **Aggregator Integration:** Home Assistant is the committed integration target (Roadmap Phase 4), creating control tiles from the Manifest automatically. Other platforms such as Apple Home, Google Home, or Alexa do not ingest PLUGS natively. They could be reached through bridges, with a lossy mapping to their fixed device types; unmapped items fall back to generic entities.  
  * *Best for:* Multi-device home automation, background scheduling, and unified dashboard control.

PLUGS does not require any particular platform vendor to provide a Viewer. The standard defines the interfaces, and anyone, including a platform vendor, may build a Viewer on top of them.

---

### Schema Architectural Preface: The Death of Rigid Device Classes

Traditional IoT standards (such as Matter, Zigbee, and legacy Bluetooth profiles) force hardware into rigid, pre-defined "device classes" like Door Locks, Thermostats, or Moisture Meters. When a manufacturer creates hybrid hardware or introduces novel features, these rigid committee categories break, forcing vendors to write proprietary clusters or custom mobile apps. 

#### The 2010 Memory Bottleneck
This rigid taxonomy was originally an engineering necessity. In 2010, Bluetooth Low Energy microcontrollers operated under extreme physical constraints, often limited to 8 kilobytes of RAM and 20-byte wireless packet caps. Because these tiny chips could not store or transmit human-readable text schemas, manufacturers hardcoded numeric byte arrays (e.g., `0x01` = locked) and were forced to place the translation logic (the UI "Rosetta Stone") on proprietary cloud servers or single-purpose phone apps.

#### The Web-Like PLUGS Paradigm
Today, inexpensive $1.50 microcontrollers (such as the ESP32) feature megabytes of flash storage, hundreds of kilobytes of RAM, and high-speed BLE pipelines. For this class of part, the memory bottleneck that forced cloud dependence no longer exists. For cheaper parts below it, PLUGS defines a minimal conformance profile: a compact CBOR Manifest of a few hundred bytes (typically 300-600 bytes for a handful of controls and readings, versus 1-2 KB as JSON).

PLUGS abandons rigid device taxonomies entirely, adopting the dynamic design model of the World Wide Web:
* **Web Browser Analogy:** Just as a web browser does not need prior knowledge of whether a site is a bank or a blog to render its buttons and text, a PLUGS viewer does not need prior knowledge of a device type to render its control panel.
* **Atomic UI Primitives:** Devices simply broadcast their available capabilities using open JSON, YAML, or CBOR arrays containing basic UI primitives (toggles, sliders, status indicators, gauges, and text inputs).
* **Instant Hybrid Compatibility:** Whether hardware is a single-purpose moisture sensor or a complex hybrid generator with integrated BMS and deadbolt controls, the device self-describes its physical feature set. Generic PLUGS viewers render the interface on the fly without requiring custom drivers, vendor apps, or schema negotiation.
* **The Manifest Is the Authority on Description:** The Manifest answers "what is this device?" for viewers, hubs, and automations. Anything richer (device-served HTML, PDF manuals, schematics) is optional and only supplements it. A battery monitor with nothing but a tiny Manifest is a complete PLUGS device.
* **Optional Semantic Roles and Units:** A Manifest may add a `role` hint (such as `battery_level` or `lock_state`) and a `unit` (such as `km/h`) from small shared registries. They help aggregators map entities and let Viewers localize, for example showing mph or km/h from the same firmware. Roles are optional; viewers render by primitive type, and unknown roles are simply ignored.
* **Layout by Order:** Items appear in the order the manufacturer lists them. PLUGS v1.0 defines no layout language.

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
      "role": "lock_state",
      "options": ["LOCKED", "UNLOCKED"],
      "confirm": "physical",
      "confirm_on": ["UNLOCKED"]
    }
  ],
  "telemetry": [
    {
      "id": "battery_level",
      "label": "Battery",
      "type": "percentage",
      "role": "battery_level"
    },
    {
      "id": "tamper_status",
      "label": "Tamper Sensor",
      "type": "status_indicator",
      "role": "tamper_state"
    }
  ],
  "security": {
    "pairing_method": "PHYSICAL_BUTTON_PRESS",
    "air_gapped": true
  }
}
```

The lock's Values, sent separately, are just:

```json
{"lock_state": "LOCKED", "battery_level": 88, "tamper_status": "NORMAL"}
```

The `confirm: "physical"` declaration tells every viewer, hub, and voice assistant that unlocking needs a physical action on the lock. The lock's firmware refuses to unlock without it, whoever asks.

---

## Security Architecture and Air-Gap Standards

Physical USB ports expose host machines to electrical destruction, BadUSB keystroke injection, and Direct Memory Access (DMA) exploits. PLUGS delivers the plug-and-play simplicity of USB with superior security insulation:

* **Zero Copper Contact:** All proximity communication occurs over sandboxed wireless airwaves (BLE or local Wi-Fi). High-voltage spikes from malicious hardware cannot travel across radio waves to damage a phone or computer motherboard.
* **BadUSB and DMA Immunity:** PLUGS devices never mount as Human Interface Devices (keyboards/mice) or system controllers. The host OS parses plain, sandboxed text payloads (JSON/YAML/CBOR), preventing hardware-level command injection.
* **No Device Code in the Viewer:** The PLUGS Viewer never executes scripts or code sent by a device, and it treats every device-supplied string as plain text, never as markup. Optional device-served HTML runs only in the host's ordinary browser, outside the Viewer.
* **Firmware Is the Final Authority:** A device checks every control change itself: unknown items, wrong types, and out-of-range values are rejected, whoever sent them.
* **Physical Intent Gatekeeping:** Administrative and security-critical functions (pairing, unlocking, safety overrides, factory reset) require an explicit physical action on the device (a button press or chassis toggle). The device firmware enforces this, and the Manifest declares it so that viewers and hubs never attempt silent execution. Simple low-risk devices with no button, such as an LED strip, may use a short pairing window that opens only when power is physically applied.

---

## Commercial and Industry Resilience

In commercial real estate, industrial facilities, and fleet management, cloud dependencies create massive financial and operational risks. Commercial hardware—such as HVAC dampers, backup generators, sub-meters, and access gates—is engineered to last 15 to 30 years, whereas cloud startups and SaaS platforms rarely survive a decade.

* **Immunity to SaaS Sunsetting & Price Hikes:** If a commercial vendor goes bankrupt, shuts down its cloud infrastructure, or imposes predatory subscription hikes, installed hardware continues operating locally without requiring a costly "rip-and-replace" project.
* **Zero-Downtime Platform Migration:** Property managers and fleet operators can transition an entire deployment of PLUGS sensors to a new software dashboard, local server, or open-source hub instantly—without dispatching technicians to re-key or swap physical devices.
* **Air-Gapped Operational Continuity:** Multi-tenant access systems, utility sub-metering, and critical machinery monitoring remain 100% operational during internet service provider outages, cloud server downtime, or WAN network failures.

---

### Legacy Hardware Retrofits

PLUGS bridges legacy and proprietary hardware through two distinct retrofit mechanisms depending on where the compatibility gap exists. In both, the Profile that describes the legacy device, with the translation needed to read its values and send its controls, lives on the Viewer, and after translation the Viewer treats it like any other PLUGS device:

1. **Device-Side Hardware Bridges (For Wired / Dumb Machines):** Low-cost ($2 ESP32) inline hardware adapters attached directly to non-wireless or wired legacy equipment (via UART, RS-485, screw terminals, or USB). In v1.0 the bridge is a transparent pipe that carries the device's raw stream to the Viewer over BLE or Wi-Fi. (A later revision may let a bridge present a Profile itself.)

2. **Host-Side Software Shims (For Proprietary Wireless Devices):** Pure software translation engines running locally on the user's phone, PC, or local hub. The client software uses the host's native wireless radio to snoop and translate proprietary BLE payloads into clean PLUGS schemas in real time with zero extra hardware.

*Note: For host PCs or local servers lacking native Bluetooth hardware, a standard off-the-shelf USB Bluetooth dongle provides the necessary wireless link.*

Profiles can be shared through a **Community Registry**. The standard defines how a Viewer asks for one, by a hash of the device's protocol layout, and a Viewer only asks with the owner's consent. The reference registry is a public GitHub repository, but any registry, including a local or LAN copy, works.

---

## Repository Layout

```text
.
├── README.md              <-- Specification Architecture Overview
├── SPECIFICATION.md       <-- Complete technical protocol specification
├── LICENSE                <-- MIT Open Source License
├── schemas/               <-- Reference JSON/YAML device definition schemas
│   ├── sample_a.json
│   ├── sample_b.json
│   ├── sample_c.json
│   ├── roles.json         <-- Versioned semantic role registry (to be created)
│   └── units.json         <-- Unit registry (to be created)
├── viewers/               <-- WebBluetooth PWA reference implementation
└── firmware/              <-- ESP32/Arduino C++ helper libraries
```

---

## Development Roadmap

* **Phase 1: Protocol and Schema Specification** (Core protocol and schema definition freeze)
* **Phase 2: Universal WebBluetooth Reference PWA** (Browser-based PWA rendering engine, with Wi-Fi/USB fallback where WebBluetooth is unavailable)
* **Phase 3: Firmware Helper Libraries** (ESP32, C++, and MicroPython reference code)
* **Phase 4: Home Assistant Integration Module** (Native component for open-source home automation)

---

Stay Grounded with PLUGS. Get out of the Cloud.

## Contributing and License

We welcome contributions from embedded software engineers, hardware designers, right-to-repair champions, and cybersecurity researchers.

* **Full Specification:** See [`SPECIFICATION.md`](./SPECIFICATION.md) for full protocol definitions.
* **License:** PLUGS is released as an open standard under the **MIT License**.
