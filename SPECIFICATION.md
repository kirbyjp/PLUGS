# PLUGS Technical Protocol Specification (v1.0)

> **Status:** Draft, pre-freeze. Items marked **TODO** are known gaps. Section 7.3 lists them.
>
> **Key words:** MUST, MUST NOT, SHALL, SHALL NOT, SHOULD, SHOULD NOT, and MAY are used as described in RFC 2119 and RFC 8174.

## 1. Overview & Architectural Principles
* **1.1 Scope:** Transport-agnostic local self-description protocol for embedded microcontrollers and hardware bridges.
* **1.2 Normative References:** BLE GATT/ATT Specification, RFC 8949 (CBOR), RFC 4648 (Base64), RFC 2119/8174 (requirement keywords), IEEE 802.11 Local HTTP, USB CDC-ACM / UART Serial Standards.

### 1.3 Architectural Vocabulary

To ensure complete precision across system components, specification documents, and client software implementations, PLUGS establishes thirteen canonical nouns. No other nouns SHALL be introduced as normative terms. Descriptive phrases (for example "presentation asset" or "Community Registry") refer to these terms and do not replace them.

1. **PLUGS Device:** A device that carries an onboard, self-describing PLUGS Manifest within its data stream.
2. **Legacy Device:** A non-PLUGS device whose raw stream lacks an onboard Manifest and requires a Profile for interpretation. This stream is exposed via bridges (hardware or software) and interpreted via Profiles by the PLUGS Viewer located on the Host.
3. **PLUGS Hardware Bridge (PHB):** A physical adapter converting wired/RS-485 media, or any other participating hardware manufacturer (even a wired but compliant PLUGS 1.0 device) into a transport pipe for the PSB. In v1.0 a PHB is a transparent pipe: it holds no Profile and no authoritative data.
4. **PLUGS Software Bridge (PSB):** A Host-side software component that acquires device streams and presents them to the PLUGS Viewer through a single ingress path. Considered part of the Transport.
5. **Host:** The computing platform (such as Android, iOS, Windows, Linux, or Home Assistant) running the PLUGS Viewer engine.
6. **PLUGS Viewer:** The application engine running on a Host that performs four core duties: discovery, stream interpretation from Bridges, dynamic UI rendering, and user interaction.
7. **Manifest:** An authoritative, self-describing PLUGS document embedded directly in onboard flash memory on a PLUGS Device. It describes structure only (what items exist, their types, labels, units, ranges, and requirements) and never contains live values. Transported by a Bridge.
8. **Profile:** The Manifest-equivalent description of a Legacy Device, plus the translation data needed to read values from, and write controls to, the device's own protocol. A Profile is authored and managed by the PLUGS Viewer and stored on the Host. After translation, the Viewer treats it exactly like a Manifest.
9. **Profile Discovery:** The process by which the Viewer identifies, retrieves, synthesizes, or selects a candidate Profile for a stream lacking an onboard Manifest.
10. **Device Signature:** A deterministic 12-character hexadecimal hash (`sig_[12-hex]`) generated from observed protocol behavior and used as a Profile Discovery hint, helping the Viewer locate, synthesize, or select the most appropriate Profile for a Legacy Device.
11. **Transport:** The physical or wireless communication channel (such as BLE, local Wi-Fi, USB, Serial, or RS-485) used to present and exchange streams, manifests, profiles, values, and control payloads via the onboard software bridge or a connected hardware bridge.
12. **Primitive:** A fundamental, domain-agnostic UI or data element (such as `toggle`, `slider`, `digital_readout`, `percentage_gauge`, or `status_indicator`) rendered dynamically by a PLUGS Viewer.
13. **Values:** The current readings and settings of the items a Manifest (or Profile) describes, keyed by item `id`. A snapshot is all of them, and an update is some of them (Section 3.10).

### 1.4 The Three Operations

Everything a Viewer does with a native PLUGS Device is one of three operations. Each Transport carries them with its own native mechanism (Section 2) and PLUGS defines no additional verbs.

| Operation | Purpose | Direction |
| :--- | :--- | :--- |
| **Read Manifest** | What exists | Device to Viewer |
| **Read Values** | What the values are right now (poll, or optionally pushed) | Device to Viewer |
| **Write Control** | Change one setting | Viewer to Device |

---

## 2. Transport & Communication Layers

### 2.1 BLE GATT Service & Characteristic Architecture
* **Primary Service UUID:** a 128-bit custom UUID. **TODO:** generate once, publish here, and derive the three characteristic UUIDs from the same base. (A 16-bit UUID can only be assigned by the Bluetooth SIG, and `0xPLUG` is not valid hexadecimal.)
* `Schema_Manifest_Char` (Indicate): Manifest transfer (Section 2.1.1).
* `Values_Char` (Read, Notify): Values snapshot and optional updates (Section 2.1.2).
* `Control_Command_Char` (Write, with response): Control writes (Section 2.1.3).

#### 2.1.1 Manifest Transfer Framing (Chunking & MTU)

A single ATT attribute value cannot exceed 512 bytes, and at the default ATT MTU of 23 only 20 bytes fit in one notification or indication. Manifests are therefore transferred as a sequence of frames.

1. **Trigger:** The Viewer enables indications on `Schema_Manifest_Char`. The device then sends frames in order, one indication at a time, and waits for the ATT confirmation of each before sending the next.
2. **Frame layout:** `[flags: 1 byte][seq: 1 byte][data: n bytes]`
   * `flags` bit 0 = START, bit 1 = END. A single-frame manifest sets both.
   * `seq` starts at 0 on the START frame and increments by 1 (modulo 256) per frame.
   * The START frame's data begins with `[encoding: 1 byte][total_len: 4 bytes, big-endian]`, followed by the first manifest bytes. `encoding` is `0x01` for JSON (UTF-8, minified) or `0x02` for CBOR (raw binary, no Base64 on BLE).
3. **Frame size:** `n` MUST NOT exceed `ATT_MTU - 3 - 2`. Devices MUST work at the default MTU of 23 (18 data bytes per frame). Devices and Viewers SHOULD negotiate a larger MTU where the platform allows it.
4. **Validation:** The Viewer MUST verify sequence continuity and that the reassembled length equals `total_len`. On any violation the Viewer MUST discard the transfer and MAY retry by re-enabling indications.
5. **Encoding rule:** The encoding declared here (JSON or CBOR) is the encoding the device uses for Values and expects for Control writes on this connection.

#### 2.1.2 Values on BLE

