# T4.1 — CERTH Edge Detection Tools

**Lead partner:** CERTH  
**iDriving components:** IC8–IC13 (+ UC2.1 Karlovac adjunct tool)

---

## UC1.1

### IC8.1 & IC8.2 — Violation detection

Reference payload for interface **T4.1-01**: `T4.1-01.json`.  
Not a strict schema — illustrates structure and naming conventions for the CERTH `detect_violation` edge tool.

The tool analyses video/camera streams, detects vehicles and motorbikes, and publishes Kafka messages when a safety violation is found. Frames are stored in MinIO and referenced via `imageURL` (claim-check pattern).

| `condition` | Description | Typical `class` |
|-------------|-------------|-----------------|
| `Mobile` | Mobile phone use | `Vehicle` |
| `Without-Seatbelt` | Person without seatbelt | `Vehicle` |
| `No-Helmet` | Rider/passenger without helmet | `Motorbike` |

#### Message envelope

Each Kafka message has **transport headers** (in Kafka headers, not in the JSON body) and a **JSON payload** with a `records[]` array. Each record has `header` + `body`.

**Topic (T4.1-01):** `idriving_certh_object_detection_uc1.1`  
**Pattern:** `idriving_<owner>_<purpose>_uc<X.Y>`

##### Kafka headers & `records[].header`

| Field | Example | Purpose |
|-------|---------|---------|
| `project-id` | `iDriving` | Project identifier |
| `use-case-id` | `UC1.1` | Use-case scope |
| `message-type` | `idriving_certh_uc1.1_detection` | Event type / schema |
| `producer-id` | `CERTH` | Producing component |
| `correlation-id` | `2829dc7f-6553-423c-8a2f-1200e969bb1e` | Distributed tracing (UUID) |
| `message-timestamp` | `2026-03-02T22:12:41.117082+00:00` | Producer UTC timestamp (ISO 8601) |
| `content-type` / `contentType` | `application/json` | Payload format |

#### Payload — `body.detection_list[]`

One entry per exported frame.

| Field | Example | Purpose |
|-------|---------|---------|
| `topicName` | `idriving_certh_object_detection_uc1.1` | Target Kafka topic |
| `frameID` | `58` | Frame number in input stream |
| `imageURL` | `images/violation_nasia_4-mobile_2026.3.3-0:12:35/frame_000058.jpg` | MinIO/S3 URI (no embedded image) |
| `detections` | `[ ... ]` | Violation records for this frame |

**Publish policy:** only when `violation: true` — on first detection per `objectID`, then every N frames (default: 5) while the violation persists.

#### `detections[]`

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

### IC9 — Vehicle count

Reference payload for interface **T4.1-02**: `T4.1-02.json`.  
Not a strict schema — illustrates structure and naming for the CERTH `count_vehicles` tool (IC9).

Detects and tracks Bus, Car, Motorbike, Truck (YOLOv11 + ByteTrack) and publishes vehicle counts. All classes aggregate into `vehicles` / `total`; `motorbikes` is currently always `0`.

#### Message envelope

**Topic (T4.1-02):** `idriving_certh_countcars_detectiontool_uc1.1`  
**Pattern:** `idriving_<owner>_<purpose>_uc<X.Y>`

Payload: top-level `header` + `body` (not wrapped in `records[]`).

##### Kafka headers & `header`

| Field | Example | Purpose |
|-------|---------|---------|
| `project-id` | `iDriving` | Project identifier |
| `use-case-id` | `UC1.1` | Use-case scope |
| `message-type` | `uc1.1_vehicle_count` | Event type / schema |
| `producer-id` | `CERTH` | Producing component |
| `correlation-id` | `a1b2c3d4-...` | Distributed tracing (UUID) |
| `message-timestamp` | `2026-03-02T22:12:41+00:00` | Producer UTC timestamp (ISO 8601) |
| `content-type` / `contentType` | `application/json` | Payload format |

#### Payload — `body`

