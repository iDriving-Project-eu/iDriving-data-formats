# T3.2 — Route Guidance & Signal Control Tools

**Lead partner:** MBL
**iDriving components:** IC2, IC3

See the [`_example/`](../_example) folder for a generic representative payload and the
header/topic conventions every message should follow.



The payload for the route guidance tool is [`T3.2_01.json`](T3.2_01.json). The one for the signal control tool is [`T3.2_02.json`](T3.2_02.json). The included information and tags are described below. 

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

### Record contents: 

#### alert_info{}: 
| Header key | Example | Purpose |
|---|---|---|
| `type` | `"obstacle"` | Identifies the type of incident/alert (e.g. 'obstacle','traffic','accident' etc.) |
| `notes` | `"Leoforos Ochi"` | Identifies the name of the street where the incident takes place. | 
| `coordinates` | `["12346678","12345678"] ` | Identifies the latitude and longitude of the incident location (if localized). |
| `default_alert` | `"Road Closed ahead! Enter the highway from Triandria intersection!"` | Identifies the preset warning message related to the specific incident/alert with general routing advice. |
| `affected_vehicle_types` | `[“car”, “motorcycle”, “truck”]` | Identifies the types of vehicles that are affected by the incident. | 


#### alternative_routes{}: 
| Header key | Example | Purpose |
|---|---|---|
| `vehicle_type` | `"car"` | Identifies the type of vehicle for which the route is intended. | 
| `start_point` | `"Λ. Όχι"` | Indentifies the starting point of the rerouted trip. | 
| `destination_point` | `"Περιφερειακή Οδός Θεσσαλονίκης"` | Indentifies the destination point of the rerouted trip. | 
| `new_path` | `"('Λ. Όχι', 'Ελένης Ζωγράφου', 'Ολυμπιάδος', 'Αγίου Δημητρίου', 'Κατσιμίδη', 'Λ. Κυριακίδη', 'Περιφερειακή Οδός Θεσσαλονίκης')"` | Indentifies the alternative path proposed for this trip and vehicle type. | 

## How to use this

Copy the example into your component folder, rename it by interface ID
(e.g. `T4.1-01.json`), and adapt the `message-type`, `producer-id`, and the contents of
each record to your interface. Keep the envelope and the header/topic conventions
consistent so the formats stay comparable across tools.
