# Prepare env

## Backend

```
python -m venv .venv
./.venv/Scripts/Activate.ps1
pip install -e .
pip install -e ".[api]"
pip install "lightrag-hku[api]"
```


## Frontend
Following [https://bun.sh/docs/installation](https://bun.sh/docs/installation) to install bun

```
bun install --frozen-lockfile
bun run build
```