| Field | Example | Purpose |
|-------|---------|---------|
| `topicName` | `idriving_certh_countcars_detectiontool_uc1.1` | Target Kafka topic |
| `frameID` | `150` or `"000120-000270"` | Frame number (int) or frame range (string) |
| `counts` | `{ vehicles, motorbikes, total }` | Vehicle count metrics |
| `imageURL` | *(optional)* | MinIO/S3 frame URI (claim-check) |
| `videoURL` | *(optional)* | MinIO/S3 video clip URI (claim-check) |

##### `counts`

| Field | Example | Purpose |
|-------|---------|---------|
| `vehicles` | `12` | Tracked vehicles in scope |
| `motorbikes` | `0` | Reserved (always `0` currently) |
| `total` | `12` | `vehicles` + `motorbikes` |

**Publish policy:** only when `total > 0`. Default mode: snapshot per frame, max once every `--mf` s (default 5). With `--upload-video`: unique track IDs per 5 s segment, one message per clip with `videoURL`.

**Related tool (UC2.1 Karlovac, T4.1-09):** the **vehicle_detection** tool uses the same flat `header` + `body` envelope but publishes individual new-vehicle events in `violations[]` with drone `telemetry` instead of aggregated `counts`. Same detector classes (Bus, Car, Motorbike, Truck) and ByteTrack pipeline — without counting.

---

## UC1.2

### IC10.1 — Illegal parking on bike lane

Reference payload for interface **T4.1-03**: `T4.1-03.json`.  
Not a strict schema — illustrates structure and naming for the CERTH `detect_illegal_parking` tool.

Combines YOLOv8 bike-lane segmentation + vehicle detection + ByteTrack. Flags vehicles parked on bike lanes when overlap with the lane mask exceeds the configured threshold.

| `condition` | Description | `class` |
|-------------|-------------|---------|
| `Illegal Parking` | Vehicle overlapping bike lane | `Vehicle` |

#### Message envelope

Each Kafka message has **transport headers** (in Kafka headers, not in the JSON body) and a **JSON payload** with a `records[]` array. Each record has `header` + `body`.

**Topic (T4.1-03):** `idriving_certh_object_detection_uc1.2`  
**Pattern:** `idriving_<owner>_<purpose>_uc<X.Y>`

##### Kafka headers & `records[].header`

| Field | Example | Purpose |
|-------|---------|---------|
| `project-id` | `iDriving` | Project identifier |
| `use-case-id` | `UC1.2` | Use-case scope |
| `message-type` | `idriving_certh_uc1.2_illegal_parking_detection` | Event type / schema |
| `producer-id` | `CERTH` | Producing component |
| `correlation-id` | `cb5469d6-6708-47bd-b7a6-9ef49aa7ff98` | Distributed tracing (UUID) |
| `message-timestamp` | `2026-06-29T12:25:54.561204Z` | Producer UTC timestamp (ISO 8601) |
| `content-type` / `contentType` | `application/json` | Payload format |

#### Payload — `body.detection_list[]`

One entry per exported frame.

| Field | Example | Purpose |
|-------|---------|---------|
| `topicName` | `idriving_certh_object_detection_uc1.2` | Target Kafka topic |
| `frameID` | `144` | Frame number in input stream |
| `imageURL` | `""` or MinIO/static URI | Claim-check frame URI; empty if no upload configured |
| `detections` | `[ ... ]` | Vehicle records for this frame |

**Publish policy:** only when at least one vehicle has `violation: true` (overlap with bike lane ≥ threshold, default 20%).

#### `detections[]`

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

### IC10.2 — Sign occlusion detection

Reference payload for interface **T4.1-04**: `T4.1-04.json`.  
Not a strict schema — illustrates structure and naming for the CERTH `sign_occlusion` tool.

Detects vehicles (YOLO + ByteTrack), compares each vehicle bbox against a user-defined sign ROI, and publishes Kafka messages when overlap with the sign exceeds the configured threshold.

| `condition` | Description | `class` |
|-------------|-------------|---------|
| `sign obstruction` | Vehicle blocking / occluding the traffic sign | `Vehicle` |

#### Message envelope

Each Kafka message has **transport headers** (in Kafka headers, not in the JSON body) and a **JSON payload** with a `records[]` array. Each record has `header` + `body`.

