# PLUGS Technical Protocol Specification (v1.0)

## 1. Overview & Architectural Principles
* **1.1 Scope:** Transport-agnostic local self-description protocol for embedded microcontrollers and hardware bridges.
* **1.2 Normative References:** BLE GATT/ATT Specification, RFC 8949 (CBOR), IEEE 802.11 Local HTTP, USB CDC-ACM / UART Serial Standards.

### 1.3 Architectural Vocabulary

To ensure complete precision across system components, specification documents, and client software implementations, PLUGS establishes eleven canonical nouns:

1. **PLUGS Device:** A device that self-describes through an onboard PLUGS Manifest streamed directly over local transport.
2. **Legacy Device:** A non-PLUGS device whose raw protocol stream is exposed via bridges (hardware or software) and interpreted via Profiles on the PLUGS Viewer located on the Host.
3. **Hardware Bridge:** A physical intermediary that connects to a wired or otherwise physically inaccessible Legacy Device and exposes its raw protocol stream to the PLUGS Viewer through a supported Transport.
4. **Software Bridge:** A software intermediary running on the Host platform that connects to a wireless Legacy Device and exposes its raw protocol stream to the PLUGS Viewer without requiring additional physical hardware.
5. **Host:** The computing platform (such as Android, iOS, Windows, Linux, or Home Assistant) running the PLUGS Viewer engine.
6. **PLUGS Viewer:** The application engine running on a Host that performs four core duties: discovery, stream interpretation, dynamic UI rendering, and user interaction.
7. **Manifest:** An authoritative, self-describing PLUGS document embedded directly in onboard flash memory on a PLUGS Device.
8. **Profile:** An external schema overlay applied to a Legacy Device, stored exclusively on the Host and managed by the PLUGS Viewer.
9. **Device Signature:** A deterministic 12-character hexadecimal hash (`sig_[12-hex]`) generated from observed protocol behavior (such as GATT layout or USB VID:PID stream headers) to uniquely identify legacy or white-labeled hardware.
10. **Transport:** The physical or wireless communication channel (such as BLE, local Wi-Fi, USB, Serial, or RS-485) used to exchange streams, manifests, profiles, telemetry, and control payloads.
11. **Primitive:** A fundamental, domain-agnostic UI or data element (such as `toggle`, `slider`, `digital_readout`, `percentage_gauge`, or `status_indicator`) rendered dynamically by a PLUGS Viewer.

---

## 2. Transport & Communication Layers
* **2.1 BLE GATT Service & Characteristic Architecture:**
  * Primary Service UUID (`0xPLUG` / 128-bit custom UUID).
  * `Schema_Manifest_Char` (Read/Indicate): Returns raw or compressed JSON/CBOR UI schema.
  * `State_Telemetry_Char` (Notify): Streaming binary/JSON telemetry payloads.
  * `Control_Command_Char` (Write Without Response): Inbound control execution.
* **2.2 Local Wi-Fi & Static HTTP Serving:**
  * Embedded single-file HTML5/JS payload serving via onboard flash SPIFFS/LittleFS.
### 2.3 Physical Serial / USB CDC-ACM Standards
Devices communicating over direct USB CDC-ACM or UART serial lines may implement the lightweight, zero-BOM `?PLUGS\n` line protocol. When queried, the device streams its onboard static schema manifest directly to the host or bridge. For full message sequencing, baud rate conventions, and headless profile requirements, see **Appendix A: Manufacturer 'PLUGS-Friendly' Serial Protocol Specification**.

---

## 3. Schema Syntax & Primitive Specification
* **3.1 Root Schema Object:** Required metadata fields (`plugs_version`, `device`, `controls`, `telemetry`, `security`).
* **3.2 Control Primitives:** `toggle`, `slider`, `button`, `dropdown_select`.
* **3.3 Telemetry Primitives:** `percentage_gauge`, `digital_readout`, `status_indicator`, `waveform_stream`.
* **3.4 Documentation & Asset Metadata:** Onboard PDF/SVG links (`/manual.pdf`, `/schematic.svg`).

### 3.5 Class-Free Primitives vs. Rigid Device Taxonomies

Legacy smart home standards force hardware into rigid, predefined category boxes (such as `thermostat`, `smart_lock`, or `light_bulb`). 

PLUGS intentionally rejects rigid device taxonomies in favor of a **Primitive-First UI Architecture**.

