# Architecture Diagrams — Graphviz Quick Start

Use Graphviz `dot` to render the `.dot` diagram to PNG.

Windows (system install):
- "C:\\Program Files\\Graphviz\\bin\\dot.exe" -Tpng ClaudeCode_Plan/architecture.dot -o ClaudeCode_Plan/architecture.png

Cross‑platform (if `dot` is on PATH):
- dot -Tpng ClaudeCode_Plan/architecture.dot -o ClaudeCode_Plan/architecture.png

Tips
- Re‑render after edits to update the PNG.
- Use `-Tsvg` for SVG output.
- If `dot` isn’t found, install Graphviz or add it to PATH.
