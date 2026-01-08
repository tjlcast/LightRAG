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

### 从大模型回复中提取ER
[_process_extraction_result](./lightrag/operate.py#L2860)
Key Input: final_result(大模型的回复字符串)

### chunk_result
[_process_extract_entities](./lightrag/lightrag.py#L1897)
Key Input: 当前时间窗口下的所有文档所有文档下切分的所有chunks
Key Output: 所有文档所有chunk下的er

```
[({"tjl": [{below}, {below}], ...}, {("t1", "t2"): [{below}]}), ...]
```

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
[merge_nodes_and_edges](./lightrag/lightrag.py#L1980)
Key Input: 当前时间窗口下的所有文档 - 拆出的所有chunk - 所有ER


#### 第一阶段
[_merge_nodes_then_upsert](./lightrag/operate.py#L2499)
Key Input: Each entity && The entities with same name.
描述： 主要是根据已经插入的与当前插入进行合并， 主要考虑的有(type\ source_id\ description\ file_path).

+ 插入entity_chunk
[entity_chunks_storage.upsert](./lightrag/operate.py#L1637)
```
{
    'Jacob Marley': {
        "chunk_ids": ['chunk-1aa50206d02ed89418f65f97c6441d1c', 'chunk-359187e7571bda4b0b08b2e0b6581e52', 'chunk-cb661c37436355ccec2769b1d0350c5f'],
        "count": 3,
    }
}
```
+ 插入graph
[knowledge_graph_inst.upsert_node](./lightrag/operate.py#L1844)
```
{
	"entity_id": "Jacob Marley",
	"entity_type": "person",
	"description": "Jacob Marley was Scrooge's partner in life and is now a ghost who returns to deliver a message, bound by a chain symbolizing his earthly burdens.<SEP>Jacob Marley is the deceased former business partner of Scrooge, whose spirit appears as a ghost burdened by remorse.<SEP>Jacob Marley is the ghost of Scrooge's deceased business partner who appears to warn Scrooge.",
	"source_id": "chunk-1aa50206d02ed89418f65f97c6441d1c<SEP>chunk-359187e7571bda4b0b08b2e0b6581e52<SEP>chunk-cb661c37436355ccec2769b1d0350c5f",
	"file_path": "unknown_source",
	"created_at": 1767842906,
	"truncate": ""
}
``` 
+ 插入vdb
[entity_vdb.upsert(payload)](./lightrag/operate.py#L1862)
```
{
	"ent-f4f536d4d5e5c1a54bcc42f8aab1a293": {
		"entity_name": "Jacob Marley",
		"entity_type": "person",
		"content": "Jacob Marley\nJacob Marley was Scrooge's partner in life and is now a ghost who returns to deliver a message, bound by a chain symbolizing his earthly burdens.<SEP>Jacob Marley is the deceased former business partner of Scrooge, whose spirit appears as a ghost burdened by remorse.<SEP>Jacob Marley is the ghost of Scrooge's deceased business partner who appears to warn Scrooge.",
		"source_id": "chunk-1aa50206d02ed89418f65f97c6441d1c<SEP>chunk-359187e7571bda4b0b08b2e0b6581e52<SEP>chunk-cb661c37436355ccec2769b1d0350c5f",
		"file_path": "unknown_source"
	}
}
```
说明: 	entity_vdb_id = compute_mdhash_id(str(entity_name), prefix="ent-")
    	entity_content = f"{entity_name}\n{description}"


#### 第二阶段
[_merge_edges_then_upsert](./lightrag/operate.py#L2606)
Key Input: Each relation && All relations with same name.
描述： 主要是根据已经插入的与当前插入进行合并， 主要考虑的有(source_id\ weight\ keywords\ description\ file_path).

+ 插入 entity_chunks_storage
[entity_chunks_storage.upsert](./lightrag/operate.py#L1637)
``` json
{
    'The Clerk<SEP>The Fire': {
        "chunk_ids": 'chunk-5dac41b3f9eeaf794f0147400b1718cd',
        "count": 1,
    }
}
```

+ 合并`source_id`
参考keep和FIFO

+ 合并`weight`和`keywords`
weight使用求和
keywords使用并集

+ 合并`description`
使用LLM进行逐个压缩，顺序根据时间

+ 插入`Graph`
判断关联节点是否存在，不存在则插入
	[插入graph](./lightrag/operate.py#L2184)
	[插入entity_chunk](./lightrag/operate.py#L2190)
	[插入vdb](./lightrag/operate.py#L2212)
	```
	entity_content = f"{need_insert_id}\n{description}"
    vdb_data = {
        entity_vdb_id: {
            "content": entity_content,
            "entity_name": need_insert_id,
            "source_id": source_id,
            "entity_type": "UNKNOWN",
            "file_path": file_path,
        }
    }
	```
存在则进行更新节点
	从 entity_chunks_storage 和 knowledge_graph_inst 中获取 source_id, 与relation中的source_id合并
	[合并好后的 source_id 更新 entity_chunk_storage](./lightrag/operate.py#L2266)
	[使用apply_source_ids_limit进行entity的source_id限制](./lightrag/operate.py#L2280)
	[使用限制后的source_id更新knowledge_graph_inst.upsert_node](./lightrag/operate.py#L2296)
	[更新 entity_vdb ](./lightrag/operate.py#L2212)
	```
	entity_content = f"{need_insert_id}\n{description}"
	entity_vdb_id: {
        "content": entity_content,
        "entity_name": need_insert_id,
        "source_id": limited_source_id_str,
        "entity_type": existing_node.get("entity_type", "UNKNOWN"),
        "file_path": existing_node.get(
            "file_path", "unknown_source"
        ),
    }
	```
[更新 graph](./lightrag/operate.py#L2335)
[更新 relationships_vdb 删除旧的插入新的](./lightrag/operate.py#L2388)
```
rel_content = f"{keywords}\t{src_id}\n{tgt_id}\n{description}"
vdb_data = {
    rel_vdb_id: {
        "src_id": src_id,
        "tgt_id": tgt_id,
        "source_id": source_id,
        "content": rel_content,
        "keywords": keywords,
        "description": description,
        "weight": weight,
        "file_path": file_path,
    }
}
```


#### 第三阶段
Update full_entities and full_relations storage

[统计 第一阶段 和 第二阶段 操作过的实体数据 final_entity_names](./lightrag/operate.py#L2710)
[统计 第二阶段 操作过的关系数据 final_relation_pairs](./lightrag/operate.py#L2720)
[插入 full_entities_storage](./lightrag/operate.py#L2730)
[插入 full_relations_storage](./lightrag/operate.py#L2740)