#### The HTML & HID Principle
Just as USB HID provided generic input building blocks (buttons, axes, sliders) and HTML provided universal web elements (`<button>`, `<input type="range">`, `<span>`, `<progress>`), PLUGS provides domain-agnostic telemetry and control primitives.

PLUGS applies this self-describing philosophy to physical hardware:

* **Generic Primitives First:** Client viewers render interactive interfaces using core primitives (`toggle`, `slider`, `percentage_gauge`, `digital_readout`, `status_indicator`).
* **Unbounded Engineering:** An engineer building a custom solar pool heater with four temperature sensors, a chemical injector, and a pump flow monitor can describe their hardware completely without breaking client viewers or waiting for a standards committee to invent a "pool heater" category.
* **Optional Semantic Roles:** To assist smart home aggregators (such as Home Assistant) in automated entity mapping, schemas may include optional semantic hints (e.g., `unit: "%"`, `role: "battery_level"`). However, client viewers render controls based strictly on the primitive type.

---

## 4. Discovery, Legacy Protocol Synthesis & Hardware Bridge Engine

To achieve universal utility on day one, the PLUGS architecture seamlessly supports natively compliant hardware while providing a reliable bridge for legacy, orphaned, and white-labeled equipment.

---

### 4.1 Bridge Hardware Architecture

Legacy appliances lacking native wireless or open PLUGS firmware participate in the ecosystem via a **Device-Side Hardware Bridge** (e.g., \$2 ESP32-S2/S3 with native USB Host / UART / RS-485 interfaces). The bridge serves as physical pipe host, stream sampler, and profile persistence node.

```text
+-----------------------+ USB / Serial +--------------------------+ BLE / Wi-Fi +------------------------+
| Legacy Hardware Tool  | ------------------> | $2 ESP32 Hardware Bridge | ----------------> | Phone / PC Client App  |
| (e.g., Geiger Counter)| Raw CDC / ASCII Text| (Hosts USB Pipe & Flash) | PLUGS GATT / Web  | (Snoop & AI Engine)    |
+-----------------------+                      +--------------------------+                     +------------------------+
```

---

### 4.2 Deterministic Device Signatures & Multi-Tier Trust Pipeline

When an unknown or legacy device connects, client software generates a **Deterministic Device Signature** (`sig_[12-hex]`) by hashing observed protocol behavior rather than brand strings. This allows PLUGS to instantly recognize badge-engineered and white-labeled hardware sold under dozens of different brand names worldwide.

#### Signature Hash Generation
* **Bluetooth Low Energy (BLE):**
  \\(\text{Device Signature} = \text{SHA-256}(\text{Primary Service UUIDs} + \text{GATT Characteristic Layout} + \text{Manufacturer Data Header})[:12]\\)
* **Serial / USB Streams:**
  \\(\text{Device Signature} = \text{SHA-256}(\text{USB VID:PID} + \text{Baud Rate} + \text{First 64 Bytes of Raw Stream})[:12]\\)

#### The 4-Tier Trust Pipeline

To maintain transparent confidence levels, client software categorizes incoming streams across four distinct trust tiers:

```text
[ Incoming Device Stream ]
           │
           ├── Tier 1: Native PLUGS Manifest (100% Onboard Flash Schema) ────────► [ Gold Standard / Verified ]
           │
           ├── Tier 2: Vendor-Supported Legacy Profile (Verified Shim) ──────────► [ High Confidence ]
           │
           ├── Tier 3: Community-Derived Profile (Signature Hash Match) ─────────► [ Best-Effort / Community ]
           │
           └── Tier 4: Unknown Device Stream (Unindexed Payload) ─────────────────► [ Snoop & Map Mode ]
```

1. **Tier 1 — Native PLUGS Devices (Gold Standard):**
   * *Mechanism:* The device responds to the Tier 0 `?PLUGS\n` query or broadcasts its schema directly from internal flash memory (`plugs_version: "1.0"`). 100% self-describing and authority-verified.
2. **Tier 2 — Vendor-Supported Legacy Profiles:**
   * *Mechanism:* Maintained directly by original manufacturers in public registries to support non-PLUGS legacy hardware via shims.
3. **Tier 3 — Community-Derived Profiles (Signature Hash Match):**
   * *Mechanism:* Matched via Device Signature (`sig_[12-hex]`) against the open community registry (`plugs-standard/registry`). Rendered with a clear "Community Profile" trust badge.