1. **Snapshot:** A Read of `Values_Char` MUST return the full Values snapshot (Section 3.10). It is read with the standard ATT long-read mechanism and MUST fit in 512 bytes. A device whose snapshot would be larger uses Wi-Fi.
2. **Notify (optional):** If the characteristic has the Notify property, the device MAY send updates. Each notification is a self-contained, valid partial Values object holding only the ids that changed. A device MAY split a large update across several notifications.
3. **Polling-only devices:** A device MAY omit Notify. A Viewer MUST work with such a device by polling, and the polling interval is the Viewer's choice.
4. **Viewer sequence:** A Viewer that enables Notify SHOULD still read a snapshot first, so it never waits for a change to learn a value.

#### 2.1.3 Control Writes on BLE

1. A Write to `Control_Command_Char` carries exactly one control object (Section 3.11) in the connection's encoding. Writes up to 512 bytes use the standard ATT long-write mechanism.
2. The write is made *with* response. On acceptance the device returns the normal ATT success response. On rejection it returns an ATT application error code, using the codes in Section 3.11, rule 3.
3. Acceptance means the device applied the change. The Viewer still shows the confirmed setting from Values (Section 6.5).

### 2.2 Local Wi-Fi & HTTP

PLUGS over Wi-Fi uses plain HTTP verbs. Request and response bodies use JSON, or CBOR when the request says `Accept: application/cbor` (for reads) or `Content-Type: application/cbor` (for writes).

| Operation | Request | Success | Failure |
| :--- | :--- | :--- | :--- |
| Read Manifest | `GET /plugs/manifest.json` or `GET /plugs/manifest.cbor` | `200` with the Manifest | n/a |
| Read Values | `GET /plugs/values` | `200` with the full snapshot | n/a |
| Write Control | `POST /plugs/control` with one control object | `200` | status per Section 3.11, rule 3, body `{"err":"<code>"}` |

1. **Polling only:** v1.0 defines no push on Wi-Fi. Viewers poll `GET /plugs/values`.
2. **Presentation assets:** A device MAY additionally serve files (such as a single-file HTML5 page, PDFs, SVGs) from onboard flash (SPIFFS/LittleFS). These are presentation assets under Section 3.4, are served over HTTP only, and are never required.
3. **Parity requirement:** A device that serves HTML MUST also implement the three endpoints above, so that no reading or control is reachable only through HTML.
4. **Transport security:** Wi-Fi security is the network's WPA2/WPA3, per Section 5.4.
5. **TODO:** device address discovery (for example an mDNS service name, or manual entry), Wi-Fi credential provisioning, and client authorization beyond network membership are not specified in v1.0.

### 2.3 Physical Serial / USB CDC-ACM Standards
Devices communicating over direct USB CDC-ACM or UART serial lines may implement the lightweight, zero-BOM line protocol. Serial is request/response (polling) in v1.0: the host sends one line and the device answers with one line.

| Operation | Host sends | Device answers |
| :--- | :--- | :--- |
| Read Manifest | `?PLUGS\n` | the Manifest, one line |
| Read Values | `?VALUES\n` | the full Values snapshot, one line |
| Write Control | one control object, one line | `{"ok":true}` or `{"err":"<code>"}`, one line |

Line encoding follows Appendix A.2: JSON is minified to a single line, and CBOR is Base64-encoded. For full sequencing, baud rate conventions, framing, and headless profile requirements, see **Appendix A**.

---

## 3. Schema Syntax & Primitive Specification

### 3.0 The Core Rule

> **The Manifest describes structure. Values carry readings and settings. Firmware is the authority.**

1. The Manifest is static. It never contains a live reading or a current setting, so it is never rewritten while the device runs.
2. Values are dynamic and are keyed by the `id` of the item they belong to.
3. Whatever a client, a hub, or a web page asks for, device firmware decides what is allowed (Section 5.7).

### 3.1 Root Schema Object
Required fields: `plugs_version`, `device`, `controls`, `telemetry`, `security`.
* `controls` and `telemetry` MUST be present and MAY be empty arrays.
* `security` MUST contain `pairing_method` and `air_gapped`.
* `documentation` is OPTIONAL (Section 3.4).
* `manifest_version` is OPTIONAL informational metadata (Section 3.9).

### 3.2 Control Primitives
`toggle`, `slider`, `button`, `dropdown_select`, `text_input`.

### 3.3 Telemetry Primitives
`percentage`, `percentage_gauge`, `digital_readout`, `status_indicator`, `waveform_stream`.

* `percentage` is a value from 0 to 100, always presented as a percent.
* `percentage_gauge` is a ranged gauge. It carries `min` and `max` and MAY represent any quantity (for example radiation level in uSv/h). The name reflects its gauge presentation, not its unit.
* `status_indicator` reports text in the device's own words. A device that wishes to report a fault does so with a `status_indicator` (see the `fault` role, Section 3.6). PLUGS defines no separate diagnostics mechanism in v1.0.

#### 3.3.1 Common Fields

| Field | Applies to | Requirement |
| :--- | :--- | :--- |
| `id` | all | MUST. Unique within the Manifest; lowercase letters, digits, and underscore. |
| `label` | all | MUST. Human-readable name. |
| `type` | all | MUST. One of the Primitives above. |
| `unit` | `digital_readout`, `percentage_gauge`, `slider`, `waveform_stream` | MUST where the quantity has a unit; omit otherwise. A string from the unit registry (Section 3.7). |
| `min`, `max` | `slider`, `percentage_gauge` | MUST. |
| `options` | `toggle` (exactly two), `dropdown_select` | MUST. |
| `max_length` | `text_input` | MUST. |
| `role` | all | SHOULD (Section 3.6). |
| `confirm`, `confirm_on` | controls | `confirm` MUST for Section 5.3 high-risk controls (Section 3.8). |
| `description` | all | MAY. Plain text. |

No item carries `value` or `current`. A Viewer that encounters an unknown `type` MUST NOT guess its meaning (Section 6.4).

### 3.4 Documentation, Presentation Assets & the Authority Rule

The Manifest is mandatory and authoritative. Everything else a device carries is an optional presentation asset.

1. **Authority Rule:** Every PLUGS Device MUST carry a Manifest (and every Legacy Device MUST be given a Profile) that defines its Primitives, ids, labels, units, ranges, and any `role` and `confirm` declarations. A Viewer can render and operate the device from this document and Values alone.
2. **Presentation assets:** Manuals, schematics, and device-served HTML are declared in the OPTIONAL `documentation` block. They exist for manufacturers who want a branded layout, custom charts, service diagrams, or manuals. They are never required for operation.
3. **Parity Rule:** A presentation asset MUST NOT be the only path to any reading or control. Everything an asset offers MUST also be present in the Manifest and available through the three operations.
4. **Conflict Rule:** If an asset and the Manifest disagree, the Manifest wins.
5. **Convenience only:** Device-served HTML is a convenience interface only. It is never authoritative. Device firmware remains the ultimate enforcement point for all control limits, permissions, and physical confirmation requirements.
6. **HTTP only:** In v1.0, assets are served over HTTP only (Section 2.2). A BLE-only device has no assets, and a Manifest alone is a complete, conformant device.
7. **No execution inside the Viewer:** The Viewer never executes device-supplied HTML/JS (Section 5.2, Section 6.7).