**Topic (T4.1-04):** `idriving_certh_object_detection_uc1.2`  
**Pattern:** `idriving_<owner>_<purpose>_uc<X.Y>`

##### Kafka headers & `records[].header`

| Field | Example | Purpose |
|-------|---------|---------|
| `project-id` | `iDriving` | Project identifier |
| `use-case-id` | `UC1.2` | Use-case scope |
| `message-type` | `idriving_certh_object_detection_uc1.2` | Event type / schema |
| `producer-id` | `CERTH` | Producing component |
| `correlation-id` | `ca4da452-07eb-4916-8929-1c9ec2ff4071` | Distributed tracing (UUID) |
| `message-timestamp` | `2026-03-05T08:51:48.422206+00:00` | Producer UTC timestamp (ISO 8601) |
| `content-type` / `contentType` | `application/json` | Payload format |

#### Payload — `body.detection_list[]`

One entry per exported frame.

| Field | Example | Purpose |
|-------|---------|---------|
| `topicName` | `idriving_certh_object_detection_uc1.2` | Target Kafka topic |
| `frameID` | `602` | Frame number in input stream |
| `imageURL` | `sign_occlusion_2026-3-5_10:51:30/frames/frame_000602.jpg` | MinIO relative path or presigned URI |
| `detections` | `[ ... ]` | Sign-occlusion records for this frame |

**Publish policy:** only when at least one vehicle overlaps the sign ROI ≥ `overlap_threshold_percent` (default: 30%). Only blocking vehicles are included in `detections`.

#### `detections[]`

| Field | Example | Purpose |
|-------|---------|---------|
| `objectID` | `1` | ByteTrack ID (stable per vehicle across frames) |
| `class` | `Vehicle` | Detected vehicle class |
| `vehicle_conf` | `94.9%` | Vehicle detector confidence (percentage string) |
| `violation` | `true` | Always `true` in exported T4.1-04 messages |
| `condition` | `sign obstruction` | Occlusion violation type |
| `violation_conf` | `100.0%` | Sign ROI overlap as percentage (`vehicle ∩ sign / sign area × 100`) |
| `license_plate` | `ZIA-5163` | OCR result; empty string `""` if no plate read |

**`violation_conf`:** overlap ratio between vehicle bbox and sign ROI, not a separate classifier score. Alert fires when `violation_conf` ≥ threshold (default 30%).

**`imageURL`:** relative MinIO path (`<run_folder>/frames/frame_NNNNNN.jpg`) or presigned URL with `--upload`.

**`license_plate`:** optional OCR on plate detections inside the blocking vehicle bbox.

---

### IC11.1 — Helmet & mobile violation detection

Reference payload for interface **T4.1-05**: `T4.1-05.json`.  
Not a strict schema — illustrates structure and naming for the CERTH `detect_helmet-mobile` tool.

Detects cyclists and motorcyclists (YOLO + ByteTrack) and flags helmet and mobile-phone violations. Frames are stored in MinIO and referenced via `imageURL` (claim-check pattern).

| `condition` | Description | Typical `class` |
|-------------|-------------|-----------------|
| `No-Helmet` | Rider without helmet | `Motorcyclist`, `Cyclist` |
| `smartphone_usage` | Mobile phone use while riding | `Motorcyclist`, `Cyclist` |
| `No-Helmet+smartphone_usage` | Both violations on the same rider | `Motorcyclist`, `Cyclist` |

> Internal model label `Mobile` is mapped to `smartphone_usage` in exported messages. Multiple simultaneous violations are joined with `+`.

#### Message envelope

Each Kafka message has **transport headers** (in Kafka headers, not in the JSON body) and a **JSON payload** with a `records[]` array. Each record has `header` + `body`.

**Topic (T4.1-05):** `idriving_certh_object_detection_uc1.2`  
**Pattern:** `idriving_<owner>_<purpose>_uc<X.Y>`

##### Kafka headers & `records[].header`

