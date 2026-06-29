# T5.3 — SUMO & CARLA (Digital Twin simulation environment)

**Lead partner:** UNI.EIFFEL
**iDriving components:** IC24, IC25

# Alerts data format

This folder holds a **generic, representative example** of the alerts data format exchanged between iDriving tools. It is a reference, not a strict schema — use it to understand the structure, naming conventions, and expected content, then place your own component-specific JSON files in your component folder.


## Alerts array

The `alerts` array contains one or more alert records. Each alert describes a traffic condition, such as predicted congestion, on a specific road or road segment.

```json
{
  "label": "Congestion on Wickenburggasse",
  "metric": "veh/h",
  "status": "predicted",
  "user_type": "car",
  "location": [0.0, 0.0],
  "time_window": 600,
  "number_event": 8,
  "priority": 10,
  "start_time": "2026-02-19T22:53:09.816086Z",
  "end_time": "2026-02-19T23:29:09.816086Z",
  "road_name": "Wickenburggasse"
}
```

## Alert record

Each entry in `body.alerts` describes one traffic-related alert.

| Field | Example | Purpose |
|---|---|---|
| `label` | `Congestion on Wickenburggasse` | Human-readable description of the alert. |
| `metric` | `veh/h` | Unit or metric used to evaluate the alert. |
| `status` | `predicted` | Indicates whether the alert is predicted, detected, active, resolved, or another supported status. |
| `user_type` | `car` | Road-user category affected by the alert. |
| `location` | `[0.0, 0.0]` | Geographic location of the alert, usually expressed as a two-value coordinate array. |
| `time_window` | `600` | Time window used for the alert calculation, expressed in seconds. |
| `number_event` | `8` | Number of events or observations contributing to the alert. |
| `priority` | `10` | Alert priority or severity level. Higher values may indicate higher urgency, depending on the consuming component. |
| `start_time` | `2026-02-19T22:53:09.816086Z` | Start timestamp of the alert validity period. |
| `end_time` | `2026-02-19T23:29:09.816086Z` | End timestamp of the alert validity period. |
| `road_name` | `Wickenburggasse` | Name or identifier of the road where the alert applies. |

## Alert description

The alert description is mainly provided through the `label` and `road_name` fields.

| Field | Example | Purpose |
|---|---|---|
| `label` | `Congestion on Keplerbrücke` | Human-readable alert text suitable for dashboards, logs, or downstream services. |
| `road_name` | `Keplerbrücke` | Road name or road segment identifier associated with the alert. |

When no official road name is available, a network or simulation road identifier may be used instead, for example:

```text
-426237458#0
```

## Alert classification

Classification fields describe the alert status, affected road-user category, metric, and priority.

| Field | Example | Purpose |
|---|---|---|
| `status` | `predicted` | Indicates the lifecycle or confidence state of the alert. |
| `user_type` | `car` | User category affected by the alert. |
| `metric` | `veh/h` | Metric used to generate or interpret the alert. |
| `priority` | `10` | Relative importance or urgency of the alert. |

Example `status` values may include:

- `predicted`
- `detected`

Example `user_type` values may include:

- `car`
- `vehicle`
- `pedestrian`
- `motorbike`
- `bicycle`
- `bus`
- `truck`

## Location

The `location` field stores the alert position as a two-value numeric array.

```json
"location": [0.0, 0.0]
```

The coordinate order should be kept consistent across all components.

Recommended conventions:

- Use valid geographic coordinates when available.
- Document whether the order is `[longitude, latitude]` or `[latitude, longitude]`.
- Use `[0.0, 0.0]` only as a placeholder when the exact location is unknown.
- Keep coordinates numeric, not string-based.

## Time information

Each alert includes both a calculation window and an alert validity period.

| Field | Example | Purpose |
|---|---|---|
| `time_window` | `600` | Aggregation or prediction window in seconds. |
| `start_time` | `2026-02-19T22:53:09.816086Z` | Start of the alert period. |
| `end_time` | `2026-02-19T23:29:09.816086Z` | End of the alert period. |

Timestamps should use ISO 8601 format in UTC when possible.

## Event count

The `number_event` field indicates how many observations, detections, or predicted events contributed to the alert.

| Field | Example | Purpose |
|---|---:|---|
| `number_event` | `8` | Number of events associated with the alert. |

For congestion alerts, this may represent the number of detected or predicted vehicle-flow events within the configured time window.


