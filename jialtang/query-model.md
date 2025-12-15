在LightRAG中，查询模式（Query Modes）决定了系统如何检索和处理信息。根据我搜索到的信息，主要有以下几种查询模式：

## LightRAG 查询模式详解

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

你可以通过在QueryParam中设置mode参数来选择不同的查询模式，例如：
```python
rag.query("你的问题", param=QueryParam(mode="hybrid"))
```