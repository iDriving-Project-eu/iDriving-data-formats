# T4.1 — Edge Detection Tools

**Lead partner:** CERTH
**iDriving components:** IC8-IC14

# T4.1 — CERTH Edge Detection Tools

**Lead partner:** CERTH  
**iDriving components:** IC8–IC14

---

## IC8.1 & IC8.2 — Violation detection data format

The example payload is `T4.1-01.json`.

**Tool purpose:** Analyse video or camera streams, detect vehicles and motorbikes, and publish Kafka messages when a safety violation is detected (mobile phone use, missing seatbelt, missing helmet). 
Frame images are stored in MinIO and referenced by URI (claim-check pattern).

**Supported violation types:**

| `condition` | Description | Typical vehicle class |
|-------------|-------------|------------------------|
| `Mobile` | Driver or passenger using a mobile phone | `Vehicle` |
| `Without-Seatbelt` | Person detected without a seatbelt | `Vehicle` |
| `No-Helmet` | Rider or passenger without a helmet | `Motorbike` |

---

## Message envelope

Every message published on the iDriving Kafka message bus has two parts:

1. **Kafka message headers** — transport-level metadata, carried in the Kafka headers (not in the JSON body).
2. **JSON payload** — the business content, which begins with a `records` array. Each record has a `header` block (mirroring the key identifiers) and a `body` block.

### Standard Kafka message headers

| Header key | Example | Purpose |
|------------|---------|---------|
| `project-id` | `iDriving` | Identifies the project; constant across all messages. |
| `use-case-id` | `UC1.1` | Identifies the use-case scope of the message. |
| `message-type` | `idriving_certh_uc1.1_detection` | Identifies the message schema / event type. |
| `producer-id` | `CERTH` | Identifies the producing component instance. |
| `correlation-id` | `2829dc7f-6553-423c-8a2f-1200e969bb1e` | Correlates related messages across components for distributed tracing. |
| `message-timestamp` | `2026-03-02T22:12:41.117082+00:00` | Producer-side UTC timestamp of the message. |
| `content-type` | `application/json` | Declares the payload format. |

### Topic naming

| Purpose | Topic |
|---------|-------|
| Violation detection events (T4.1-01) | `idriving_certh_object_detection_uc1.1` |

---

## Payload structure

Each violation detection message contains one or more records. Every record has two main parts:

- **Header** — message-level metadata used to identify the project, use case, producer, message type, timestamp, and correlation ID.
- **Body** — the business content, including the frame reference and the list of detected violations.

```json
{
  "records": [
    {
      "header": { ... },
      "body": {
        "detection_list": [ ... ]
      }
    }
  ]
}
```

---

## Record header

The `records[].header` block repeats the key identifiers from the Kafka headers.

| Field | Example | Purpose |
|-------|---------|---------|
| `project-id` | `iDriving` | Project identifier. |
| `use-case-id` | `UC1.1` | Use-case scope. |
| `contentType` | `application/json` | Declares the payload format (mirror of Kafka `content-type`). |
| `message-type` | `idriving_certh_uc1.1_detection` | Event type / schema identifier. |
| `producer-id` | `CERTH` | Producing component. |
| `correlation-id` | `2829dc7f-6553-423c-8a2f-1200e969bb1e` | UUID used for distributed tracing. |
| `message-timestamp` | `2026-03-02T22:12:41.117082+00:00` | UTC timestamp when the message was produced (ISO 8601). |

Timestamps should use ISO 8601 format in UTC when possible.

---

## Record body — `detection_list`

The `body.detection_list` array contains one entry per exported frame. Each entry groups all violation detections found in that frame.

```json
"detection_list": [
  {
    "topicName": "idriving_certh_object_detection_uc1.1",
    "frameID": 58,
    "imageURL": "images/violation_nasia_4-mobile_2026.3.3-0:12:35/frame_000058.jpg",
    "detections": [ ... ]
  }
]
```

