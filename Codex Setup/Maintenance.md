# Codex Cloud Manual Maintenance Script

Copy everything in the block below into the Codex “Manual Maintenance script”. It activates the project’s virtualenv and verifies Graphviz is available. This runs before every task.

```
set -e

# Activate the virtual environment created in Setup
. .venv/bin/activate || { echo "Missing .venv; re-run Setup."; exit 1; }

# Ensure Graphviz CLI is available
command -v dot >/dev/null 2>&1 || { echo "Graphviz 'dot' not found"; exit 1; }

# Optional: quick checks (uncomment if desired)
# python -c "import sys; print(sys.executable)" >/dev/null
# dot -V >/dev/null

# Optional: render the diagram each run (commented out by default)
# dot -Tpng Codex_Plan/architecture.dot -o Codex_Plan/architecture.png >/dev/null 2>&1 || true
```

Notes
- No reinstall work here; it should be fast. If it fails, re-run Setup.
- In interactive terminals, you can also `source .venv/bin/activate` once per terminal session.

