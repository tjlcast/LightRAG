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


## Data schema
### chunk_result

[({"tjl": [{below}, {below}], ...}, {("t1", "t2"): [{below}]}), ...]

```
{
	"entity_name": "Scrooge",
	"entity_type": "person",
	"description": "Scrooge is a man who is skeptical and terrified by the appearance of a ghost, initially doubting his senses and trying to dismiss the apparition as a hallucination.",
	"source_id": "chunk-1aa50206d02ed89418f65f97c6441d1c",
	"file_path": "unknown_source",
	"timestamp": 1767838847
}
```

```
{
	"src_id": "Scrooge",
	"tgt_id": "Marley's Ghost",
	"weight": 1,
	"description": "Scrooge is terrified and incredulous upon seeing Marley's Ghost, engaging in a tense dialogue where he questions the ghost's reality and purpose.",
	"keywords": "fear, disbelief, confrontation",
	"source_id": "chunk-1aa50206d02ed89418f65f97c6441d1c",
	"file_path": "unknown_source",
	"timestamp": 1767838847
}
```


### merge_nodes_and_edges

