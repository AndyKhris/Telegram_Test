# Architecture Diagrams — Graphviz Quick Start

Use Graphviz `dot` to render the `.dot` diagrams to PNG.

Windows (system install):
- Render main diagram: "C:\\Program Files\\Graphviz\\bin\\dot.exe" -Tpng Codex_Plan/architecture.dot -o Codex_Plan/architecture.png
- Render v3 diagram:   "C:\\Program Files\\Graphviz\\bin\\dot.exe" -Tpng Codex_Plan/v3/architecture_v3.dot -o Codex_Plan/v3/architecture_v3.png

Cross‑platform (if `dot` is on PATH):
- dot -Tpng Codex_Plan/architecture.dot -o Codex_Plan/architecture.png
- dot -Tpng Codex_Plan/v3/architecture_v3.dot -o Codex_Plan/v3/architecture_v3.png

Tips
- Re‑render after editing the `.dot` files to update PNGs.
- For SVG export, replace `-Tpng` with `-Tsvg`.
- If `dot` isn’t found, install Graphviz or add it to PATH.
