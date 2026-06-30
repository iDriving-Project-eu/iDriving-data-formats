# T3.2 — Route Guidance & Signal Control Tools

**Lead partner:** MBL
**iDriving components:** IC2, IC3

The payload for the route guidance tool exists in different versions, corresponing to different use cases. The file used for route guidance in PUC1.1 is [`T3.2-RG-PUC1.1.json`](T3.2-RG-PUC1.1.json) and the one used in PUC2.2 is [`T3.2-RG-PUC2.2.json`](T3.2-RG-PUC2.2.json). The file used for the signal control tool (included only in PUC1.1) is [`T3.2-SC-PUC1.1.json`](T3.2-SC-PUC1.1.json). The type of data and header keys are described below with examples. 

## Route Guidance Tool 

### Kafka message headers (same for both PUC) 

| Header key | Example | Purpose |
|---|---|---|
| `project-id` | `iDriving` | Identifies the project; constant across all messages. |
| `use-case-id` | `UC1.1` | Identifies the use-case scope of the message. |
| `content-type` | `application/json` | Declares the payload format. |
| `message-type` | `rerouting` | Identifies the message schema / event type. |
| `producer-id` | `idriving` | Identifies the producing component instance. |
| `correlation-id` | `1ef9d1e0-91fd-47a2-bd96-3be46d69b4bf` | Correlates related messages across components for distributed tracing. |
| `message-timestamp` | `2026-01-20T15:23:59.882539+00:00` | Producer-side UTC timestamp of the message. |


### Body (version of PUC 2.2)

#### alert_info{}: 
| Header key | Example | Purpose |
|---|---|---|
| `type` | `"obstacle"` | Identifies the type of incident/alert (e.g. 'obstacle','congestion','accident' etc.) |
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


### Body (version of PUC 1.1)\

#### reroute_recom{}: rerouting recommendation 
| Header key | Example | Purpose |
|---|---|---|
| `generated alerts` | ` "Attention! Congestion in Kaiser-Franz-Josef-Kai. Delays expected. Take Kalvariengürtel and Bahnhofgürtel, instead."` | Identifies a general rerouting recommendation message (displayed on a VMS for all drivers). |

#### alert_info{}: information of the specific incident/alert 
| Header key | Example | Purpose |
|---|---|---|
| `type` | `"congestion"` | Identifies the type of incident/alert (e.g. 'obstacle','congestion','accident' etc.) |
| `notes` | `"Kaiser-Franz-Josef-Kai"` | Identifies the name of the street where the incident takes place. | 
| `startTime` | `23700` | Identifies the starting time of the incident (in seconds from 00:00 or the beginning of simulation). | 
| `duration` | `3600` | Identifies the duration of the incident (in seconds) for a preset simulation scenario. | 
| `coordinates` | `["12346678","12345678"] ` | Identifies the latitude and longitude of the incident location (if localized). |
| `link_closure` | `true` | Identifies if the incident road is closed for traffic or not. | 
| `OppDirClosure` | `false` | Identifies if the incident road is closed for traffic also in the opposite direction or not. | 
| `AdditionalClosureLinkIDs` | `["123458"]` | Identifies the IDs of network edges that are also affected by the incident (if any). |
| `veh_types` | `[“car”, “motorcycle”, “truck”]` | Identifies the types of vehicles that are affected by the incident. | 
| `level` | `“high”` | Identifies the intensity of the incident. | 
| `baseAltRoad` | `"Wickenburggasse"` | Identifies the default alternative for the specific road (if any). |
| `EmergencyVehicles_Locations` | `"Sporgasse"` | Identifies the location of emergency vehicles (firefighters or ambulances). |

#### ev_route{}: block for emergency vehicle routing 
| Header key | Example | Purpose |
|---|---|---|
| `veh_type` | `“emergency”` | Identifies the type of vehicle for which the route is intended (Emergency Vehicle, EV). | 
| `veh_id` | `“ev_0_23701_0”` | Identifies the vehicle ID of the specific EV in SUMO. |
| `start_edge` | `"Sporgasse"` | Indentifies the starting point of the EV trip. | 
| `destination_edge` | `"Wickenburggasse"` | Indentifies the destination point of the EV trip. | 
| `fastest_route` | `"('Sporgasse', 'Paulustorgasse', 'Maria-Theresia-Allee', 'Bergmanngasse', 'Humboldtstraße', 'Wickenburggasse')"` | Indentifies the fastest path for the EV to follow for this trip. | 

#### rerouted_trips: block with all alternative route suggestions 
| Header key | Example | Purpose |
|---|---|---|
| `vehicle type` | `“car”` | Identifies the type of vehicle for which the route is intended ("car". "truck", "motorbike") |
| `veh ID` | `"t1801"` | Identifies vehicle ID in SUMO. |
| `new_path` | `"('Körösistraße', 'Wickenburggasse', 'Jahngasse', 'Parkstraße')"` |  Indentifies the alternative path proposed for this trip and vehicle type. |
| `old_path` | `"('Körösistraße', 'Wickenburggasse', 'Jahngasse', 'Parkstraße')"` |  Indentifies the original path that needs to be adapted. |


## Signal Control Tool 

### Body 

#### signals{}:  
| Header key | Example | Purpose |
|---|---|---|
| `<n>` | `0` | Identifies the signalized intersection for which the signal plan is computed (n: internal ID) | 
| `signal_id` | `"redlight1"` | Indentifies a specific traffic stream (connection of upstream-downstream edges) of the intersection which is controlled by one traffic light (e.g. stream from North to South). | 
| `red` | `60` | Indentifies the total red time of the stream (in seconds) during one cycle. | 
| `green` | `30` | Indentifies the total green time of the stream (in seconds) during one cycle.  | 