```json
"documentation": {
  "user_manual": "/docs/manual.pdf",
  "wiring_diagram": "/docs/schematic.svg",
  "assets": [
    { "path": "/ui/index.html", "media_type": "text/html", "description": "Browser dashboard" }
  ]
}
```

`user_manual` and `wiring_diagram` are paths. `assets` is a list of additional files, each with `path`, `media_type`, and an optional `description`.

### 3.5 Class-Free Primitives vs. Rigid Device Taxonomies

Legacy smart home standards force hardware into rigid, predefined category boxes (such as `thermostat`, `smart_lock`, or `light_bulb`).

PLUGS intentionally rejects rigid device taxonomies in favor of a **Primitive-First UI Architecture**.

#### The HTML & HID Principle
Just as USB HID provided generic input building blocks (buttons, axes, sliders) and HTML provided universal web elements (`<button>`, `<input type="range">`, `<span>`, `<progress>`), PLUGS provides domain-agnostic telemetry and control primitives. The analogy goes further: a USB device carries a small, static descriptor that says what it has, and a generic host driver moves the changing data. The Manifest is that descriptor, Values are the data reports, and the Viewer is the generic driver.

PLUGS applies this self-describing philosophy to physical hardware:

* **Generic Primitives First:** Client viewers render interactive interfaces using core primitives (`toggle`, `slider`, `percentage_gauge`, `digital_readout`, `status_indicator`).
* **Unbounded Engineering:** An engineer building a custom solar pool heater with four temperature sensors, a chemical injector, and a pump flow monitor can describe their hardware completely without breaking client viewers or waiting for a standards committee to invent a "pool heater" category.
* **Layout by Order:** Viewers present items in the order they appear in the Manifest arrays. A manufacturer controls layout simply by how it writes the list. PLUGS v1.0 defines no layout or grouping fields.
* **Optional Semantic Roles:** To assist smart home aggregators (such as Home Assistant) and creative Viewers, Manifests MAY include optional semantic hints (Section 3.6). Client viewers render controls based strictly on the primitive type.

### 3.6 Semantic Roles (SHOULD)

1. **Optional hint:** `role` is a SHOULD, not a MUST. A missing role never prevents rendering or operation.
2. **Single vocabulary:** Roles come from one small, versioned registry (`schemas/roles.json`, **TODO:** to be created). Initial entries: `battery_level`, `lock_state`, `tamper_state`, `power_switch`, `temperature`, `voltage`, `current`, `power`, `fault`. Implementations MUST NOT invent private role names in a Manifest that claims `plugs_version: "1.0"`.
3. **The `fault` role:** A `status_indicator` with `role: "fault"` reports a human-readable fault description in the device's own words. By convention the value `NONE` means no active fault. A Viewer MAY emphasize such an item. A device that has no fault text to share simply omits it.
4. **Unknown roles:** A Viewer or aggregator that does not recognize a role MUST ignore it and render the item from its Primitive type as a generic entity.
5. **Roles never override safety:** A `role` never removes or weakens a `confirm` declaration (Section 3.8).
6. **Synthesized profiles:** `role` values proposed by the AI Synthesis Engine (Section 4.4) are suggestions. They MUST be confirmed by the user, and values outside the registry MUST be discarded.

### 3.7 Units (Registry)

1. **Declared once:** An item declares its unit in the Manifest, and the device always reports Values in that unit.
2. **Viewer converts:** A Viewer MAY convert for display, for example `mph` to `km/h`, or `degF` to `degC`, so one firmware serves every country. The device never needs a unit-switching control.
3. **Registry:** Unit strings come from a small registry (`schemas/units.json`, **TODO:** to be created). Initial entries: `V`, `A`, `W`, `kW`, `Wh`, `kWh`, `Ah`, `%`, `degC`, `degF`, `Hz`, `rpm`, `m`, `km`, `mi`, `m/s`, `km/h`, `mph`, `L`, `gal`, `Pa`, `kPa`, `psi`, `bar`, `s`, `min`, `h`, `CPM`, `uSv/h`.
4. **Unknown units:** A Viewer that does not recognize a unit string MUST display the value with the unit string as given and MUST NOT convert it.

### 3.8 Physical Confirmation Declaration (`confirm`)

Section 5.3 requires physical human intent for high-risk actions. Because a role such as `lock_state` could otherwise let an aggregator or voice engine offer "unlock" as an ordinary command, the Manifest declares the requirement explicitly.

1. **Declaration:** A control whose state change belongs to a Section 5.3 high-risk class (door unlocking, safety limit overrides, factory reset, and so on) MUST declare `"confirm": "physical"`.
2. **Scope:** For a control with named `options`, an optional `"confirm_on": [...]` list restricts the requirement to transitions into the listed option values. Without it, every change requires physical confirmation.
3. **Dual-sided enforcement:** Declaration and enforcement are separate duties.
   * Device firmware MUST reject the action unless the physical chassis event has occurred, regardless of which client or channel (Viewer, aggregator, voice engine, device-served HTML) sent it (Section 5.3, Section 5.7).
   * Viewers and aggregators MUST read the declaration, MUST present the physical-interaction prompt, and MUST NOT attempt silent execution.
4. A declaration is not a substitute for firmware enforcement, and firmware enforcement does not excuse omitting the declaration.

### 3.9 Manifest Encoding & Size

1. **Encodings:** JSON, YAML (JSON-compatible subset recommended for constrained devices), or CBOR (RFC 8949). v1.0 uses the same string keys in every encoding. A compact integer-key mapping is reserved for a future revision.
2. **Fetched, not synchronized:** A Viewer requests the Manifest on every connection. A Viewer MAY keep a copy for its favorites list or offline display, but a cached Manifest is never authoritative, and PLUGS defines no cache-validation protocol.
3. **`manifest_version`:** OPTIONAL string or integer for debugging and display. It has no protocol meaning.
4. **Informative sizing:** A small Manifest with several controls and readings is typically 1-2 KB as minified JSON and roughly 300-600 bytes as CBOR. A trivial single-reading device can be smaller, but should not be assumed to fit in 200 bytes once labels and units are included.
5. **Minimal conformance profile:** A device with very little flash is conformant if its Manifest contains the Section 3.1 required fields, only Section 3.3.1 MUST fields, no assets, and uses CBOR. Modern ESP32-class parts have ample flash and RAM for fuller Manifests; the minimal profile exists for much cheaper parts.
6. **Viewer limits:** A Viewer MUST accept Manifests up to 64 KiB and MAY reject larger ones.

