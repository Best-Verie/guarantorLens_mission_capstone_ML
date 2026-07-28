# Diagram source code (regenerate on a free online generator)

Paste each file's contents into a free code-based diagram generator and export PNG/SVG.

| File | Type | Where to paste |
|---|---|---|
| 01_ml_pipeline.mmd | Mermaid flowchart (**horizontal, LR**) | https://mermaid.live |
| 02_architecture.mmd | Mermaid flowchart (**horizontal, LR**) | https://mermaid.live |
| 02b_architecture_icons.mmd | Mermaid **architecture-beta** (with storage/server/database **icons**) | https://mermaid.live (needs Mermaid v11+) |
| 03_erd.mmd | Mermaid erDiagram | https://mermaid.live |
| 04_class_diagram.mmd | Mermaid classDiagram | https://mermaid.live |
| 05_gantt.mmd | Mermaid gantt | https://mermaid.live |
| 06_use_case.puml | PlantUML use-case | https://www.plantuml.com/plantuml or https://www.planttext.com |
| 06_use_case.mmd | Mermaid fallback (flowchart) | https://mermaid.live |
| 07_data_flow.mmd | Mermaid flowchart (DFD) | https://mermaid.live |
| 08_sequence.mmd | Mermaid sequenceDiagram | https://mermaid.live |

Notes:
- All are built from the real code (`app/models.py`, `app/scoring.py`), so fields/classes match the report text.
- **Orientation:** the pipeline and architecture use `flowchart LR` (horizontal, fits a portrait doc); data-flow and use-case are already horizontal (LR). Sequence, Gantt, ERD and class have their own layouts.
- **Icons:** the flowchart diagrams (01, 02, 07) no longer use the `fa:fa-...` shorthand. mermaid.live now ships **Font Awesome 7**, whose free set only carries *solid* glyphs, so Mermaid's generic `fa` class renders **blank boxes** (verified: icons collapse to ~4px with no glyph). The labels are self-explanatory without icons. If you still want icons, use `02b_architecture_icons.mmd`, which uses `architecture-beta` with built-in **native** icons (internet, cloud, database, disk, server) that DO render. For brand logos there, enable the **logos** icon pack on mermaid.live, then swap e.g. `(database)` -> `(logos:postgresql)`, `(server)` -> `(logos:fastapi)`, `(internet)` -> `(logos:react)`.
- **Gantt:** the dates are placeholders — replace them with your real schedule, then make Chapter 1's timeline text match.
- On mermaid.live you can export PNG/SVG from the Actions menu.