| Field | Example | Purpose |
|-------|---------|---------|
| `project-id` | `iDriving` | Project identifier |
| `use-case-id` | `UC1.2` | Use-case scope |
| `message-type` | `idriving_certh_uc1.2_helmet_mobile_detection` | Event type / schema |
| `producer-id` | `CERTH` | Producing component |
| `correlation-id` | `1f4abbee-0924-43b1-b1f1-bca712ac88f2` | Distributed tracing (UUID) |
| `message-timestamp` | `2026-03-03T11:33:48.664140+00:00` | Producer UTC timestamp (ISO 8601) |
| `content-type` / `contentType` | `application/json` | Payload format |

#### Payload — `body.detection_list[]`

One entry per exported frame.

| Field | Example | Purpose |
|-------|---------|---------|
| `topicName` | `idriving_certh_object_detection_uc1.2` | Target Kafka topic |
| `frameID` | `102` | Frame number in input stream |
| `imageURL` | `violation_uc1.2_motorbike-no_helmet-mobile_2026.3.3-13_33_44/frames/frame_000102.jpg` | MinIO relative path or presigned URI |
| `detections` | `[ ... ]` | Violation records for this frame |

**Publish policy:** only when `violation: true`. Default: export every violation frame (`upload_every_violation_frame: true`). Alternative: first detection per `objectID`, then every N frames (default: 5).

#### `detections[]`

| Field | Example | Purpose |
|-------|---------|---------|
| `objectID` | `1` | ByteTrack ID (stable per rider across frames) |
| `class` | `Motorcyclist` | `Motorcyclist` or `Cyclist` |
| `vehicle_conf` | `91.0%` | Rider detector confidence (percentage string) |
| `violation` | `true` | Always `true` in exported T4.1-05 messages |
| `condition` | `No-Helmet` | Violation type (see table above) |
| `violation_conf` | `81.4%` | Violation detector confidence (percentage string) |
| `license_plate` | `AB-1234` | OCR result; `null` if OCR failed; omitted if no plate |

**`condition` mapping:** `Mobile` → `smartphone_usage`; combined violations use `+` (e.g. `No-Helmet+smartphone_usage`).

**`license_plate`:** may be reused from a previous frame for the same `objectID`. OCR format configurable via `--lp-format`.

**`imageURL`:** MinIO bucket `uc1.2-certh-helmet-mobile-violation` — relative path or presigned URL (JPEG, quality 80, max width 1280 px).

---

### IC11.2 — Cyclist hand-signal & helmet detection

Reference payload for interface **T4.1-06**: `T4.1-06.json`.  
Not a strict schema — illustrates structure and naming for the CERTH `cyclist_hand_detection` tool.

Detects cyclists (YOLO + ByteTrack), checks helmet status, and analyses upper-body pose (YOLO-pose) for hand signals during turns. Frames are stored in MinIO and referenced via `imageURL` (claim-check pattern).

| `condition` | `violation` | Description |
|-------------|-------------|-------------|
| `No-Helmet` | `true` | Cyclist without helmet |
| `Helmet` | `false` | Cyclist wearing helmet |
| `null` | `false` | Cyclist detected, helmet status unknown / not set |

> Hand-signal analysis runs internally (`hand_label`), but **hand-signal violations are not exported** in the current version — only helmet violations set `violation: true`. For full `No-Hand-Signal` / `Wrong-Hand-Signal` export see `uc1.2-cyclist_hand_violation`.

#### Message envelope

Each Kafka message has **transport headers** (in Kafka headers, not in the JSON body) and a **JSON payload** with a `records[]` array. Each record has `header` + `body`.

**Topic (T4.1-06):** `idriving_certh_object_tracking_uc1.2`  
**Pattern:** `idriving_<owner>_<purpose>_uc<X.Y>`

##### Kafka headers & `records[].header`

| Field | Example | Purpose |
|-------|---------|---------|
| `project-id` | `iDriving` | Project identifier |
| `use-case-id` | `UC1.2` | Use-case scope |
| `message-type` | `cyclist_Signal_Violation` | Event type / schema |
| `producer-id` | `CERTH` | Producing component |
| `correlation-id` | `cafea1f0-a934-44df-aeb5-c92fc55d7b03` | Distributed tracing (UUID) |
| `message-timestamp` | `2026-03-03T21:22:23.238038+00:00` | Producer UTC timestamp (ISO 8601) |
| `content-type` / `contentType` | `application/json` | Payload format |