4. **Tier 4 — Unknown Device Stream (Unindexed Payload):**
   * *Mechanism:* Unrecognized serial or GATT stream. Triggers the interactive **Snoop Wizard** (Section 4.3) and **AI Synthesis Engine** (Section 4.4).

---

### 4.3 Interactive Differential Snooping & User Prompting

If the device is an unindexed Tier 4 stream, the client app initiates an interactive 3-phase snoop wizard to map telemetry and controls:

#### Phase A: Passive Background Capture (Telemetry Isolation)
* The device sits idle for 5–10 seconds.
* The bridge captures a 1KB raw buffer of repeating background data (e.g., periodic serial telemetry updates).
* The client engine isolates static delimiters (`CPM:`, `temp=`, `\r\n`) from dynamic numerical values.

#### Phase B: Interactive Stimulus (Differential Control Discovery)
* The client app prompts the user: *"Press or toggle a physical control on the device (e.g., Mute or Power)."*
* The user presses the button.
* The bridge captures the burst payload and performs a byte-level diff against the passive baseline buffer:
  \\(\text{Delta Payload} = \text{Burst Buffer} - \text{Passive Baseline}\\)
* The user tags the action in the UI (e.g., label = `"Audio Clicker Mute"`). The isolated command bytes are mapped to a `toggle` or `button` primitive.

#### Phase C: User Context Injection
To maximize schema precision, the user optionally inputs basic brand/model metadata (e.g., `"GMC-300 Geiger Counter"`).

---

### 4.4 AI Schema Synthesis & Onboard Persistence

The client app feeds the raw payload buffer, differential byte map, and user context into a local LLM or parser prompt engine.

#### Prompt Engineering Synthesis Matrix
```text
SYSTEM: You are the PLUGS 1.0 Protocol Schema Synthesizer.
CONTEXT: Device Model = "GMC-300 Geiger Counter"
RAW STREAM BUFFER: "CPM: 32, uSv/h: 0.14, Bat: 88%\r\n"
DIFFERENTIAL MAP: Input "Mute" triggers output payload 0x4D 0x55 0x54 0x45
TASK: Synthesize a valid PLUGS 1.0 JSON schema mapping fields to UI primitives.
```

#### Output: Synthesized Legacy Schema (`profile_gmc300.json`)
```json
{
  "plugs_version": "1.0",
  "device": {
    "name": "GMC-300 Geiger Counter",
    "type": "radiation_sensor",
    "legacy_retrofit": true,
    "bridge_fingerprint": "FTDI_0403_6001_GMC300"
  },
  "controls": [
    {
      "id": "audio_mute",
      "label": "Clicker Audio",
      "type": "toggle",
      "payload_on": "0x4D555445",
      "payload_off": "0x554E4D55",
      "current": "ON"
    }
  ],
  "telemetry": [
    {
      "id": "cpm_count",
      "label": "Counts Per Minute (CPM)",
      "type": "digital_readout",
      "unit": "CPM",
      "value": 32
    },
    {
      "id": "radiation_dosage",
      "label": "Radiation Level",
      "type": "percentage_gauge",
      "unit": "uSv/h",
      "min": 0.0,
      "max": 10.0,
      "value": 0.14
    },
    {
      "id": "battery_level",
      "label": "Battery",
      "type": "percentage",
      "unit": "%",
      "value": 88
    }
  ],
  "security": {
    "pairing_method": "HARDWARE_BRIDGE_PASS_THROUGH",
    "air_gapped": true
  }
}
```

#### Profile Flashing & Local Persistence
1. **Flash Upload:** Client app sends `profile_gmc300.json` back to the hardware bridge over BLE or serial.
2. **Flash Persistence:** The bridge saves the JSON payload to onboard LittleFS flash mapped to the device signature.
3. **Instant Re-Connection:** Subsequent connections automatically load the local schema in under 200ms.

---

### 4.5 The Open GitHub Registry & Community Flywheel (`plugs-standard/registry`)

Community-derived profiles (Tier 3) are indexed in an open, version-controlled GitHub repository (`plugs-standard/registry`):

* **Lookup Resolution Order:** `Local Bridge Flash` $\rightarrow$ `Client App Offline Database` $\rightarrow$ `GitHub Raw CDN Lookup` (`https://raw.githubusercontent.com/plugs-standard/registry/main/signatures/sig_a3f912bc8e01.json`).
* **Crowdsourced Schema Enrichment:** When a tinkerer uncovers an unmapped telemetry field (e.g., discovering Cell 4 Balance Voltage on a generic battery monitor) and submits a GitHub Pull Request, every PLUGS user owning a device matching that signature hash automatically gains access to the updated metric.

