    # T5.1 — SCC Dynamic Monitoring Platform

**Lead partner:** UNI.EIFFEL
**iDriving components:** IC23

# Violation data format

This folder holds a **generic, representative example** of the violation data format exchanged between iDriving tools. It is a reference, not a strict schema — use it to understand the structure, naming conventions, and expected content, then place your own component-specific JSON files in your component folder.

The example payload is [`violation-data-format.json`](violation-data-format.json).

## Message envelope

Each violation message contains one or more `records`. Every record has two main parts:

1. **Header** — message-level metadata used to identify the project, use case, producer, message type, timestamp, and correlation ID.
2. **Body** — the business content, including the aggregation window, site information, road-level data, user groups, and violation metrics.


## Record body

The `body` block contains the violation statistics and contextual information.

| Field | Example | Purpose |
|---|---|---|
| `timestamp` | `2026-01-20T15:22:57.624701+00:00` | Timestamp when the record body was generated. |
| `window` | `{ ... }` | Defines the aggregation period covered by the metrics. |
| `version` | `1.0` | Version of this data format. |
| `site` | `{ ... }` | Identifies the use case, city, and network/site. |
| `data` | `[ ... ]` | Contains road-level violation data. |

## Aggregation window

The `window` object defines the time interval covered by the violation metrics.

| Field | Example | Purpose |
|---|---|---|
| `granularity` | `hour` | Aggregation level used for the metrics. |
| `start` | `2025-11-24T22:00:00+00:00` | Start of the aggregation window. |
| `end` | `2025-11-24T22:59:59.999999+00:00` | End of the aggregation window. |
| `timezone` | `UTC` | Timezone used for the aggregation window. |

## Site information

The `site` object identifies the deployment or network context.

| Field | Example | Purpose |
|---|---|---|
| `uc_id` | `UC1.1` | Use-case identifier. |
| `city` | `null` | City where the data was collected, if available. |
| `network_id` | `iDriving` | Identifier of the network or deployment. |

## Violation data

The `data` array contains one or more road-level entries. Each entry represents violation statistics for a specific road or road segment.

| Field | Example | Purpose |
|---|---|---|
| `road_id` | `unknown_road` | Identifier of the road or road segment. (Road, adresses, Digital Twin inner IDs) |
| `scc_domain` | `safety` | Smart City Challenge domain associated with the data. |
| `groups` | `[ ... ]` | User-category groups containing violation metrics. |

## User groups

Each road entry contains a `groups` array. Each group represents one road-user category, such as vehicles, pedestrians, or motorbikes.

| Field | Example | Purpose |
|---|---|---|
| `user` | `Vehicle` | Road-user category. |
| `metrics` | `{ ... }` | Aggregated violation metrics for this user category. |

Example user categories include:

- `Vehicle`
- `Pedestrian`
- `Motorbike`

## Metrics

Each group contains a `metrics` object with aggregated safety indicators.

| Metric | Example | Purpose |
|---|---:|---|
| `total_vehicles` | `14` | Total number of detected users or vehicles in this group. |
| `total_violations` | `14` | Total number of detected violations. |
| `helmet_violation` | `0` | Number of helmet-related violations. |
| `seatbelt_violation` | `9` | Number of seatbelt-related violations. |
| `redlight_violation` | `5` | Number of red-light violations. |
| `wrong_lane` | `0` | Number of wrong-lane violations. |
| `Smartphone Usage` | `0` | Number of smartphone-usage violations. |




Keep the envelope, header fields, timestamp format, and road-user grouping structure consistent so the violation data remains comparable across iDriving tools.