#### Payload — `body.detection_list[]`

One entry per exported frame.

| Field | Example | Purpose |
|-------|---------|---------|
| `topicName` | `idriving_certh_object_tracking_uc1.2` | Target Kafka topic |
| `frameID` | `163` | Frame number in input stream |
| `imageURL` | `images/test6_run_2026.03.03-23.22.13/frame_000163.jpg` | MinIO relative path or presigned URI |
| `detections` | `[ ... ]` | Cyclist records for this frame |

**Publish policy:** exports every frame with at least one tracked cyclist when `--kafka`, `--save-json`, or `--upload-*` is enabled. MinIO frame upload requires a non-empty `hand_label` and `--upload-frames`.

#### `detections[]`

| Field | Example | Purpose |
|-------|---------|---------|
| `objectID` | `5` | ByteTrack ID (stable per cyclist across frames) |
| `class` | `cyclist` | Always `cyclist` |
| `confidence` | `89.6%` | Cyclist detector confidence (percentage string) |
| `violation` | `true` | `true` when `condition` is `No-Helmet` |
| `condition` | `No-Helmet` | Violation / status label (see table above) |
| `violation_conf` | `89.6%` | Confidence of the violation detection |
| `bbox` | `[x1, y1, x2, y2]` | Cyclist bounding box in pixel coordinates |
| `hand_label` | `right hand raised` | Pose-derived hand state (`""` if none); not a violation field |

**`hand_label` values (informational):** `left hand raised`, `right hand raised`, `both hands raised`, or `""`.

**`imageURL`:** MinIO bucket `uc1.2-certh-cyclist-signal` — relative path or presigned URL (JPEG, quality 80, max width 1280 px).

---

### IC12 — Road user tracking

Reference payload for interface **T4.1-07**: `T4.1-07.json`.  
Not a strict schema — illustrates structure and naming for the CERTH `detect_behaviour_uc1_2` tool.

Detects and tracks road users (Vehicle, Pedestrian, Cyclist via YOLO + ByteTrack), optionally reads license plates for vehicles, and publishes **tracking** messages (no violation logic). Intended for downstream behaviour analysis (e.g. yielding, crossing, turn scenarios).

| `class` | Description |
|---------|-------------|
| `Vehicle` | Car or other vehicle |
| `Pedestrian` | Pedestrian |
| `Cyclist` | Cyclist |

> `Plate` is detected internally for OCR but is **not** exported as a tracked object.

#### Message envelope

Each Kafka message has **transport headers** (in Kafka headers, not in the JSON body) and a **JSON payload** with a `records[]` array. Each record has `header` + `body`.

**Topic (T4.1-07):** `idriving_certh_object_tracking_uc1.2`  
**Pattern:** `idriving_<owner>_<purpose>_uc<X.Y>`

##### Kafka headers & `records[].header`

| Field | Example | Purpose |
|-------|---------|---------|
| `project-id` | `iDriving` | Project identifier |
| `use-case-id` | `UC1.2` | Use-case scope |
| `message-type` | `idriving_certh_object_tracking_uc1.2` | Event type / schema |
| `producer-id` | `CERTH` | Producing component |
| `correlation-id` | `a3b2c1d0-e5f6-7890-abcd-ef1234567890` | Distributed tracing (UUID) |
| `message-timestamp` | `2026-06-29T19:22:15.123456+00:00` | Producer UTC timestamp (ISO 8601) |
| `content-type` / `contentType` | `application/json` | Payload format |

#### Payload — `body.detection_list[]`

One entry per exported frame. Tracking messages **do not** include `imageURL`.

| Field | Example | Purpose |
|-------|---------|---------|
| `topicName` | `idriving_certh_object_tracking_uc1.2` | Target Kafka topic |
| `frameID` | `142` | Frame number in input stream |
| `detections` | `[ ... ]` | Tracked objects in this frame |

