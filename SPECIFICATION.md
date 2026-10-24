# PLUGS Technical Protocol Specification (v1.0)

## 1. Overview & Architectural Principles
* **1.1 Scope:** Transport-agnostic local self-description protocol for embedded microcontrollers and hardware bridges.
* **1.2 Normative References:** BLE GATT/ATT Specification, RFC 8949 (CBOR), IEEE 802.11 Local HTTP, USB CDC-ACM / UART Serial Standards.
* **1.3 Terminology:** `Device`, `Client Viewer`, `Hardware Bridge`, `Schema Profile`, `Primitive`.

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
  *Devices communicating over direct USB CDC-ACM or UART serial lines may implement the lightweight, zero-BOM `?PLUGS\n` line protocol. When queried, the device streams its onboard static schema manifest directly to the host or bridge. For full message sequencing, baud rate conventions, and headless profile requirements, see **Appendix A: Manufacturer 'PLUGS-Friendly' Serial Protocol Specification**.

---

## 3. Schema Syntax & Primitive Specification
* **3.1 Root Schema Object:** Required metadata fields (`plugs_version`, `device`, `controls`, `telemetry`, `security`).
* **3.2 Control Primitives:** `toggle`, `slider`, `button`, `dropdown_select`.
* **3.3 Telemetry Primitives:** `percentage_gauge`, `digital_readout`, `status_indicator`, `waveform_stream`.
* **3.4 Documentation & Asset Metadata:** Onboard PDF/SVG links (`/manual.pdf`, `/schematic.svg`).

---

## 4. Legacy Protocol Synthesis & Hardware Bridge Engine

### 4.1 Bridge Hardware Architecture
Legacy appliances lacking native wireless or open PLUGS firmware participate in the ecosystem via a **Device-Side Hardware Bridge** (e.g., \$2 ESP32-S2/S3 with native USB Host / UART / RS-485 interfaces). The bridge serves as physical pipe host, stream sampler, and profile persistence node.

```text
+-----------------------+     USB / Serial      +--------------------------+     BLE / Wi-Fi     +------------------------+
| Legacy Hardware Tool  |  ------------------>  | $2 ESP32 Hardware Bridge |  ---------------->  | Phone / PC Client App  |
| (e.g., Geiger Counter)|  Raw CDC / ASCII Text | (Hosts USB Pipe & Flash) |   PLUGS GATT / Web  | (Snoop & AI Engine)    |
+-----------------------+                       +--------------------------+                     +------------------------+
```

---

### 4.2 Multi-Tier Hardware Fingerprinting
When an unknown legacy device is attached to a bridge, the bridge computes a **Composite Fingerprint Signature** to identify the hardware without executing complex code:

```text
Fingerprint Signature = SHA256(USB_VID:PID || Product_Descriptor_String || Header_Stream_Prefix)
```

#### Tier 1: Global & Local Schema Registry Lookup
Before triggering local synthesis, the client app or bridge checks for a pre-built profile match using the composite signature:

1. **Local Bridge Cache:** Query onboard flash (`/profiles/fp_[hash].json`).
2. **Client App Local DB:** Query client-side offline database.
3. **Open GitHub Registry:** If internet access is available, query `https://raw.githubusercontent.com/plugs-standard/registry/main/schemas/`.

If a match is found, the validated PLUGS JSON schema is downloaded and written directly to the bridge’s onboard flash memory for instant 1-click configuration.

---

### 4.3 Interactive Differential Snooping & User Prompting
If the device is unindexed, the client app initiates an interactive 3-phase snoop wizard to map telemetry and controls:

#### Phase A: Passive Background Capture (Telemetry Isolation)
* The device sits idle for 5–10 seconds.
* The bridge captures a 1KB raw buffer of repeating background data (e.g., periodic serial telemetry updates).
* The client engine isolates static delimiters (`CPM:`, `temp=`, `\r\n`) from dynamic numerical values.

#### Phase B: Interactive Stimulus (Differential Control Discovery)
* The client app prompts the user: *"Press or toggle a physical control on the device (e.g., Mute or Power)."*
* The user presses the button.
* The bridge captures the burst payload and performs a byte-level diff against the passive baseline buffer:

```text
Delta_Payload = Burst_Buffer - Passive_Baseline
```

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

#### Profile Flashing & Community Feedback Loop
1. **Flash Upload:** Client app sends `profile_gmc300.json` back to the hardware bridge over BLE.
2. **Flash Persistence:** The bridge saves the JSON payload to onboard LittleFS flash mapped to `USB_VID:PID`.
3. **Community Contribution:** The client viewer prompts the user: *"Would you like to submit this synthesized schema to the open GitHub registry to auto-enable this device for all PLUGS users?"*

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