### 3.10 Values

Values are the current readings and settings of the items a Manifest describes. A Values payload is a JSON object (or CBOR map with string keys) keyed by item `id`.

1. **Snapshot:** A full Values payload that includes every item that has a current reading or setting.
2. **Update:** A partial Values payload that includes only the ids that changed.
3. **Units:** Numeric values use the unit the Manifest declares for that item (Section 3.7).
4. **Unavailable:** A `null` value, or an id missing from a snapshot, means the value is currently unavailable. The Viewer shows it as unavailable.
5. **Unknown ids:** A Viewer MUST ignore ids that are not in the Manifest.
6. **Value types:**

| Primitive | Value | Readable | Writable |
| :--- | :--- | :--- | :--- |
| `toggle`, `dropdown_select` | string, one of `options` | yes | yes |
| `slider` | number from `min` to `max` | yes | yes |
| `text_input` | string, at most `max_length` | yes | yes |
| `button` | not present in Values | no | write `true` |
| `percentage` | number, 0 to 100 | yes | no |
| `percentage_gauge` | number | yes | no |
| `digital_readout` | number or string | yes | no |
| `status_indicator` | string | yes | no |
| `waveform_stream` | array of numbers, oldest first | yes | no |

Example (an e-bike). The Manifest describes the items once:

```json
{ "id": "speed", "label": "Speed", "type": "digital_readout", "unit": "km/h" }
```

and Values carry only the numbers:

```json
{"speed": 18.4, "battery_level": 76, "distance": 12.3}
```

### 3.11 Control Writes

1. **One control per write:** A write is a single-entry object keyed by `id`: `{"volume": "up"}`, `{"output": "OFF"}`, `{"brightness": 60}`, `{"reset_trip": true}`. Writes that carry several ids are reserved for a future revision.
2. **Value types:** As in the table in Section 3.10: an option string for `toggle` and `dropdown_select`, a number for `slider`, a string for `text_input`, and `true` for `button`.
3. **Rejection:** Firmware MUST reject a write that fails validation (Section 5.7). Every rejection carries one of the following codes:

| Code | Meaning | BLE ATT error | HTTP status | Serial reply |
| :--- | :--- | :--- | :--- | :--- |
| `unknown_id` | No such control | `0x80` | `404` | `{"err":"unknown_id"}` |
| `invalid_value` | Wrong type, unknown option, or over-length text | `0x81` | `400` | `{"err":"invalid_value"}` |
| `out_of_range` | Number outside `min`..`max` | `0x82` | `400` | `{"err":"out_of_range"}` |
| `confirm_required` | Physical confirmation not present | `0x83` | `403` | `{"err":"confirm_required"}` |
| `busy` | Device cannot act now | `0x84` | `503` | `{"err":"busy"}` |

BLE codes are in the ATT application-error range (`0x80`-`0x9F`).

---

## 4. Discovery, Legacy Protocol Synthesis & Hardware Bridge Engine

To achieve universal utility on day one, the PLUGS architecture seamlessly supports natively compliant hardware while providing a reliable bridge for legacy, orphaned, and white-labeled equipment.

---

### 4.1 Bridge Hardware Architecture

Legacy appliances lacking native wireless or open PLUGS firmware participate in the ecosystem via a **Device-Side Hardware Bridge** (e.g., \$2 ESP32-S2/S3 with native USB Host / UART / RS-485 interfaces). In v1.0 the bridge is a transparent physical pipe and stream sampler. The Profile lives on the Viewer.

```text
+-----------------------+ USB / Serial +--------------------------+ BLE / Wi-Fi +------------------------+
| Legacy Hardware Tool  | ------------------> | $2 ESP32 Hardware Bridge | ----------------> | Phone / PC Client App  |
| (e.g., Geiger Counter)| Raw CDC / ASCII Text| (Transparent USB pipe)   | Raw stream        | (Profile, Snoop & AI)  |
+-----------------------+                     +--------------------------+                   +------------------------+
```

*Future work:* A later revision may let a hardware bridge present a Profile to Viewers as a Manifest, so that a pre-configured bridge looks like a native PLUGS Device. Doing so will require a way to mark such a Manifest as bridge-synthesized rather than native. v1.0 does not define it.

---

### 4.2 Deterministic Device Signatures & Multi-Tier Trust Pipeline

When an unknown or legacy device connects, client software generates a **Deterministic Device Signature** (`sig_[12-hex]`) by hashing observed protocol behavior rather than brand strings. This allows PLUGS to instantly recognize badge-engineered and white-labeled hardware sold under dozens of different brand names worldwide.

#### Signature Hash Generation

* **Bluetooth Low Energy (BLE):**
  ```text
  Device Signature = SHA-256(Primary Service UUIDs + GATT Characteristic Layout + Manufacturer Data Header)[:12 hex chars]
  ```
  The Manufacturer Data Header SHOULD be limited to the 2-byte company identifier, because the remaining manufacturer bytes commonly carry live values. Service UUIDs and characteristics are sorted before hashing.

* **Serial / USB Streams (text, line-framed):**
  ```text
  Device Signature = SHA-256(VID:PID + "|" + Baud Rate + "|" + Normalized Line Set)[:12 hex chars]
  ```
  The signature MUST NOT depend on live telemetry values. The **Normalized Line Set** is produced as follows:
  1. Capture at least 5 seconds of passive stream (Section 4.3, Phase A) and discard the first, possibly partial, line.
  2. For each complete line, replace every numeric token matching `[+-]?[0-9]+(\.[0-9]+)?([eE][+-]?[0-9]+)?` with `<N>`, and every hexadecimal literal matching `0x[0-9A-Fa-f]+` with `<H>`. Keys, delimiters, units, and whitespace are preserved byte-for-byte, so `CPM: 32, uSv/h: 0.14, Bat: 88%` becomes `CPM: <N>, uSv/h: <N>, Bat: <N>%`.
  3. Collect up to 16 distinct normalized lines, sort them lexicographically (byte order), and join them with a single `\n`.

  Two Viewers observing the same protocol therefore produce the same signature regardless of capture phase or live readings.
  * *Note:* Binary (non-line-framed) serial protocols are not yet covered. **TODO:** define a structure-based signature for binary framing.

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

1. **Tier 1: Native PLUGS Devices (Gold Standard):**
   * *Mechanism:* The device answers the Read Manifest operation (Section 1.4) from internal flash memory (`plugs_version: "1.0"`). 100% self-describing and authority-verified.