**Publish policy:** every frame with at least one tracked object when `--kafka` or `--save-json` is enabled.

#### `detections[]`

| Field | Example | Purpose |
|-------|---------|---------|
| `objectID` | `1` | ByteTrack ID (stable per object across frames) |
| `class` | `Vehicle` | `Vehicle`, `Pedestrian`, or `Cyclist` |
| `vehicle_conf` | `91.2%` | Detector confidence (percentage string) |
| `bbox` | `[120.5, 80.0, 450.3, 320.7]` | Bounding box `[x1, y1, x2, y2]` in pixels |
| `license_plate` | `ABC-1234` | OCR result for vehicles only; omitted for Pedestrian/Cyclist |

**No violation fields:** this interface carries tracking data only (`violation`, `condition`, `violation_conf` are not used).

**`license_plate`:** only for `Vehicle`. May be reused from a previous frame for the same `objectID`. OCR format configurable via `--lp-format`.

**Local JSON filename:** `frame_NNNNNN_tracking.json` when saved with `--save-json`.

---

## UC2.1

### Drone vehicle detection — Karlovac *(no IC; adjunct to IC9)*

Reference payload for interface **T4.1-09**: `T4.1-09.json`.  
Not a strict schema — illustrates structure and naming for the CERTH `vehicle_detection` tool (UC2.1 Karlovac).

Detects road vehicles from drone or ground camera feed (YOLOv11 + ByteTrack) and publishes a Kafka message when a **new** tracked vehicle appears. Includes GPS telemetry from MAVLink when `--telemetry` is enabled.

| `class` | Description |
|---------|-------------|
| `Bus` | Bus |
| `Car` | Car |
| `Motorbike` | Motorbike |
| `Truck` | Truck |

> Messages use the term **violation** in code/config (`uc2.1_vehicle_violation`) to mean a **new vehicle detection event**, not a traffic-rule violation.  
> Same flat `header` + `body` envelope as **IC9** (T4.1-02) but with `violations[]` + `telemetry` instead of `counts`.

#### Message envelope

**Topic (T4.1-09):** `idriving_certh_object_detection_uc2.1`  
**Pattern:** `idriving_<owner>_<purpose>_uc<X.Y>`

Payload: top-level `header` + `body` (not wrapped in `records[]`).

##### Kafka headers & `header`

| Field | Example | Purpose |
|-------|---------|---------|
| `project-id` | `iDriving` | Project identifier |
| `use-case-id` | `UC2.1` | Use-case scope |
| `message-type` | `uc2.1_vehicle_violation` | Event type / schema |
| `producer-id` | `CERTH` | Producing component |
| `correlation-id` | `b4c5d6e7-f8a9-0123-bcde-f45678901234` | Distributed tracing (UUID) |
| `message-timestamp` | `2026-06-29T14:30:22.456789+00:00` | Producer UTC timestamp (ISO 8601) |
| `content-type` / `contentType` | `application/json` | Payload format |

#### Payload — `body`

| Field | Example | Purpose |
|-------|---------|---------|
| `topicName` | `idriving_certh_object_detection_uc2.1` | Target Kafka topic |
| `frameID` | `245` | Frame number when the new vehicle was first seen |
| `violations` | `[ ... ]` | New vehicle detection(s) in this message |
| `telemetry` | `{ lat, lon, alt }` | Drone GPS position at detection time |
| `imageURL` | *(optional)* | MinIO presigned URI of annotated frame (`--upload-frames`) |
| `videoURL` | *(optional)* | Reserved in schema; not attached to per-track messages currently |

**Publish policy:** one message per **new** `track_id` (first appearance only). Each message typically contains a single entry in `violations[]`.

#### `violations[]`

| Field | Example | Purpose |
|-------|---------|---------|
| `track_id` | `3` | ByteTrack ID (stable per vehicle across frames) |
| `class` | `Car` | Detected vehicle class |
| `score` | `0.872` | Detector confidence (raw float, 0–1) |
| `bbox` | `[412.3, 180.5, 598.7, 340.2]` | Bounding box `[x1, y1, x2, y2]` in pixels |