---

## 5. Security, Pairing, and Air-Gap Standards
* **5.1 Zero-Trust Input Parsing:** Strict JSON/CBOR schema validation preventing buffer overflow attacks.
* **5.2 Physical Proximity Pairing:** Chassis button confirmation requirements for administrative keys.
* **5.3 BadUSB / DMA Insulation:** Wireless air-gap protections against electrical or host-level USB exploits.

---

## 6. Client Viewer Engine Mechanics
* **6.1 Local Favorites Database:** Storing local MAC/UUID connection pointers and cached schemas.
* **6.2 Offline Error State Management:** Clear physical troubleshooting messaging (e.g., *"Device out of range or off"* vs cloud timeouts).

---

## 7. Compliance & Testing Guidelines
* **7.1 Schema Validation Suite:** Automated JSON-Schema verification scripts.
* **7.2 Hardware Bridge Interoperability Benchmarks.**

## Appendix A: Manufacturer 'PLUGS-Friendly' Serial Protocol Specification

### A.1 Overview & Physical Pipe Parameters
The PLUGS-Friendly Serial Protocol enables low-cost microcontrollers lacking wireless radios (such as microcontrollers in digital multimeters, solar charge controllers, or Geiger counters) to declare their telemetry metrics and controls over a simple wired UART or USB CDC-ACM connection.

* **Baud Rate:** 115200 bps (default) or autobaud.
* **Data Format:** 8 Data Bits, No Parity, 1 Stop Bit (8N1).
* **Line Framing:** Standard newline ASCII character (`\n` or `\r\n`).
* **Electrical Interfaces:** USB CDC-ACM, 3.3V/5V TTL UART, or RS-485.

---

### A.2 The 5-Byte Discovery Handshake (`?PLUGS\n`)

When an ESP32 hardware bridge or client PC connects to a serial device, it initiates discovery by sending a 5-byte ASCII query.

```text
Host Query:       ?PLUGS\n
Device Response:  [Full JSON Schema Manifest]\n
```

#### Protocol Execution Steps:
1. **Query Phase:** Host sends ASCII string `?PLUGS\n`.
2. **Device Detection:** Microcontroller recognizes `?PLUGS\n` in its serial RX interrupt buffer.
3. **Payload Stream:** Microcontroller streams its stored JSON schema manifest directly out of internal flash memory over the TX line.
4. **Timeout & Fallback:** If no response is received within 200ms, the host assumes the device is a legacy or non-compliant tool and initiates Tier 1 (Database Lookup) or Tier 2 (Differential Snoop & AI Synthesis).

---

### A.3 Headless and Displayless Device Profile Flags

For hardware engineered without physical LCD screens or rubber navigation buttons, the schema manifest includes the `"headless"` boolean flag in the device metadata block.

#### Schema Manifest Example with Headless Flag:
```json
{
  "plugs_version": "1.0",
  "device": {
    "name": "Compact Solar Inverter",
    "type": "power_inverter",
    "headless": true,
    "physical_display": false,
    "physical_buttons": false
  },
  "telemetry": [
    {
      "id": "pv_voltage",
      "label": "Solar Array Input",
      "type": "digital_readout",
      "unit": "V",
      "value": 48.2
    },
    {
      "id": "ac_output_power",
      "label": "AC Load Power",
      "type": "digital_readout",
      "unit": "W",
      "value": 350
    }
  ],
  "controls": [
    {
      "id": "inverter_output",
      "label": "AC Output Power",
      "type": "toggle",
      "current": "ON"
    }
  ],
  "documentation": {
    "user_manual": "/docs/manual.pdf",
    "wiring_diagram": "/docs/schematic.svg"
  }
}
```

#### Client Viewer Behavior for Headless Devices:
When `"headless": true` is present:
* The client viewer automatically prioritizes interactive touch controls on the user's phone or PC screen.
* Full diagnostic alarms and plain-text error descriptions are rendered automatically, replacing cryptic LED blinking codes or 7-segment error numbers.

---

### A.4 Onboard Documentation Serving
Devices with onboard flash storage (such as SPIFFS, LittleFS, or external SPI flash chips) declare local file paths in the `"documentation"` schema block. 

Client viewers can request these files over serial or local wireless links, ensuring user manuals and wiring schematics remain accessible off-grid without internet connectivity.
