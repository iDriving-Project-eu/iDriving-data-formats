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
## IC9 - Vehicle count data format

Reference payload for interface **T4.1-02**: `T4.1-02.json`.  
Not a strict schema — illustrates structure and naming for the CERTH `count_vehicles` tool (IC9).

Detects and tracks Bus, Car, Motorbike, Truck (YOLOv11 + ByteTrack) and publishes vehicle counts. All classes aggregate into `vehicles` / `total`; `motorbikes` is currently always `0`.

---

## Message envelope

**Topic (T4.1-02):** `idriving_certh_countcars_detectiontool_uc1.1`  
**Pattern:** `idriving_<owner>_<purpose>_uc<X.Y>`

Payload: top-level `header` + `body` (not wrapped in `records[]`).

### Kafka headers & `header`

| Field | Example | Purpose |
|-------|---------|---------|
| `project-id` | `iDriving` | Project identifier |
| `use-case-id` | `UC1.1` | Use-case scope |
| `message-type` | `uc1.1_vehicle_count` | Event type / schema |
| `producer-id` | `CERTH` | Producing component |
| `correlation-id` | `a1b2c3d4-...` | Distributed tracing (UUID) |
| `message-timestamp` | `2026-03-02T22:12:41+00:00` | Producer UTC timestamp (ISO 8601) |
| `content-type` / `contentType` | `application/json` | Payload format |

---

## Payload — `body`

| Field | Example | Purpose |
|-------|---------|---------|
| `topicName` | `idriving_certh_countcars_detectiontool_uc1.1` | Target Kafka topic |
| `frameID` | `150` or `"000120-000270"` | Frame number (int) or frame range (string) |
| `counts` | `{ vehicles, motorbikes, total }` | Vehicle count metrics |
| `imageURL` | *(optional)* | MinIO/S3 frame URI (claim-check) |
| `videoURL` | *(optional)* | MinIO/S3 video clip URI (claim-check) |

### `counts`

| Field | Example | Purpose |
|-------|---------|---------|
| `vehicles` | `12` | Tracked vehicles in scope |
| `motorbikes` | `0` | Reserved (always `0` currently) |
| `total` | `12` | `vehicles` + `motorbikes` |

**Publish policy:** only when `total > 0`. Default mode: snapshot per frame, max once every `--mf` s (default 5). With `--upload-video`: unique track IDs per 5 s segment, one message per clip with `videoURL`.

---

## IC10.1 - UC1.2 — Illegal parking on bike lane

Reference payload for interface **T4.1-03**: `T4.1-03.json`.  
Not a strict schema — illustrates structure and naming for the CERTH `detect_illegal_parking` tool.

Combines YOLOv8 bike-lane segmentation + vehicle detection + ByteTrack. Flags vehicles parked on bike lanes when overlap with the lane mask exceeds the configured threshold.

| `condition` | Description | `class` |
|-------------|-------------|---------|
| `Illegal Parking` | Vehicle overlapping bike lane | `Vehicle` |

---

## Message envelope

Each Kafka message has **transport headers** (in Kafka headers, not in the JSON body) and a **JSON payload** with a `records[]` array. Each record has `header` + `body`.

**Topic (T4.1-03):** `idriving_certh_object_detection_uc1.2`  
**Pattern:** `idriving_<owner>_<purpose>_uc<X.Y>`

### Kafka headers & `records[].header`

| Field | Example | Purpose |
|-------|---------|---------|
| `project-id` | `iDriving` | Project identifier |
| `use-case-id` | `UC1.2` | Use-case scope |
| `message-type` | `idriving_certh_uc1.2_illegal_parking_detection` | Event type / schema |
| `producer-id` | `CERTH` | Producing component |
| `correlation-id` | `cb5469d6-6708-47bd-b7a6-9ef49aa7ff98` | Distributed tracing (UUID) |
| `message-timestamp` | `2026-06-29T12:25:54.561204Z` | Producer UTC timestamp (ISO 8601) |
| `content-type` / `contentType` | `application/json` | Payload format |

---

## Payload — `body.detection_list[]`

One entry per exported frame.

| Field | Example | Purpose |
|-------|---------|---------|
| `topicName` | `idriving_certh_object_detection_uc1.2` | Target Kafka topic |
| `frameID` | `144` | Frame number in input stream |
| `imageURL` | `""` or MinIO/static URI | Claim-check frame URI; empty if no upload configured |
| `detections` | `[ ... ]` | Vehicle records for this frame |

**Publish policy:** only when at least one vehicle has `violation: true` (overlap with bike lane ≥ threshold, default 20%).

---

## `detections[]`

| Field | Example | Purpose |
|-------|---------|---------|
| `objectID` | `1` | ByteTrack ID (stable per vehicle across frames) |
| `class` | `Vehicle` | Always `Vehicle` for illegal parking events |
| `vehicle_conf` | `84.5%` | Vehicle detector confidence (percentage string) |
| `violation` | `true` | `true` if vehicle overlaps bike lane above threshold |
| `condition` | `Illegal Parking` | Violation type when `violation` is `true`; otherwise `null` |
| `violation_conf` | `22.8%` | Bike-lane overlap ratio as percentage (not a separate classifier score) |
| `license_plate` | `AB-123-CD` | OCR result; `null` if OCR failed; omitted if no plate |

**`violation_conf`:** computed as `(vehicle ∩ bike_lane) / vehicle_area × 100`. Compared against `overlap_thres` in config (default `0.2` = 20%).

**`imageURL`:** MinIO presigned URL with `--upload-frames`, static server URL if configured, or `""` when only saving JSON locally.

**`license_plate`:** optional OCR (format configurable via `--lp-format`, default `FR`).

---