#### `telemetry`

| Field | Example | Purpose |
|-------|---------|---------|
| `lat` | `45.4923` | GPS latitude (degrees) |
| `lon` | `15.5558` | GPS longitude (degrees) |
| `alt` | `170.4` | GPS altitude (metres) |

All fields are `null` when telemetry is disabled or unavailable. With `--mock-telemetry`, fixed coordinates from config are used.

**Local JSON filename:** `violation_NNNNNN_track_<id>.json` when saved with `--save-json`.

**MinIO bucket:** `uc2.1-certh-vehicle-detection`.

---

## UC2.2

### IC13 — Obstacle detection

Reference payload for interface **T4.1-08**: `T4.1-08.json`.  
Not a strict schema — illustrates structure and naming for the CERTH `obstacle_detection` tool (IC13).

Detects road obstacles from drone imagery (YOLO11) and publishes Kafka messages when export-class obstacles are found. Frames are stored in MinIO and referenced via `imageURL` (claim-check pattern).

| `class` | Exported | Description |
|---------|----------|-------------|
| `Fallen Tree` | yes | Fallen tree on road |
| `Normal Tree` | no | Reference/background class (excluded from export) |
| `Rocks` | yes | Rock debris |
| `Root` | yes | Exposed tree root |
| `Stump` | yes | Tree stump |

> `Normal Tree` (class id 1) is detected and drawn but excluded from JSON/Kafka via `export_exclude_class_id` in config.

#### Message envelope

Each Kafka message has **transport headers** (in Kafka headers, not in the JSON body) and a **JSON payload** with a `records[]` array. Each record has `header` + `body`.

**Topic (T4.1-08):** `idriving_certh_object_detection_uc2.2`  
**Pattern:** `idriving_<owner>_<purpose>_uc<X.Y>`

##### Kafka headers & `records[].header`

| Field | Example | Purpose |
|-------|---------|---------|
| `project-id` | `iDriving` | Project identifier |
| `use-case-id` | `UC2.2` | Use-case scope |
| `message-type` | `idriving_certh_uc2.2_obstacle_detection` | Event type / schema |
| `producer-id` | `CERTH` | Producing component |
| `correlation-id` | `41f92162-aa17-46c0-bd16-31b50ae7233c` | Distributed tracing (UUID) |
| `message-timestamp` | `2026-05-29T09:01:03.360980Z` | Producer UTC timestamp (ISO 8601) |
| `content-type` / `contentType` | `application/json` | Payload format |

#### Payload — `body.detection_list[]`

One entry per exported frame.

| Field | Example | Purpose |
|-------|---------|---------|
| `topicName` | `idriving_certh_object_detection_uc2.2` | Target Kafka topic |
| `frameID` | `142` | Frame number in input stream |
| `imageURL` | `""` or MinIO presigned URI | Claim-check frame URI; empty if no upload configured |
| `longitude` | `23.77` | Drone longitude (degrees); random placeholder until telemetry wired |
| `latitude` | `38.15` | Drone latitude (degrees) |
| `altitude` | `48.8` | Drone altitude (metres) |
| `detections` | `[ ... ]` | Obstacle records for this frame |

**Publish policy:** when at least one export-class obstacle is detected and `--kafka`, `--save-json`, or `--upload-*` is enabled.

#### `detections[]`

| Field | Example | Purpose |
|-------|---------|---------|
| `objectID` | `1` | Sequential index within the message (1-based); not a tracker ID |
| `class` | `Fallen Tree` | Obstacle class name |
| `confidence` | `63.3%` | Detector confidence (percentage string) |

**No bbox in export:** bounding boxes are computed internally for drawing but are not included in the Kafka JSON.

**GNSS fields:** `longitude`, `latitude`, `altitude` sit at the `detection_list[]` item level (not inside each detection). Placeholder random values are used until a drone telemetry consumer is connected.

**`imageURL`:** MinIO bucket `uc2.2-certh-obstacle-detection` — presigned URL with `--upload-frames`, or `""` when saving JSON only.
