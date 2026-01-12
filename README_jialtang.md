## Prepare env

### Backend

```
python -m venv .venv
./.venv/Scripts/Activate.ps1
pip install -e .
pip install -e ".[api]"
pip install "lightrag-hku[api]"
```


### Frontend
Following [https://bun.sh/docs/installation](https://bun.sh/docs/installation) to install bun

```
bun install --frozen-lockfile
bun run build
```


## Build Data schema

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


## Query

### naive
response_type: str = "Multiple Paragraphs"
"""Defines the response format. Examples: 'Multiple Paragraphs', 'Single Paragraph', 'Bullet Points'."""
```
query_result = await naive_query(
    query.strip(),
    self.chunks_vdb,
    param,
    global_config,
    hashing_kv=self.llm_response_cache,
    system_prompt=system_prompt,
)
```
[查询vdb](./lightrag/operate.py#L3390)
[处理vdb结果 process_chunks_unified](./lightrag/operate.py#L4847)
	rerank
	token调整
	按照chunk数组下标赋予: id="DC{idx}"
[generate_reference_list_from_chunks](./lightrag/operate.py#L4857)
	返回多个文档片段时，能够按照引用源（文件）的重要程度（以出现频率衡量）进行排序，并为每个片段分配相应的引用编号，方便在最终答案中进行引用标注
	```
	    reference_list, processed_chunks_with_ref_ids = generate_reference_list_from_chunks(
        processed_chunks
    )
	```
	```
	reference_list[0] = {'reference_id': '1', 'file_path': './book.txt'}
	processed_chunks_with_ref_ids[0] = {'content': 'abc...', 'created_at': 1767861587, 'file_path': './book.txt', 'source_type': 'vector', 'chunk_id': 'chunk-9e3921da66da5d761ab73cd849af6c43', 'id': 'DC1', 'reference_id': '1'}
	```
[构造统一结构体](./lightrag/operate.py#L4864)
[构建LLM提示词](./lightrag/operate.py#L4904)
[查询query缓存](./lightrag/operate.py#L4937)
[没有命中缓存进行LLM对话](./lightrag/operate.py#L4947)
[获取结构化结果](./lightrag/lightrag.py#L2445)
```
if llm_response.get("is_streaming"):
    return llm_response.get("response_iterator")
else:
    return llm_response.get("content", "")
```

### local

[get_keywords_from_query提取高层和底层关键字](./lightrag/operate.py#L3065)
	[一种cache(llm_response_cache)的使用](./lightrag/operate.py#L3272)
	```
	key format: {mode}:{cache_type}:{hash}, 其中has中包含query和type
	```
	[没有命中缓存， 使用例子和query在LLM进行提取](./lightrag/operate.py#L3313)
	[保存到cache](./lightrag/operate.py#L3349)
	```
	await save_to_cache(
        hashing_kv,
        CacheData(
            args_hash=args_hash,
            content=json.dumps(cache_data),
            prompt=text,
            mode=param.mode,
            cache_type="keywords",
            queryparam=queryparam_dict,
        ),
    )
	```
[_build_query_context](./lightrag/operate.py#L3088)
	[_perform_kg_search](./lightrag/operate.py#L4069)
	这个函数根据ll_keywords和hl_keywords进行召回
	```
	return {
        "final_entities": final_entities,
        "final_relations": final_relations,
        "vector_chunks": vector_chunks,
        "chunk_tracking": chunk_tracking,
        "query_embedding": query_embedding,
    }
	```
		[query向量化](./lightrag/operate.py#L3459)
		[_get_node_data](./lightrag/operate.py#L3470)
			输入ll_keywords拼接的字符串
			[entities_vdb.query](./lightrag/operate.py#L4177)
			获取最相似的实体名字
			input: 'Plot, Characters, Setting, Conflict, Moral' 
			output: 
			```
			{
				"__id__": "ent-c00a35ddba52071d452455238b12b616",
				"__created_at__": 1767861608,
				"entity_name": "Valentine",
				"content": "Valentine\nValentine is a storyxxx.",
				"source_id": "chunk-ae57de73be0e0c37f9fb8a1226c5fb18",
				"file_path": "./book.txt",
				"__metrics__": 0.5677251443797806,
				"id": "ent-c00a35ddba52071d452455238b12b616",
				"distance": 0.5677251443797806,
				"created_at": 1767861608
			}
			```
			[get_nodes_batch|node_degrees_batch](./lightrag/operate.py#L4186)
			[_find_most_related_edges_from_entities](./lightrag/operate.py#L4209)
			与某些特定实体相关的所有关系时使用，返回的边是按重要性（度数和权重）排序的，使得最相关的边排在前面
		[_get_edge_data](./lightrag/operate.py#L3478)
		[Round-robin merge entities](./lightrag/operate.py#L3521)
		[Round-robin merge relations](./lightrag/operate.py#L3542)

### global

### hybrid




1. **naive（朴素模式）**
   - 这是最基本的检索方式，只使用向量检索文本块，不涉及知识图谱
   - 类似传统的RAG方法，直接在文本块中进行相似度搜索

2. **local（局部模式）**
   - 专注于实体检索
   - 根据查询关键词找到相关实体，然后获取这些实体的详细信息和邻接关系
   - 更适合需要深入了解特定实体的查询

3. **global（全局模式）**
   - 关注关系检索
   - 从更高层次理解查询中的概念，寻找实体间的潜在关系
   - 适用于需要理解全局关系和宏观视角的查询

4. **hybrid（混合模式）**
   - 结合了local和global两种模式
   - 同时利用实体检索和关系检索的优势
   - 提供更全面的检索结果

5. **mix（混合模式）**
   - 集成了知识图谱和向量检索
   - 综合使用local、global和naive三种方式的结果
   - 是目前最全面的检索模式

6. **bypass（绕过模式）**
   - 跳过检索阶段，直接将对话历史和当前问题发送给大语言模型
   - 不使用知识图谱或向量检索，完全依赖LLM自身的知识
   - 适用于不需要检索外部知识的通用问答


每种模式都有其适用场景：
- 当你需要简单的文本匹配时，可以使用[naive](..\lightrag\api\routers\ollama_api.py#L18-L18)模式
- 当你想深入了解特定实体相关信息时，使用[local](..\lightrag\api\routers\ollama_api.py#L19-L19)模式
- 当你需要理解全局关系和宏观概念时，使用`global`模式
- 当你希望得到更全面的结果时，使用[hybrid](..\lightrag\api\routers\ollama_api.py#L21-L21)或[mix](..\lightrag\api\routers\ollama_api.py#L22-L22)模式
- 当你只需要LLM自身知识回答问题时，使用[bypass](..\lightrag\api\routers\ollama_api.py#L23-L23)模式

## API

### create entity
[create_entity的api](./lightrag/api/routers/graph_routes.py#L446)
[rag.acreate_entity](./lightrag/lightrag.py#L3862)
[async def acreate_entity(](./lightrag/utils_graph.py#L898-L899)
	[existing_node = await chunk_entity_relation_graph.has_node(entity_name)](./lightrag/utils_graph.py#L932)
	+ 检查是否有同名的实体
	[upsert_node 写入graph中](./lightrag/utils_graph.py#L947)
	[upsert_node 写入vdb中](./lightrag/utils_graph.py#L971)
	[如果当前配置了entity_chunks_storage， 并且写入中包含source_id](./lightrag/utils_graph.py#L979)

### edit entity
[edit entity 的 api](./lightrag/api/routers/graph_routes.py#L220)
[rag aedit_entity(](./lightrag/lightrag.py#L3779-L3785)
[utils_graph aedit_entity(](./lightrag/utils_graph.py#L515-L525)
	参数第一层的entity_name标识修改源的实体(src_entity)， 第二层的(updated_data.entity_name)标识修改目标的实体(tgt_entity)
	判断是否要修改名字(参数 is_renaming)， 以及 tgt_entity 是否存在
	情节1: 不修改名字
		[utils_graph.py _edit_entity_impl 直接修改src_entity的属性](./lightrag/utils_graph.py#L697)
			tips: mv params into old entity
			[查询存在的实体](./lightrag/utils_graph.py#L288)
			[合并 参数属性 与 查询的实体](./lightrag/utils_graph.py#L297)
			[插入new_entity_name 到 graph 存储](./lightrag/utils_graph.py#L308)
			[获取旧节点关联的边， 并创建新的边](./lightrag/utils_graph.py#L312)
			[删除旧的节点](./lightrag/utils_graph.py#L334)
			[删除旧的节点 的 vdb](./lightrag/utils_graph.py#L337)
			[插入新边的vdb](./lightrag/utils_graph.py#L367)
			[插入新店的vdb](./lightrag/utils_graph.py#L390)
			最后考虑后面的source_id的合并
	情节2: 修改名字，但tgt存在
		[target_exists 判断 tgt_entity 是否存在](./lightrag/utils_graph.py#L601)
		[utils_graph.py 修改 src_entity 的属性](./lightrag/utils_graph.py#L625)
		[entity_name 合并两个节点](./lightrag/utils_graph.py#L644)
	情节3: 修改名字，tgt不存在
		同情节1
		[utils_graph.py](./lightrag/utils_graph.py#L697)



### create relation
[create_relation api](./lightrag/api/routers/graph_routes.py#L519)
[rag.acreate_relation](./lightrag/lightrag.py#L3892)
[utils_graph acreate_relation](./lightrag/utils_graph.py#L1012)
[确认 src_entity 和 tgt_entity 都存在](./lightrag/utils_graph.py#L1050)
构建如下数据结构
```
edge_data = {
    "description": relation_data.get("description", ""),
    "keywords": relation_data.get("keywords", ""),
    "source_id": relation_data.get("source_id", "manual_creation"),
    "weight": float(relation_data.get("weight", 1.0)),
    "file_path": relation_data.get("file_path", "manual_creation"),
    "created_at": int(time.time()),
}
```
[插入 vdb](./lightrag/utils_graph.py#L1112)
content = f"{keywords}\t{source_entity}\n{target_entity}\n{description}"
[把参数中的source_id关联edge与chunks](./lightrag/utils_graph.py#L1126)

### edit relation
[edit relation api](./lightrag/api/routers/graph_routes.py#L411)
[rag.aedit_relation](./lightrag/lightrag.py#L3826)
[判断并获取 graph 中的 relation](./lightrag/utils_graph.py#L760)
[删除 vdb 中的 relation](./lightrag/utils_graph.py#L769)
[合并参数并插入 graph 中](./lightrag/utils_graph.py#L776)
[重新拼接并插入 vdb 中](./lightrag/utils_graph.py#L808)
// Update relation_chunks_storage in two scenarios:
//   - source_id has changed (edit scenario)
//   - relation_chunks_storage has no existing data (migration/initialization scenario)
[更新 relation_chunks_storage](./lightrag/utils_graph.py#L861)

