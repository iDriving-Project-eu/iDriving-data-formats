# Example data format

This folder holds a **generic, representative example** of the data formats exchanged
between iDriving tools. It is a reference, not a strict schema — use it to understand the
shape and conventions we expect, then place your own component-specific JSON files in your
component folder.

The example payload is [`example-data-format.json`](example-data-format.json).

## Message envelope

Every message published on the iDriving Kafka message bus has two parts:

1. **Kafka message headers** — transport-level metadata, carried in the Kafka headers (not
   in the JSON body).
2. **JSON payload** — the business content, which itself begins with a `header` block that
   mirrors the key identifiers, followed by one or more `records`.

### Standard Kafka message headers

| Header key | Example | Purpose |
|---|---|---|
| `project-id` | `iDriving` | Identifies the project; constant across all messages. |
| `use-case-id` | `UC1.1` | Identifies the use-case scope of the message. |
| `message-type` | `object-detection-event` | Identifies the message schema / event type. |
| `producer-id` | `certh.detection-tools.t4.1` | Identifies the producing component instance. |
| `correlation-id` | `550e8400-e29b-41d4-a716-446655440000` | Correlates related messages across components for distributed tracing. |
| `message-timestamp` | `2026-05-19T08:30:00Z` | Producer-side UTC timestamp of the message. |
| `content-type` | `application/json` | Declares the payload format. |

### Topic naming pattern

Kafka topics follow the pattern `idriving_<owner>_<purpose>_uc<X.Y>`, where `<owner>`
identifies the producing partner or component, `<purpose>` describes the kind of message
carried on the topic, and `<X.Y>` identifies the use-case scope. The `<owner>` and
`<purpose>` segments are stable across deployments; only the use-case suffix varies between
instances of the same logical interface.

## Payload conventions

- A top-level `header` block repeats the key identifiers (`project-id`, `use-case-id`,
  `message-type`, `producer-id`, `correlation-id`, `message-timestamp`) plus a
  `schema-version`.
- Domain content goes in a `records` array so a single message can carry one or many items.
- Each record carries its own `record-id`, `timestamp`, and (where relevant) a `location`.
- **Large binary payloads** (images, video, point clouds) are **not** embedded. Instead,
  use the claim-check pattern: store the object in shared MinIO/S3 storage and reference it
  by URI (`claim-check-uri`), as shown in the `media` block of the example.

## How to use this

Copy the example into your component folder, rename it by interface ID
(e.g. `T4.1-01.json`), and adapt the `message-type`, `producer-id`, and the contents of
each record to your interface. Keep the envelope and the header/topic conventions
consistent so the formats stay comparable across tools.
