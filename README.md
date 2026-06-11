# iDriving — Data Formats

This repository supports the **iDriving prototype deliverables** by collecting and
demonstrating the **data formats exchanged between the project's tools**. It gives partners
a single, shared place to publish concrete JSON examples of the messages their components
produce, so the integration interfaces described in the deliverables can be illustrated with
real, linkable payloads instead of inline snippets.

## How it is organised

- One folder per component, named by its task ID (e.g.
`T4.1-Edge-Detection-Tools`). Each partner adds the JSON files for their component's
interfaces inside their own folder.
- `[_example/](_example)` — a generic, representative example plus the header, topic and
payload conventions every message should follow. Start here.

## For partners

1. Open your component's folder.
2. Add one JSON file per interface, named by its interface ID as it appears in the
  deliverable (e.g. `T4.1-01.json`).
3. Follow the envelope and conventions shown in `[_example/](_example)`.
4. The interface tables in the deliverable link back to these files.


| Task | Component                                           | Lead partner |
| ---- | --------------------------------------------------- | ------------ |
| T3.2 | Route Guidance & Signal Control Tools               | MBL          |
| T3.3 | Mobile & In-Vehicle Application                     | Simavi       |
| T3.4 | Aerial Surveillance UAV System                      | ACCELI       |
| T3.5 | Autonomous UAV Deployment for Area Coverage         | CERTH        |
| T4.1 | Edge Detection Tools                                | CERTH        |
| T4.2 | Behaviour Violation Detectors                       | TEKNIKER     |
| T4.3 | Pothole Detection and Severity Assessment Tool      | CERTH        |
| T4.4 | Weather Prediction & Real-Time Weather Alert System | DREVEN       |
| T4.5 | 3D-Smart Tool                                       | CERTH        |
| T5.1 | SCC Dynamic Monitoring Platform                     | UNI.EIFFEL   |
| T5.3 | SUMO & CARLA                                        | UNI.EIFFEL   |
| T5.4 | AI-Optimized Maintenance Through Digital Twin       | TEKNIKER     |
| T5.5 | Digital Twin-Based Control Centre                   | SIMAVI       |