2. **Tier 2: Vendor-Supported Legacy Profiles:**
   * *Mechanism:* Maintained directly by original manufacturers in a Community Registry or other public location, to support non-PLUGS legacy hardware via shims.
3. **Tier 3: Community-Derived Profiles (Signature Hash Match):**
   * *Mechanism:* Matched via Device Signature (`sig_[12-hex]`) against a Community Registry (Section 4.5). Rendered with a clear "Community Profile" trust badge.
4. **Tier 4: Unknown Device Stream (Unindexed Payload):**
   * *Mechanism:* Unrecognized serial or GATT stream. Triggers the interactive **Snoop Wizard** (Section 4.3) and **AI Synthesis Engine** (Section 4.4).

---

### 4.3 Interactive Differential Snooping & User Prompting

If the device is an unindexed Tier 4 stream, the client app initiates an interactive 3-phase snoop wizard to map telemetry and controls:

#### Phase A: Passive Background Capture (Telemetry Isolation)
* The device sits idle for 5-10 seconds.
* The bridge captures a 1KB raw buffer of repeating background data (e.g., periodic serial telemetry updates).
* The client engine isolates static delimiters (`CPM:`, `temp=`, `\r\n`) from dynamic numerical values.

#### Phase B: Interactive Stimulus (Differential Control Discovery)
* The client app prompts the user: *"Press or toggle a physical control on the device (e.g., Mute or Power)."*
* The user presses the button.
* The bridge captures the burst payload and performs a byte-level diff against the passive baseline buffer:
  ```text
  Delta Payload = Burst Buffer - Passive Baseline
  ```
* The user tags the action in the UI (e.g., label = `"Audio Clicker Mute"`). The isolated command bytes are mapped to a `toggle` or `button` primitive.

#### Phase C: User Context Injection
To maximize schema precision, the user optionally inputs basic brand/model metadata (e.g., `"GMC-300 Geiger Counter"`).

---

### 4.4 AI Schema Synthesis & Profiles

The client app feeds the raw payload buffer, differential byte map, and user context into a local LLM or parser prompt engine.

1. **User review:** A synthesized Profile MUST be shown to the user for review before it is saved or submitted to a registry.
2. **Validation:** Synthesized output MUST pass schema validation. Roles are handled per Section 3.6, rule 6.
3. **Declarative only:** A Profile is data. The Section 5.2 execution prohibition applies to it unchanged.

#### Profile-Only Translation Fields

A Profile is a Manifest-shaped description plus minimal translation fields. Native Manifests never need them, and a Viewer ignores them in a native Manifest.

* **`read` (telemetry items):** how to extract the item's value from the raw stream.
  * Text lines: `{"pattern": "<regular expression>", "group": 1}`. The captured group is the value. `group` defaults to 1.
  * Binary frames: `{"offset": 4, "length": 2, "format": "u16be", "scale": 0.01}`. Formats: `u8`, `u16be`, `u16le`, `i16be`, `i16le`, `f32be`, `f32le`. `scale` defaults to 1.
  * A Viewer MUST evaluate patterns with a linear-time regular-expression engine, or enforce a match-time limit, so a Profile cannot stall the Viewer.
* **`write` (control items):** the bytes to send.
  * `toggle` and `dropdown_select`: an object mapping each option to a payload, for example `{"MUTED": "0x4D555445", "UNMUTED": "0x554E4D55"}`.
  * `button`: `{"payload": "0x..."}`.
  * `slider`: `{"template": "SET <N>\r\n"}`, where `<N>` is replaced by the value.

These fields are intentionally minimal and extensible in a later revision.

#### Prompt Engineering Synthesis Matrix
```text
SYSTEM: You are the PLUGS 1.0 Protocol Schema Synthesizer.
CONTEXT: Device Model = "GMC-300 Geiger Counter"
RAW STREAM BUFFER: "CPM: 32, uSv/h: 0.14, Bat: 88%\r\n"
DIFFERENTIAL MAP: Input "Mute" triggers output payload 0x4D 0x55 0x54 0x45
TASK: Synthesize a valid PLUGS 1.0 Profile mapping fields to UI primitives.
```

