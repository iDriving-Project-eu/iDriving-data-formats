# T4.1 — CERTH Edge Detection Tools

**Lead partner:** CERTH  
**iDriving components:** IC8–IC14

## IC8.1 & IC8.2 Violation detection data format

Reference payload for interface **T4.1-01**: `T4.1-01.json`.  
Not a strict schema — illustrates structure and naming conventions for the CERTH `detect_violation` edge tool.

The tool analyses video/camera streams, detects vehicles and motorbikes, and publishes Kafka messages when a safety violation is found. Frames are stored in MinIO and referenced via `imageURL` (claim-check pattern).

| `condition` | Description | Typical `class` |
|-------------|-------------|-----------------|
| `Mobile` | Mobile phone use | `Vehicle` |
| `Without-Seatbelt` | Person without seatbelt | `Vehicle` |
| `No-Helmet` | Rider/passenger without helmet | `Motorbike` |

---

## Message envelope

Each Kafka message has **transport headers** (in Kafka headers, not in the JSON body) and a **JSON payload** with a `records[]` array. Each record has `header` + `body`.

**Topic (T4.1-01):** `idriving_certh_object_detection_uc1.1`  
**Pattern:** `idriving_<owner>_<purpose>_uc<X.Y>`

### Kafka headers & `records[].header`

| Field | Example | Purpose |
|-------|---------|---------|
| `project-id` | `iDriving` | Project identifier |
| `use-case-id` | `UC1.1` | Use-case scope |
| `message-type` | `idriving_certh_uc1.1_detection` | Event type / schema |
| `producer-id` | `CERTH` | Producing component |
| `correlation-id` | `2829dc7f-6553-423c-8a2f-1200e969bb1e` | Distributed tracing (UUID) |
| `message-timestamp` | `2026-03-02T22:12:41.117082+00:00` | Producer UTC timestamp (ISO 8601) |
| `content-type` / `contentType` | `application/json` | Payload format |

---

## Payload — `body.detection_list[]`

One entry per exported frame.

| Field | Example | Purpose |
|-------|---------|---------|
| `topicName` | `idriving_certh_object_detection_uc1.1` | Target Kafka topic |
| `frameID` | `58` | Frame number in input stream |
| `imageURL` | `images/violation_nasia_4-mobile_2026.3.3-0:12:35/frame_000058.jpg` | MinIO/S3 URI (no embedded image) |
| `detections` | `[ ... ]` | Violation records for this frame |

**Publish policy:** only when `violation: true` — on first detection per `objectID`, then every N frames (default: 5) while the violation persists.

---

## `detections[]`

| Field | Example | Purpose |
|-------|---------|---------|
| `objectID` | `1` | ByteTrack ID (stable per vehicle across frames) |
| `class` | `Vehicle` | `Vehicle` or `Motorbike` — never the violation label |
| `vehicle_conf` | `92.7%` | Vehicle detector confidence (percentage string) |
| `violation` | `true` | Always `true` in exported T4.1-01 messages |
| `condition` | `Mobile` | Violation type (see table above) |
| `violation_conf` | `51.2%` | Violation detector confidence (percentage string) |
| `license_plate` | `HKY-2821` | OCR result; `null` if OCR failed; omitted if no plate |

**`condition` mapping (internal → external):** `Mobile` → `Mobile`, `Person-NoSeatbelt` → `Without-Seatbelt`, `No-Helmet` → `No-Helmet`.

**Conflict resolution:** if positive evidence (`Helmet`, `Person-Seatbelt`) exceeds violation confidence by >10%, the violation is suppressed and no message is sent.

**`license_plate`:** may be reused from a previous frame for the same `objectID`. OCR format configurable (`GR`, `UK`, etc.).

**`imageURL`:** `images/<run_folder>/frame_NNNNNN.jpg` — JPEG, quality 80, max width 1280 px.

---

## Heartbeat

**Topic:** `idriving.heartbeat.certh.detect_violation` — sent every 30 s with `--kafka`.

| Field | Example | Purpose |
|-------|---------|---------|
| `component_id` | `idriving.heartbeat.certh.detect_violation` | Component identifier |
| `timestamp` | `2026-03-02T22:12:41.117082+00:00` | UTC timestamp (ISO 8601) |
| `status` | `UP` | Health status |
| `version` | `1.0.0` | Tool version |

---