| Field | Example | Purpose |
|-------|---------|---------|
| `topicName` | `idriving_certh_object_detection_uc1.1` | Kafka topic on which this message is published. |
| `frameID` | `58` | Frame number in the input video or stream. |
| `imageURL` | `images/violation_nasia_4-mobile_2026.3.3-0:12:35/frame_000058.jpg` | Claim-check URI to the annotated frame image in MinIO/S3. |
| `detections` | `[ ... ]` | Array of violation records for this frame. |

### When messages are published

The tool publishes a violation message only when:

1. At least one tracked object has `violation: true`, and
2. The per-track export policy is satisfied:
   - **First detection:** export on the first frame where a violation appears for a given `objectID`
   - **Periodic:** export every N frames (default: 5) while the violation persists for that `objectID`

Non-violating frames are not published.

---

## Detection record — `detections[]`

Each entry in `detections` describes one tracked vehicle or motorbike with a detected violation.

```json
{
  "objectID": 1,
  "class": "Vehicle",
  "vehicle_conf": "92.7%",
  "violation": true,
  "condition": "Mobile",
  "violation_conf": "51.2%",
  "license_plate": "HKY-2821"
}
```

| Field | Example | Purpose |
|-------|---------|---------|
| `objectID` | `1` | Track ID assigned by the multi-object tracker (ByteTrack). Stable identifier for the same vehicle across consecutive frames. |
| `class` | `Vehicle` | Vehicle type. Always the tracked object class, never the violation label. |
| `vehicle_conf` | `92.7%` | Detector confidence for the vehicle or motorbike, expressed as a percentage string. |
| `violation` | `true` | Indicates whether a violation was detected. Always `true` in exported T4.1-01 messages. |
| `condition` | `Mobile` | Type of violation or safety condition (see classification below). |
| `violation_conf` | `51.2%` | Detector confidence for the violation, expressed as a percentage string. |
| `license_plate` | `HKY-2821` | OCR result for the license plate, if available. May be `null` when a plate is detected but OCR fails. Omitted when no plate is associated with the track. |

---

## Vehicle classification — `class`

The `class` field always identifies the tracked road user, not the violation type.

| Value | Purpose |
|-------|---------|
| `Vehicle` | Car or other four-wheeled vehicle. |
| `Motorbike` | Motorcycle or two-wheeled vehicle. |

Example `class` values:

```
Vehicle
Motorbike
```

---

## Violation classification — `condition`

The `condition` field describes the detected safety violation. It is only meaningful when `violation` is `true`.

| `condition` | Description | Typical `class` | Internal model label |
|-------------|-------------|-----------------|----------------------|
| `Mobile` | Mobile phone use while driving | `Vehicle` | `Mobile` |
| `Without-Seatbelt` | Person without seatbelt | `Vehicle` | `Person-NoSeatbelt` |
| `No-Helmet` | Rider or passenger without helmet | `Motorbike` | `No-Helmet` |

Example `condition` values for violations:

```
Mobile
Without-Seatbelt
No-Helmet
```

### Conflict resolution

Before export, the pipeline may compare violation detections against positive evidence (e.g. `Helmet` vs `No-Helmet`, `Person-Seatbelt` vs `Person-NoSeatbelt`). If the positive detection confidence is more than 10% higher than the violation confidence, the violation is suppressed and **no message is sent** for that frame.

---

## Confidence fields

Confidence values are expressed as percentage strings, not raw floats.

| Field | Example | Purpose |
|-------|---------|---------|
| `vehicle_conf` | `92.7%` | Confidence of the vehicle/motorbike detection. |
| `violation_conf` | `51.2%` | Confidence of the violation detection (mobile, seatbelt, or helmet). |

Format: one decimal place followed by `%`, e.g. `"76.4%"`.

---

## License plate — `license_plate`

The `license_plate` field carries the OCR result for the vehicle's plate, when available.