#### Output: Synthesized Legacy Profile (`profile_gmc300.json`)
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
      "label": "Clicker Mute",
      "type": "toggle",
      "options": ["MUTED", "UNMUTED"],
      "write": { "MUTED": "0x4D555445", "UNMUTED": "0x554E4D55" }
    }
  ],
  "telemetry": [
    {
      "id": "cpm_count",
      "label": "Counts Per Minute (CPM)",
      "type": "digital_readout",
      "unit": "CPM",
      "read": { "pattern": "CPM:\\s*([0-9]+)" }
    },
    {
      "id": "radiation_dosage",
      "label": "Radiation Level",
      "type": "percentage_gauge",
      "unit": "uSv/h",
      "min": 0.0,
      "max": 10.0,
      "read": { "pattern": "uSv/h:\\s*([0-9.]+)" }
    },
    {
      "id": "battery_level",
      "label": "Battery",
      "type": "percentage",
      "role": "battery_level",
      "read": { "pattern": "Bat:\\s*([0-9]+)%" }
    }
  ],
  "security": {
    "pairing_method": "HARDWARE_BRIDGE_PASS_THROUGH",
    "air_gapped": true
  }
}
```

After translation, the Viewer produces Values for these items exactly as a native device would. Where the legacy stream does not report a setting (here, the mute state), the Viewer shows the last commanded setting marked as unconfirmed.

---

### 4.5 Community Registry

A **Community Registry** is a source of Profiles that a Viewer can query by Device Signature. PLUGS defines the interface, not the host.

1. **Entry:** A registry entry is one Profile document, named by its Device Signature (`sig_[12-hex].json`).
2. **Lookup order:** Host offline database, then each Community Registry source the user has configured. A source may be a local mirror, a LAN server, or a public server.
3. **Opt-in:** A Viewer MUST function with no registry configured. A Viewer MUST NOT contact a registry without the user's consent, because a lookup discloses a Device Signature (Section 5.6).
4. **Validation:** A Viewer MUST validate every registry Profile against the schema before use and MUST treat it as data only (Section 5.2). A registry Profile is Tier 3 unless it is marked vendor-supported and the Viewer can verify that.
5. **Enrichment:** How contributions are submitted and reviewed is a matter for each registry and is outside this specification. A registry's contributors can improve an entry (for example, adding an unmapped field), and every Viewer that later fetches that signature gets the update.

*Non-normative note:* The reference implementation is an open, version-controlled repository on GitHub (`plugs-standard/registry`), with Profiles at `signatures/sig_<12-hex>.json`, fetched as raw files and improved through pull requests. Other registry implementations are equally valid.

---

## Section 5: Security, Sandboxing, Local Authentication & Privacy Boundaries

### 5.1 Local-First & Air-Gapped Security Architecture

The PLUGS standard enforces a strict **Local-First Air-Gap Boundary**. All device discovery, value acquisition, manifest parsing, and primitive interactions execute 100% locally over proximity transport channels (such as BLE, local Wi-Fi sockets, USB, or Serial) via the **PLUGS Software Bridge (PSB)**.

1. **Zero-WAN Default:** A PLUGS Device, Hardware Bridge (PHB), or Software Bridge (PSB) SHALL NOT require internet access, cloud server communication, external identity providers, or WAN routing to function.
2. **Elimination of Cloud Attack Surfaces:** By operating strictly over local transport channels without inbound WAN ports or persistent cloud socket connections, PLUGS devices significantly reduce or eliminate entire classes of cloud-mediated vulnerabilities, including central database leaks, remote botnet recruitment, and wide-area network exploits.

---

### 5.2 Declarative Execution Only & Viewer Sandboxing

To protect Host platforms (such as mobile phones, PCs, or local automation hubs) from malicious or malformed device streams, PLUGS enforces strict declarative schema parsing at the **PLUGS Viewer** boundary.

1. **Declarative Execution Only:** Streams acquired by the PSB and handed to the PLUGS Viewer consist strictly of structured data payloads (JSON, YAML, or CBOR) defining UI Primitives (`toggle`, `slider`, `digital_readout`, `percentage_gauge`, `status_indicator`) and their Values.
2. **Strict Code Execution Prohibition:** A PLUGS Viewer SHALL NOT execute remote arbitrary binary code, mobile app scripts, JavaScript widgets, native dynamic libraries, or executable binaries transmitted inside a device stream.
3. **Sandbox Isolation:** The PLUGS Viewer operates inside an isolated process sandbox provided by the Host OS. The architecture is designed to prevent malformed or malicious device streams from obtaining direct access to the Host file system, local network resources, or elevated operating system privileges.
4. **Device-Served HTML Boundary:** Optional device-served HTML/JS (Section 3.4) is not part of the Viewer. It runs, if the user chooses to open it, as an ordinary page in the Host's standard browser on the device's own local origin, entirely outside the Viewer's process sandbox and with no access to Viewer state, credentials, or Profiles. The Viewer SHALL NOT load it in-process (see Section 6.7).
5. **Text, not markup:** Every string from a Manifest, Profile, or Values payload is data to be displayed as plain text (Section 6.6).

---

### 5.3 Physical Proximity & Human Intent Verification

To eliminate unauthorized remote or drive-by state changes over wireless local transport (e.g., an unauthorized connection unlocking a door from outside a building), sensitive state mutations enforce physical human intent verification based on action class:

1. **High-Risk / Security-Critical Actions (MUST):** Sensitive operations, including initial device pairing (for devices with high-risk controls, see rule 6), security key generation, door unlocking, safety limit overrides, and factory resets, MUST require a physical chassis interaction on the target hardware, such as:
   * A physical button press or dual-button pattern hold.
   * A physical switch toggle or mechanical key turn.
   * A short-range physical proximity threshold (e.g., NFC tap or physical RSSI thresholding).
2. **Standard Operational Actions (MAY):** Routine reads and non-critical control adjustments (such as adjusting a thermostat setpoint, changing a TV's volume, or dimming a light) MAY be executed over an established secure local session without requiring physical chassis confirmation.
3. **Firmware enforcement:** Device firmware MUST reject any high-risk action that lacks the physical event, regardless of the client, channel, or semantic role that issued it (Section 3.8, Section 5.7).
4. **Declaration:** Every control covering a high-risk action MUST declare `confirm: "physical"` in the Manifest so that Viewers and aggregators present the prompt and never attempt silent execution.
5. **Confirmation window:** A physical event for a `confirm: "physical"` control arms a confirmation window of 5 to 30 seconds. Until the event occurs, the device answers the write with `confirm_required` (Section 3.11, rule 3). The Viewer prompts the user and retries. Once the user acts, the retried write is accepted, and the window is consumed by one accepted action.
6. **Power-up pairing window (low-risk devices):** A device that has no `confirm: "physical"` control and no other Section 5.3 high-risk action MAY satisfy physical intent for pairing with a short pairing window that opens only when power is physically applied (plugging in, inserting a battery, or operating a power switch).
   * The window SHOULD be no longer than 60 seconds and closes after a successful pairing or on timeout.
   * It MUST NOT be reopened by any remote command or by a software-initiated restart.
   * A device with any high-risk control MUST instead require a chassis button, switch, key, or proximity event for pairing.
   * *Note:* Bonding in the window typically uses the Transport's own mechanism (Section 5.4). "Just Works" BLE bonding gives no protection against a man in the middle, so the shortness of the window and the physical power-up are the protection. Devices SHOULD show the window state when they have any indicator.

---

### 5.4 Transport-Decoupled Authentication & Session Security

PLUGS sits directly above the transport layer and leverages the native security mechanisms provided by the underlying Transport (such as BLE GATT security, Wi-Fi WPA2/WPA3, or USB OS security) and the Host operating system.

1. **Transport-Native Security Consumption:** The PSB receives secure, encrypted streams directly from the Host OS transport stack. PLUGS does not define or duplicate low-level cryptographic handshakes or transport encryption protocols.
2. **Zero Cloud Authentication:** A PLUGS Device or Stream SHALL NOT require cloud accounts, vendor OAuth tokens, subscription credentials, or external identity providers to authorize local control.
3. **No Vendor Key Escrow:** Neither manufacturers nor registry operators possess master recovery keys, backdoors, or administrative overrides for paired devices. Cryptographic session tokens reside exclusively within the local Host OS credential store and the physical device flash.

---

### 5.5 Local Ownership Transfer & Credential Reset

To support the natural lifecycle of hardware (including secondhand sales, estate sales, property transfers, and off-grid equipment reuse), hardware ownership is tied strictly to physical possession.

1. **Local Credential Reset:** A PLUGS Device or System SHALL provide a documented, purely physical mechanism on the hardware chassis to wipe stored pairing credentials and restore factory defaults. This may be a hardware reset button pattern, a jumper, or, for devices with no button, a documented power-cycle sequence (for example, applying power a stated number of times within a stated interval).
2. **Vendor-Independent Transfer:** Ownership transfer and credential resets MUST execute 100% locally on the device chassis without requiring manufacturer authorization, cloud server access, or vendor software tools.

---

### 5.6 Privacy Boundaries & Telemetry Protection

PLUGS is designed to maximize user telemetry privacy by decoupling hardware operation from vendor-controlled cloud infrastructure.

1. **Zero Unsolicited Telemetry Exfiltration:** Values, usage patterns, sensor readings, and operational logs remain strictly confined to the local PSB and PLUGS Viewer runtime unless explicitly forwarded by a user-configured domestic integration.
2. **Anonymized Discovery Signatures:** Device Signatures (`sig_[12-hex]`) generated during Profile Discovery derive solely from observed protocol layouts (such as GATT service tables or normalized USB stream structure). They SHALL NOT contain personally identifiable information (PII), geographic location data, or network identities.
3. **Owner-Controlled Forwarding:** Any bridging of local PLUGS values into a remote dashboard, cloud aggregator, or third-party monitoring system is an explicit, owner-initiated choice configured on home soil.
4. **Opt-in registry lookups:** A Community Registry lookup discloses a Device Signature and therefore happens only with the user's consent (Section 4.5).

---

### 5.7 Firmware Authority & Input Validation

Device firmware is the final enforcement point. The Viewer, a presentation asset, an aggregator, or a voice engine is never trusted to enforce limits.

1. For every control write, firmware MUST reject: an unknown `id`; a value of the wrong type; an option not in `options`; a number outside `min`..`max`; and text longer than `max_length`. This applies to every sender and every channel.
2. Firmware MUST apply Section 5.3 confirmation requirements to every sender and every channel.
3. A device MUST NOT rely on a Manifest's ranges being checked by anything other than itself. The ranges in the Manifest are the device's real limits, not hints.
4. Rejections use the codes in Section 3.11, rule 3.

---

## 6. Client Viewer Engine Mechanics

This section defines how a PLUGS Viewer behaves, given the schema and operations above.

* **6.1 Local Favorites Database:** Storing local MAC/UUID connection pointers, a convenience copy of the last Manifest, and Host-held Profiles.
* **6.2 Offline Error State Management:** Clear physical troubleshooting messaging (e.g., *"Device out of range or off"* vs cloud timeouts).

### 6.3 Connection Sequence
1. On every connection to a native PLUGS Device, the Viewer reads the Manifest (Section 1.4). It never trusts a cached copy over a fresh one.
2. The Viewer then reads a Values snapshot.
3. If the device offers push (for example BLE Notify), the Viewer MAY subscribe. Otherwise it polls at an interval of its choosing.
4. A Viewer MUST work with a device that supports only the three operations as defined, with no push.

### 6.4 Rendering
1. A Viewer MUST render and operate a device from its Manifest (or Profile) and Values alone.
2. Items appear in Manifest array order.
3. A Viewer MUST NOT guess the behavior of an unknown Primitive `type`. It MUST show the item as unsupported, labeled with its `label`, and MUST NOT send commands to it.
4. Unknown or missing `role` values are ignored; the item renders as its Primitive type (Section 3.6). Unknown optional fields are ignored.
5. Units are converted for display only where the unit is in the registry (Section 3.7). Unknown units display as given.
6. An unavailable value is shown as unavailable (Section 3.10).
7. How a Viewer draws the result is its own business. It may build a native interface, or generate a page of its own from the Manifest, provided Section 6.6 holds. Creative rendering of the same Manifest is expected and welcome.

### 6.5 Controls
1. The Viewer writes one control object per change (Section 3.11).
2. The Viewer displays the setting the device reports in Values, not the setting it last commanded, and shows an unconfirmed indication until Values catches up. (For a Legacy Device whose stream does not report a setting, the last commanded setting is shown as unconfirmed.)
3. On rejection, the Viewer shows a message that matches the code (Section 3.11, rule 3).
4. On `confirm_required`, the Viewer MUST tell the user that a physical action on the device is required, and retries once the user has acted (Section 5.3, rule 5). It MUST NOT attempt silent execution.
5. A Viewer MUST NOT infer that a control is exempt from physical confirmation because it lacks `confirm`. For Legacy Device Profiles, where firmware enforcement cannot be assumed, see the open item in Section 7.3.

### 6.6 Text Safety
A Viewer MUST treat every string it receives from a device, Profile, or registry (labels, descriptions, option names, status text, text values) as plain text. It MUST NOT interpret any of it as markup or script, and it MUST escape or otherwise neutralize it wherever it builds a page or interface from it.

### 6.7 Presentation Asset Handling (No In-Viewer Execution)
1. A Viewer MUST NOT execute, or render in-process, any device-supplied HTML or JavaScript. In particular it MUST NOT load such content into a WebView or other rendering context that it controls.
2. A Viewer MAY offer the user a link to a presentation asset. Opening it hands the URL (or file) to the Host's standard browser or document viewer, as an ordinary page on the device's own origin (Section 5.2, rule 4). The Viewer SHOULD show the user the address before opening it.
3. PDF manuals SHOULD be handed to the Host's PDF viewer. SVG files can contain scripts and external references, so a Viewer MUST either hand them off or render them only after stripping scripts and external references.
4. Device-served pages hold no elevated authority. Section 5.7 firmware validation applies to commands from them exactly as to commands from a Viewer.
5. The Viewer never treats an asset as a source of truth for values or structure. If an asset disagrees with the Manifest, the Manifest wins (Section 3.4).

### 6.8 Transport Availability and Fallback
1. WebBluetooth availability varies across browser engines and Host operating systems. Viewers MUST NOT assume availability and SHOULD offer local Wi-Fi or USB fallback transports.
2. PLUGS does not require any particular Host platform to provide a Viewer. The standard defines the interfaces, and anyone may build a Viewer on any platform, as a browser page, a native app, or a hub integration.

### 6.9 Aggregator Mapping (Informative)
1. Aggregators can map a Manifest to their own entity models using `role` where present and Primitive `type` otherwise. Home Assistant is the committed integration target (Roadmap Phase 4).
2. Other platforms (for example Apple Home, Google Home, Alexa) do not natively ingest PLUGS and would require a bridge. Mapping to their fixed device types is lossy, and unmapped items fall back to generic entities.
3. Any aggregator MUST honor `confirm` declarations (Section 3.8).

---

## 7. Compliance & Testing Guidelines

### 7.1 Schema Validation Suite
Automated JSON-Schema verification scripts, including checks that: required fields are present; `id` values are unique; no Manifest item contains `value` or `current`; every `percentage_gauge` and `slider` has `min` and `max`; every `toggle` has exactly two `options`; every `text_input` has `max_length`; every Section 5.3 high-risk control has `confirm: "physical"`; Primitive names match Sections 3.2 and 3.3 exactly.

### 7.2 Hardware and Transport Interoperability Benchmarks
Includes:
* BLE manifest transfer at the default MTU of 23 and at larger MTUs.
* Serial framing tests with CBOR payloads containing `0x0A` bytes.
* Values snapshot and update tests, including a polling-only device and a split update.
* Control-write rejection tests: each error code on each Transport.
* Firmware validation tests (Section 5.7), including out-of-range, wrong-type, and over-length writes sent from a non-Viewer client.
* Confirmation-window tests (Section 5.3, rule 5) and power-up pairing window tests (rule 6), including that a software restart does not reopen the window.
* Text-safety tests: Manifests whose strings contain markup and script must render as plain text.
* Device Signature test vectors (the same stream captured at different phases and with different live values MUST yield the same signature).

### 7.3 Open Items
* **Service and characteristic UUIDs** (Section 2.1).
* **Wi-Fi device discovery, credential provisioning, and client authorization beyond network membership** (Section 2.2).
* **Legacy Profiles and `confirm`:** Legacy Devices cannot be assumed to enforce physical intent in firmware. The v1.0 treatment of high-risk controls in Profiles (for example, a legacy door controller) is not yet defined.
* **Binary serial signatures** (Section 4.2).
* **Role registry and unit registry files** (Sections 3.6 and 3.7).
* **Serial file transfer for onboard documentation** (Appendix A.4).
* **Future work, deliberately deferred:** bridge-presented Profiles (Section 4.1), control grouping or layout hints, writes with several ids, and any dedicated diagnostics mechanism.

---

## Appendix A: Manufacturer 'PLUGS-Friendly' Serial Protocol Specification

### A.1 Overview & Physical Pipe Parameters
The PLUGS-Friendly Serial Protocol enables low-cost microcontrollers lacking wireless radios (such as microcontrollers in digital multimeters, solar charge controllers, or Geiger counters) to declare their Manifest and report Values over a simple wired UART or USB CDC-ACM connection.

* **Baud Rate:** 115200 bps (default) or autobaud.
* **Data Format:** 8 Data Bits, No Parity, 1 Stop Bit (8N1).
* **Line Framing:** Standard newline ASCII character (`\n` or `\r\n`).
* **Electrical Interfaces:** USB CDC-ACM, 3.3V/5V TTL UART, or RS-485.

---

### A.2 The Serial Line Protocol

Serial is request/response. The host sends one line and the device answers with exactly one line.

```text
Host:    ?PLUGS\n          Device: [Manifest, one line]\n
Host:    ?VALUES\n         Device: [Values snapshot, one line]\n
Host:    {"volume":"up"}\n Device: {"ok":true}\n   or   {"err":"out_of_range"}\n
```

`?PLUGS\n` is a 7-byte ASCII query (`?`, `P`, `L`, `U`, `G`, `S`, newline).

#### Line Framing (Normative)
Every line the device sends, and every control line the host sends, is **one line** terminated by `\n`, in one of two forms:
* **JSON:** minified, on a single line with no literal newline characters (newlines inside strings MUST be escaped).
* **CBOR:** raw CBOR bytes MUST be Base64-encoded (RFC 4648 standard alphabet, with padding), so binary values such as `0x0A` can never be mistaken for the line delimiter.

A receiver identifies the form by the first non-whitespace byte of the line: `{` means JSON, anything else is treated as Base64 CBOR. (A Base64 string never begins with `{`.) The host uses the same form that the device used for its Manifest. Acknowledgement lines use the same form, with the same map.

#### Execution Steps
1. **Query phase:** Host sends `?PLUGS\n`.
2. **Device detection:** The microcontroller recognizes the query in its serial RX buffer.
3. **Manifest line:** The microcontroller sends its stored Manifest line from internal flash. Native PLUGS serial devices SHOULD NOT send unsolicited lines. A Viewer discards any line it cannot match to its request.
4. **Timeouts:** The Host waits 200 ms for the first byte of a response, allows 200 ms between subsequent bytes, and caps the whole response at 5 seconds.
5. **Fallback:** If no valid Manifest arrives, the host assumes the device is a Legacy Device and moves to Tier 2/Tier 3 Profile lookup (by Device Signature), then to Tier 4 (Differential Snoop & AI Synthesis), per Section 4.2.
6. **Values and control:** After the Manifest, the host polls with `?VALUES\n` at an interval of its choosing and writes controls as single lines. Rejections use the codes in Section 3.11, rule 3.

---

### A.3 Headless and Displayless Device Profile Flags

For hardware engineered without physical LCD screens or rubber navigation buttons, the Manifest includes the `"headless"` boolean flag in the device metadata block.

* `physical_display: false` means no display of any kind.
* `physical_buttons: false` means no user-interface or navigation buttons. It does not remove the Section 5.3 and 5.5 requirement for a physical intent input and reset mechanism (a recessed pairing/reset button, a jumper, or, for low-risk devices, the power-up pairing window and power-cycle reset). A headless device that exposes high-risk actions MUST still provide a physical input for them.

#### Manifest Example with Headless Flag:
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
      "role": "voltage"
    },
    {
      "id": "ac_output_power",
      "label": "AC Load Power",
      "type": "digital_readout",
      "unit": "W",
      "role": "power"
    },
    {
      "id": "inverter_fault",
      "label": "Fault",
      "type": "status_indicator",
      "role": "fault"
    }
  ],
  "controls": [
    {
      "id": "inverter_output",
      "label": "AC Output Power",
      "type": "toggle",
      "options": ["ON", "OFF"],
      "role": "power_switch"
    }
  ],
  "security": {
    "pairing_method": "POWER_UP_WINDOW",
    "air_gapped": true
  },
  "documentation": {
    "user_manual": "/docs/manual.pdf",
    "wiring_diagram": "/docs/schematic.svg"
  }
}
```

Its Values reply to `?VALUES\n`:
```json
{"pv_voltage":48.2,"ac_output_power":350,"inverter_fault":"NONE","inverter_output":"ON"}
```

#### Client Viewer Behavior for Headless Devices:
When `"headless": true` is present:
* The client viewer automatically prioritizes interactive touch controls on the user's phone or PC screen.
* Fault text reported by the device (for example in a `status_indicator` with `role: "fault"`) is displayed as plain text, replacing cryptic LED blinking codes or 7-segment error numbers. The text is whatever the manufacturer chose to report; PLUGS does not translate error codes.

---

### A.4 Onboard Documentation Serving
Devices with onboard flash storage (such as SPIFFS, LittleFS, or external SPI flash chips) declare file paths in the `"documentation"` Manifest block (Section 3.4). Documentation is optional, is served over HTTP only in v1.0 (Section 2.2), and is never required for operation. **TODO:** a command for transferring files over serial is not defined in v1.0, so serial-only devices SHOULD NOT rely on onboard documentation being retrievable.
