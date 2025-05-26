# GraphRAG从0到1

## 目录
- [index工作流](#index工作流) 
    - [一、创建基本文本单元](#一创建基本文本单元)
    - [二、提取图谱](#二提取图谱)
    - [三、完善图谱](#三完善图谱)
    - [四、提取协变量](#四提取协变量)
    - [五、创建社区](#五创建社区)
    - [六、创建最终文本单元](#六创建最终文本单元)
    - [七、生成社区报告](#七生成社区报告)
    - [八、向量化](#八向量化)


## index工作流
标准模式按照如下工作流执行：
```python
case IndexingMethod.Standard:
    return [
        "create_base_text_units",
        "create_final_documents",
        "extract_graph",
        "finalize_graph",
        *(["extract_covariates"] if config.extract_claims.enabled else []),
        "create_communities",
        "create_final_text_units",
        "create_community_reports",
        "generate_text_embeddings",
        *(update_workflows if is_update_run else []),
    ]
```

## 一、创建基本文本单元
核心源码：
```python
documents = await load_table_from_storage("documents", context.storage)
chunks = config.chunks

output = create_base_text_units(
    documents,
    ...
)
#  分块
aggregated = aggregated.apply(lambda row: chunker(row), axis=1)
```
#### ducuments.parquet
documents: pd.DataFrame
columns:
- id: uuid
- text: txt文件文本内容
- title: txt文件名
- creation_date:创建时间

#### text_units.parquet
output:  pd.DataFrame
columns:
- text: 文本块文本
- document_ids: 文本块对应的txt文件id
- n_tokens: 文本块的token数量

分块策略：
size决定单个文本块长度, overlap决定块之间重叠的长度
```text
# settings.yaml中配置
chunks:
- size: 1200
- overlap: 100
- group_by_columns: [id]
```




## 二、创建最终文档

将分散的文本单元(text units)按照它们所属的文档(document)进行重组， 信息重新拼接到text_units.parquet中
核心源码：
```python
output = create_final_documents(documents, text_units)
```
输入：
- documents: ducuments.parquet
- text_units: text_units.parquet

输出： documents.parquet
- columns:
    - id
    - human_readable_id
    - text
    - n_tokens
    - document_ids
    - entity_ids
    - relationship_ids
    - covariate_ids

## 三、提取图谱

### 1. 提取实体和关系
核心函数为：
```python
async def _process_document(
        self, text: str, prompt_variables: dict[str, str]
    ) -> str:
        response = await self._model.achat(
            self._extraction_prompt.format(**{
                **prompt_variables,
                self._input_text_key: text,
            }),
        )
        results = response.output.content or ""

        if self._max_gleanings > 0:
            for i in range(self._max_gleanings):
                response = await self._model.achat(
                    CONTINUE_PROMPT,
                    name=f"extract-continuation-{i}",
                    history=response.history,
                )
                results += response.output.content or ""

                # if this is the final glean, don't bother updating the continuation flag
                if i >= self._max_gleanings - 1:
                    break

                response = await self._model.achat(
                    LOOP_PROMPT,
                    name=f"extract-loopcheck-{i}",
                    history=response.history,
                )
                if response.output.content != "Y":
                    break

        return results
```
输入为: text_units.parquet
- pd.DataFrame
    - columns包括: 
        - id: 文本单元的id
        - text: **文本单元的文本**
        - document_ids: 文本单元所属文档的id
        - n_tokens: 文本单元的token数

输出为: `entities.parquet` 和 `relationship.parquet`
- entities.parquet
    - columns包括: 
        - title: 实体名称
        - type: 实体类型
        - text_units_id:: 引用的文本单元id
        - frequency: 实体出现的次数
        - description: 描述 `[list]`

- relationship.parquet
    - columns包括:
        - source: 关系的起始实体
        - target: 关系的目标实体
        - text_units_id: 引用的文本单元id
        - weight: 关系权重
        - description: 描述 `[list]`
        


具体工作原理如下：

1. 首先，系统使用语言模型（如GPT）对文档进行一次初始提取，获取实体和它们之间的关系。

2. 如果`max_gleanings`设置为大于0的值，系统会继续进行额外的提取轮次，最多进行`max_gleanings`次额外提取：
   - 系统会向语言模型发送`CONTINUE_PROMPT`提示，要求它继续提取可能遗漏的实体和关系
   - 然后系统会向语言模型发送`LOOP_PROMPT`提示，询问是否还有更多实体和关系需要提取
   - 如果语言模型回答"Y"，则继续下一轮提取；如果回答"N"或已达到`max_gleanings`次数，则停止提取

3. 默认情况下，`max_gleanings`的值为1，表示除了初次提取外，最多再进行一次额外提取。

### 2. 描述总结

核心源码
```python
async def _summarize_descriptions(
        self, id: str | tuple[str, str], descriptions: list[str]
    ) -> str:
    """Summarize descriptions into a single description."""
    sorted_id = sorted(id) if isinstance(id, list) else id

    # Safety check, should always be a list
    if not isinstance(descriptions, list):
        descriptions = [descriptions]

    # Sort description lists
    if len(descriptions) > 1:
        descriptions = sorted(descriptions)

    # Iterate over descriptions, adding all until the max input tokens is reached
    usable_tokens = self._max_input_tokens - num_tokens_from_string(
        self._summarization_prompt
    )
    descriptions_collected = []
    result = ""

    for i, description in enumerate(descriptions):
        usable_tokens -= num_tokens_from_string(description)
        descriptions_collected.append(description)

        # If buffer is full, or all descriptions have been added, summarize
        if (usable_tokens < 0 and len(descriptions_collected) > 1) or (
            i == len(descriptions) - 1
        ):
            # Calculate result (final or partial)
            result = await self._summarize_descriptions_with_llm(
                sorted_id, descriptions_collected
            )

            # If we go for another loop, reset values to new
            if i != len(descriptions) - 1:
                descriptions_collected = [result]
                usable_tokens = (
                    self._max_input_tokens
                    - num_tokens_from_string(self._summarization_prompt)
                    - num_tokens_from_string(result)
                )

    return result
```
算法流程：
```text
[desc1, desc2, desc3, desc4, desc5] 
    ↓ (token限制，处理前3个)
summary1 = summarize([desc1, desc2, desc3])
    ↓ (继续处理)
[summary1, desc4, desc5]
    ↓ (最终总结)
final_result = summarize([summary1, desc4, desc5])
```

输入：`entities.parquet` 和 `relationship.parquet`
输出为：`entity_summaries`和 `relationship_summaries`，合并到输入中。即将原来的`description`列表替换为`summarized_description`

**config.snapshots.raw_graph**决定`raw_entities`和`raw_relationships`是否保存


## 四、生成图谱
`finalize_graph`具体计算了以下列以及计算方法：

### 📊 实体表 (Entities) 新增/计算的列

#### 1. **degree (度数)**
该实体作为source或target出现在多少条关系中
```python
# 计算方法
graph = create_graph(relationships, edge_attr=["weight"])
degrees = compute_degree(graph)  # 使用NetworkX的graph.degree

# 具体计算
degree = len([edge for edge in graph.edges if node in edge])
# 即：该实体作为source或target出现在多少条关系中

# 缺失值处理
final_entities["degree"] = final_entities["degree"].fillna(0).astype(int)
```

#### 2. **x, y (坐标位置)**
```python
# 计算方法
layout = layout_graph(graph, callbacks, layout_enabled, embeddings=graph_embeddings)

# 两种计算方式：
# 方式1：零布局 (layout_enabled=False)
x, y = 0, 0  # 所有节点都在原点

# 方式2：UMAP布局 (layout_enabled=True)
# 基于图结构和可选的嵌入向量计算2D坐标
positions = umap.fit_transform(node_features)
x, y = positions[node_index]
```
**参数config.umap.enabled决定是否使用UMAP布局**

### 📈 关系表 (Relationships) 新增/计算的列

#### 1. **combined_degree (组合度数)**
```python
# 计算方法
def compute_edge_combined_degree(edge_df, node_degree_df, ...):
    # 步骤1：获取源节点度数
    source_degrees = edge_df.merge(
        node_degree_df.rename(columns={"title": "source", "degree": "source_degree"}),
        on="source", how="left"
    )
    
    # 步骤2：获取目标节点度数  
    target_degrees = source_degrees.merge(
        node_degree_df.rename(columns={"title": "target", "degree": "target_degree"}),
        on="target", how="left"
    )
    
    # 步骤3：计算组合度数
    combined_degree = target_degrees["source_degree"] + target_degrees["target_degree"]
    
    return combined_degree

# 具体含义：关系两端节点的度数之和
# 例：张三(度数5) ←→ 李四(度数3) → combined_degree = 8
```

### 📋 最终输出列

#### 实体表最终列 (ENTITIES_FINAL_COLUMNS)：
```python
[
    "id",                    # UUID标识符
    "human_readable_id",     # 可读整数ID  
    "title",                 # 实体名称 (原有)
    "type",                  # 实体类型 (原有)
    "description",           # 实体描述 (原有)
    "text_unit_ids",         # 文本单元ID (原有)
    "degree",                # 度数 (新计算)
    "x",                     # X坐标 (新计算)
    "y"                      # Y坐标 (新计算)
]
```

#### 关系表最终列 (RELATIONSHIPS_FINAL_COLUMNS)：
```python
[
    "id",                    # UUID标识符
    "human_readable_id",     # 可读整数ID
    "source",                # 源实体 (原有)
    "target",                # 目标实体 (原有)  
    "description",           # 关系描述 (原有)
    "text_unit_ids",         # 文本单元ID (原有)
    "weight",                # 关系权重 (原有)
    "combined_degree"        # 组合度数 (新计算)
]
```

### 💡 计算意义总结

| 列名 | 计算方法 | 业务意义 |
|------|----------|----------|
| **degree** | 统计连接边数 | 实体重要性指标 |
| **x, y** | 图布局算法 | 可视化坐标 |
| **combined_degree** | 两端度数相加 | 关系重要性指标 |

这些计算为原始的实体-关系数据增加了**拓扑特征**、**空间信息**和**唯一标识**，使数据具备了用于检索、排序、可视化和分析的完整属性。


## 五、创建社区

**将知识图谱中的实体和关系数据转换为分层的社区结构**，通过图聚类算法自动发现实体之间的群体关系，并构建具有层次结构的社区数据。

### 输入参数

```python
def create_communities(
    entities: pd.DataFrame,        # 实体数据表
    relationships: pd.DataFrame,   # 关系数据表  
    max_cluster_size: int,        # 最大聚类大小
    use_lcc: bool,               # 是否使用最大连通分量
    seed: int | None = None,     # 随机种子（可选）
) -> pd.DataFrame:
```

#### 输入数据结构：

**entities DataFrame** - 实体表：
- `id`: 实体唯一标识符
- `title`: 实体名称/标题
- 其他实体属性...

**relationships DataFrame** - 关系表：
- `id`: 关系唯一标识符
- `source`: 源实体名称
- `target`: 目标实体名称
- `weight`: 关系权重
- `text_unit_ids`: 相关文本单元ID列表

### 输出

函数返回一个包含社区信息的`pd.DataFrame`，具体列结构由`COMMUNITIES_FINAL_COLUMNS`定义：

```python
COMMUNITIES_FINAL_COLUMNS = [
    "id",                    # 社区唯一UUID
    "human_readable_id",     # 人类可读的短ID（等于community）
    "community",             # Leiden算法生成的社区ID
    "level",                 # 社区在层次结构中的深度
    "parent",                # 父社区ID
    "children",              # 子社区ID列表
    "title",                 # 社区友好名称
    "entity_ids",            # 社区成员实体ID列表
    "relationship_ids",      # 社区内部关系ID列表
    "text_unit_ids",         # 社区相关文本单元ID列表
    "period",                # 数据摄取日期（ISO8601格式）
    "size",                  # 社区大小（实体数量）
]
```

#### 输出数据示例：

```python
{
    "id": "uuid-12345",
    "human_readable_id": 1,
    "community": 1,
    "level": 0,
    "parent": -1,
    "children": [],
    "title": "Community 1",
    "entity_ids": ["E1", "E2", "E3"],
    "relationship_ids": ["R1", "R2", "R3"],
    "text_unit_ids": ["T1", "T2", "T3"],
    "period": "2024-01-15",
    "size": 3
}
```

#### 核心功能特点

1. **分层社区检测**: 使用Leiden算法生成多层次的社区结构
2. **关系约束**: 只保留源节点和目标节点都在同一社区内的关系
3. **完整性保证**: 每个社区包含完整的实体、关系和文本单元信息
4. **层次结构维护**: 保持父子关系，支持树形结构遍历
5. **增量更新支持**: 包含时间戳和大小信息，便于后续增量更新

## 六、创建最终文本单元
创建最终文本单元表，将原始文本单元表与实体和关系表进行关联，生成包含实体和关系信息的文本单元表。


`create_final_text_units` 函数的输入输出结构：
### 输入结构

#### 1. `text_units: pd.DataFrame` (原始文本单元表)
```python
# 输入列结构（至少包含以下列）：
{
    "id": str,           # 文本单元唯一标识符
    "text": str,         # 文本内容
    "document_ids": list[str],  # 所属文档ID列表
    "n_tokens": int,     # 令牌数量
    # 可能还有其他列，但函数只使用上述4列
}
```

#### 2. `final_entities: pd.DataFrame` (最终实体表)
```python
# 输入列结构：
{
    "id": str,                    # 实体ID
    "text_unit_ids": list[str],   # 该实体出现的文本单元ID列表
    # 其他实体相关列...
}
```

#### 3. `final_relationships: pd.DataFrame` (最终关系表)
```python
# 输入列结构：
{
    "id": str,                    # 关系ID
    "text_unit_ids": list[str],   # 该关系出现的文本单元ID列表
    # 其他关系相关列...
}
```

### 输出结构

#### `pd.DataFrame` (最终文本单元表)
```python
# 输出列结构（严格按照 TEXT_UNITS_FINAL_COLUMNS 定义）：
{
    "id": str,                      # 文本单元ID
    "human_readable_id": int,       # 人类可读ID（从1开始的序号）
    "text": str,                    # 文本内容
    "n_tokens": int,                # 令牌数量
    "document_ids": list[str],      # 所属文档ID列表
    "entity_ids": list[str],        # 关联的实体ID列表
    "relationship_ids": list[str],  # 关联的关系ID列表
    "covariate_ids": list[str],     # 关联的协变量ID列表
}
```

## 七、生成社区报告

### 核心函数 `create_community_reports`

#### 1. 实体展平:将社区中所有实体展平，每个实体单独一行
```python
nodes = explode_communities(communities, entities)
```

**explode_communities 函数详解：**
```python
def explode_communities(communities: pd.DataFrame, entities: pd.DataFrame) -> pd.DataFrame:
    # 将communities中的entity_ids列表展开，每个entity_id一行
    community_join = communities.explode("entity_ids").loc[:, ["community", "level", "entity_ids"]]
    
    # 将实体表与社区表按entity_id关联
    nodes = entities.merge(community_join, left_on="id", right_on="entity_ids", how="left")
    
    # 过滤掉不属于任何社区的节点 (community != -1)
    return nodes.loc[nodes.loc[:, COMMUNITY_ID] != -1]
```


#### 2. 构建上下文（重要！！！）
##### 🔍 **层级上下文 vs 本地上下文详解**

###### 📍 **本地上下文 (Local Context)**

**定义**：本地上下文是指直接从图数据中提取的原始信息，包含社区内的实体、关系和声明的详细信息。

**构建过程**：
```python
def build_local_context(nodes, edges, claims, callbacks, max_context_tokens=16_000):
    """为报告生成准备社区数据"""
    # 1. 获取所有层级
    levels = get_levels(nodes, schemas.COMMUNITY_LEVEL)
    
    # 2. 为每个层级构建本地上下文
    for level in levels:
        communities_at_level_df = _prepare_reports_at_level(
            nodes, edges, claims, level, max_context_tokens
        )
```

**包含内容**：
- **实体信息**：节点的标题、描述、度数等
- **关系信息**：边的源、目标、描述、权重等  
- **声明信息**：与实体相关的声明和证据
- **原始图结构**：直接从知识图谱中提取的结构化数据

**特点**：
- ✅ 信息最详细、最完整
- ❌ 可能超出token限制
- 🎯 适用于小规模社区

###### 📍 **层级上下文 (Level Context)**

**定义**：层级上下文是一种智能的上下文管理策略，当本地上下文超出token限制时，用子社区的报告来替代详细的本地信息。

**构建过程**：
```python
def build_level_context(report_df, community_hierarchy_df, local_context_df, level, max_context_tokens):
    """为给定层级的每个社区准备上下文
    
    对于每个社区：
    - 检查本地上下文是否在限制内，如果是则使用本地上下文
    - 如果本地上下文超出限制，则迭代地用子社区报告替换本地上下文，从最大的子社区开始
    """
```

**智能替换策略**：

1. **有效上下文检查**：
```python
# 过滤有效和无效的上下文
valid_context_df = level_context_df[~level_context_df[schemas.CONTEXT_EXCEED_FLAG]]
invalid_context_df = level_context_df[level_context_df[schemas.CONTEXT_EXCEED_FLAG]]
```

2. **子社区报告替换**：
```python
# 对于每个无效上下文，尝试用子社区报告替换
sub_context_df = _get_subcontext_df(level + 1, report_df, local_context_df)
community_df = _get_community_df(level, invalid_context_df, sub_context_df, 
                                community_hierarchy_df, max_context_tokens)
```

3. **混合上下文构建**：
```python
# 构建混合上下文，优先使用报告，必要时保留本地上下文
community_df[schemas.CONTEXT_STRING] = community_df[schemas.ALL_CONTEXT].apply(
    lambda x: build_mixed_context(x, max_context_tokens)
)
```

##### 🔄 **两者的关系和转换**

###### **层级化处理流程**：

```
📊 本地上下文 (详细原始数据)
    ↓ (如果超出token限制)
🔄 层级上下文 (智能替换策略)
    ↓ (用子社区报告替换)
📝 最终上下文 (适合LLM处理)
```

###### **具体替换逻辑**：

1. **检查阶段**：
   - 计算本地上下文的token数量
   - 标记超出限制的社区 (`CONTEXT_EXCEED_FLAG`)

2. **替换阶段**：
   - 获取子社区的已生成报告
   - 按大小排序，优先替换最大的子社区
   - 保持在token限制内

3. **混合阶段**：
   - 结合子社区报告和剩余的本地上下文
   - 确保最终上下文符合要求



#### 3. 社区总结
基于上下文，生成社区总结

##### 3.1 分层处理机制
```python
levels = get_levels(nodes, schemas.COMMUNITY_LEVEL)  # 获取层级 [2, 1, 0] （2 为最低层，数字越大层级越低）
for level in levels:  # 从低层级到高层级处理
    # 高层级可以利用低层级已生成的报告
```

##### 3.2 上下文长度管理
```python
# 在build_level_context中
if context_size > max_context_tokens:
    # 使用子社区报告替代详细的本地上下文
    use_sub_community_reports()
else:
    # 使用完整的本地上下文
    use_local_context()
```

## 八、向量化
 `generate_text_embeddings` 函数主要负责为知识图谱中的各种文本数据生成向量嵌入（embeddings）。

### 具体工作：

1. **处理多种数据类型的嵌入**：
   - 文档文本嵌入 (`document_text_embedding`)
   - 关系描述嵌入 (`relationship_description_embedding`) 
   - 文本单元嵌入 (`text_unit_text_embedding`)
   - 实体标题嵌入 (`entity_title_embedding`)
   - 实体描述嵌入 (`entity_description_embedding`)
   - 社区标题嵌入 (`community_title_embedding`)
   - 社区摘要嵌入 (`community_summary_embedding`)
   - 社区完整内容嵌入 (`community_full_content_embedding`)

2. **数据预处理**：
   - 从输入的 DataFrame 中提取相关的 ID 和文本列
   - 对于实体描述嵌入，会将标题和描述合并为 "title:description" 格式

3. **批量生成嵌入**：
   - 根据配置中指定的 `embedded_fields` 来决定需要生成哪些类型的嵌入
   - 调用 `_run_and_snapshot_embeddings` 函数来实际执行嵌入生成

这个函数是 GraphRAG 流水线中的关键步骤，它将结构化的知识图谱数据转换为可以进行语义搜索和相似性匹配的向量表示，为后续的检索和生成任务奠定基础。





