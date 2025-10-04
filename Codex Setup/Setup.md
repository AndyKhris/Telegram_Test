# Codex Cloud Manual Setup Script

Copy everything in the block below into the Codex “Manual Setup script”. It installs Graphviz (system tool), creates a Python virtual environment in `.venv`, installs Python packages, and verifies the install.

```
set -euo pipefail
export DEBIAN_FRONTEND=noninteractive

# 1) OS packages: Graphviz + fonts + venv tooling
if command -v apt-get >/dev/null 2>&1; then
  apt-get update
  apt-get install -y --no-install-recommends graphviz fonts-dejavu-core python3-venv
  apt-get clean && rm -rf /var/lib/apt/lists/*
elif command -v apk >/dev/null 2>&1; then
  apk add --no-cache graphviz ttf-dejavu python3 py3-virtualenv
elif command -v yum >/dev/null 2>&1; then
  yum install -y graphviz fontconfig python3-venv || true
fi

# 2) Python virtualenv + dependencies
python3 -m venv .venv
. .venv/bin/activate
python -m pip install -U pip

# Install from requirements.txt if present; otherwise install the MVP stack
if [ -f requirements.txt ]; then
  pip install --no-cache-dir -r requirements.txt
else
  pip install --no-cache-dir \
    fastapi "uvicorn[standard]" httpx "python-telegram-bot>=21" \
    replicate fal-client cachetools redis "pydantic>=2" tenacity structlog \
    python-dotenv graphviz
fi

# 3) Sanity checks
dot -V
python - <<'PY'
import importlib

# Always-check core deps
mods=["fastapi","uvicorn","httpx","telegram","replicate",
      "cachetools","redis","pydantic","tenacity","structlog"]
for m in mods:
    importlib.import_module(m)

# fal client import name varies by version: try "fal" then fallback to "fal_client"
try:
    importlib.import_module("fal")
except ModuleNotFoundError:
    importlib.import_module("fal_client")

print("python deps ok")
PY
```

Notes
- Requires internet during setup for `apt-get`/`pip`.
- Keep Container Caching On so Graphviz and `.venv` are reused.
