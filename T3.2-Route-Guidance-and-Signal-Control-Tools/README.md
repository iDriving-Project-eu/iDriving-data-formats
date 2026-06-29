# T3.2 — Route Guidance & Signal Control Tools

**Lead partner:** MBL
**iDriving components:** IC2, IC3

See the [`_example/`](../_example) folder for a generic representative payload and the
header/topic conventions every message should follow.



The payload for the route guidance tool is [`T3.2_01.json`](T3.2_01.json). 

### Kafka message headers

| Header key | Example | Purpose |
|---|---|---|
| `project-id` | `iDriving` | Identifies the project; constant across all messages. |
| `use-case-id` | `UC1.1` | Identifies the use-case scope of the message. |
| `message-type` | `object-detection-event` | Identifies the message schema / event type. |
| `producer-id` | `certh.detection-tools.t4.1` | Identifies the producing component instance. |
| `correlation-id` | `550e8400-e29b-41d4-a716-446655440000` | Correlates related messages across components for distributed tracing. |
| `message-timestamp` | `2026-05-19T08:30:00Z` | Producer-side UTC timestamp of the message. |
| `content-type` | `application/json` | Declares the payload format. |

## Record contents 



## How to use this

Copy the example into your component folder, rename it by interface ID
(e.g. `T4.1-01.json`), and adapt the `message-type`, `producer-id`, and the contents of
each record to your interface. Keep the envelope and the header/topic conventions
consistent so the formats stay comparable across tools.