| Value | Meaning |
|-------|---------|
| `"HKY-2821"` | OCR succeeded; plate text returned. |
| `null` | Plate region detected but OCR failed or text is unreadable. |
| *(field absent)* | No plate associated with this track in the current or stored history. |

The tool may reuse a previously read plate from the same `objectID` when the plate is not visible in the current frame.

Supported OCR formats are configured at runtime (e.g. `GR`, `UK`).

---

## Media reference — `imageURL`

Large binary payloads (images) are not embedded in the JSON. The claim-check pattern is used instead: the frame is uploaded to MinIO and referenced by a relative URI.

```json
"imageURL": "images/violation_nasia_4-mobile_2026.3.3-0:12:35/frame_000058.jpg"
```

| Part | Example | Purpose |
|------|---------|---------|
| Prefix | `images/` | Media root inside the shared bucket. |
| Run folder | `violation_nasia_4-mobile_2026.3.3-0:12:35` | Identifies the processing run (input name + timestamp). |
| Filename | `frame_000058.jpg` | Zero-padded frame number. |

Recommended conventions:

- Do not embed base64 image data in the JSON payload.
- Keep `imageURL` as a relative path or presigned URI, depending on deployment.
- Use `.jpg` for exported frames (default JPEG quality: 80, max width: 1280 px).

---

## Example payload — `T4.1-01.json`

```json
{
  "records": [
    {
      "header": {
        "project-id": "iDriving",
        "use-case-id": "UC1.1",
        "contentType": "application/json",
        "message-type": "idriving_certh_uc1.1_detection",
        "producer-id": "CERTH",
        "correlation-id": "2829dc7f-6553-423c-8a2f-1200e969bb1e",
        "message-timestamp": "2026-03-02T22:12:41.117082+00:00"
      },
      "body": {
        "detection_list": [
          {
            "topicName": "idriving_certh_object_detection_uc1.1",
            "frameID": 58,
            "imageURL": "images/violation_nasia_4-mobile_2026.3.3-0:12:35/frame_000058.jpg",
            "detections": [
              {
                "objectID": 1,
                "class": "Vehicle",
                "vehicle_conf": "92.7%",
                "violation": true,
                "condition": "Mobile",
                "violation_conf": "51.2%",
                "license_plate": "HKY-2821"
              }
            ]
          }
        ]
      }
    }
  ]
}
```

---

## Heartbeat data format

When running with `--kafka`, the tool also publishes periodic liveness messages to confirm it is operational.

| Purpose | Topic |
|---------|-------|
| Component heartbeat | `idriving.heartbeat.certh.detect_violation` |

Sent every 30 seconds while the tool is running.

```json
{
  "component_id": "idriving.heartbeat.certh.detect_violation",
  "timestamp": "2026-03-02T22:12:41.117082+00:00",
  "status": "UP",
  "version": "1.0.0"
}
```

| Field | Example | Purpose |
|-------|---------|---------|
| `component_id` | `idriving.heartbeat.certh.detect_violation` | Unique identifier of the running component. |
| `timestamp` | `2026-03-02T22:12:41.117082+00:00` | UTC timestamp of the heartbeat (ISO 8601). |
| `status` | `UP` | Component health status. |
| `version` | `1.0.0` | Tool version string. |

Example `status` values:

```
UP
```

---

## How to use this folder

1. Copy `T4.1-01.json` as the reference payload for interface **T4.1-01** (violation detection event).
2. Keep the envelope conventions consistent (`project-id`, `use-case-id`, topic pattern, `records[]` structure).
3. Adapt `message-type`, `producer-id`, and `detections[]` contents to your deployment.
4. Add additional interface files (e.g. `T4.1-02.json`) for other tools in the T4.1 component family as they are documented.

Keep the envelope, header fields, timestamp format, `condition` naming, and claim-check media references consistent so violation data remains comparable across iDriving tools.
