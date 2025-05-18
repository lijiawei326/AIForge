# Microsoft Graph RAG

## 读取文件
针对文件类型设定loader，lodaer的输出是一个df。

可以在settings.yaml中的input中加入以下参数：
- file_pattern: 正则表达式，用于匹配文件，其中定义的字段会通过group传递给`load_file`函数，会作为额外的列

- file_filter:需要过滤的值，通过字典类型传入SQL查询中
```python
query = "SELECT * FROM c WHERE RegexMatch(c.id, @pattern)"
parameters: list[dict[str, Any]] = [
    {"name": "@pattern", "value": file_pattern.pattern}
]
if file_filter:
    for key, value in file_filter.items():
        query += f" AND c.{key} = @{key}"
        parameters.append({"name": f"@{key}", "value": value})
items = list(
    self._container_client.query_items(
        query=query,
        parameters=parameters,
        enable_cross_partition_query=True,
    )
)
```
例如：
```python
file_filter = {"year": "2023", "source": "news"}
query = "SELECT * FROM c WHERE RegexMatch(c.id, @pattern) AND c.year = @year AND c.source = @source"
parameters = [
                {"name": "@pattern", "value": ".*\\.txt$"},
                {"name": "@year", "value": "2023"},
                {"name": "@source", "value": "news"}
            ]
```
- metadata：会将给定的列作为元数据

假设有如下DataFrame：
| text | source | year | project |
|--------------|--------|------|-----------|
| ...内容... | news | 2023 | my_proj |

合并后会多出一列：
| text | source | year | project | metadata |
|--------------|--------|------|-----------|------------------------------------------|
| ...内容... | news | 2023 | my_proj | {"source": "news", "year": "2023", "project": "my_proj"} |


### 示例
```
file_pattern: "^(?P<source>[^/]+)_(?P<year>\d{4})-(?P<month>\d{2})-(?P<day>\d{2})\.txt$"
```

| id | title | text | source | year | month | day | project |
|------|------------------------|--------------|--------|------|-------|-----|-------------|
| ... | news_2023-12-25.txt | ...内容... | news | 2023 | 12 | 25 | my_project |

## 文本分割
有两种分割策略可选：
1. 按token分割：设置size和overlap
2. 按句子分割：使用nltk库

`NLTK` 的句子分割器一般是通过 `sent_tokenize` 函数实现，能够有效地将文本分割成句子。比如一个代码示例是：

```python
    import nltk
    from nltk.tokenize import sent_tokenize

    # 下载 punkt 模型（如果尚未下载）
    nlt k.download('punkt')

    # 示例中文文本
    text = "你好！你最近怎么样？这是一个句子分割的示例。让我们看看它是如何工作的。"

    # 使用 sent_tokenize 进行句子分割
    sentences = sent_tokenize(text|, language='chinese')

    # 输出分割后的句子
    for sentence in sentences:
        print(sentence)
```

输出文本：

```
    你好！
    你最近怎么样？
    这是一个句子分割的示例。
    让我们看看它是如何工作的。
```

---

以上是`create_base_text_units`这一步工作的内容，结果是生成了两个parquet文件：
- documents.parquet
- text_units.parquet

即文档的信息表和文本块的信息表

当得到这两个文档以后，会执行`create_final_documents` 的`workflow`，将这两个文档的字段内容进行合并，即把`text_units.parquet`文件中的字段合并到`documents.parquet`文件中。在`output`文件夹下的`documents.parquet`文件会进行更新，关联到`text_units.parquet`文件中的`id`列

---
## 提取实体与关系

设计提示词，利用大模型最文本块提取实体与关系，得到两张表。

```python
entities_df = pd.DataFrame([
    {"title": "实体A", "description": ["描述A1", "描述A2"]},
    {"title": "实体B", "description": ["描述B1"]}
])

relationships_df = pd.DataFrame([
    {"source": "实体A", "target": "实体B", "description": ["关系描述1", "关系描述2"]}
])
```

提取完后会对所以结果进行合并：
```python
def _merge_entities(entity_dfs) -> pd.DataFrame:
    all_entities = pd.concat(entity_dfs, ignore_index=True)
    return (
        all_entities.groupby(["title", "type"], sort=False)
        .agg(
            description=("description", list),
            text_unit_ids=("source_id", list),
            frequency=("source_id", "count"),
        )
        .reset_index()
    )


def _merge_relationships(relationship_dfs) -> pd.DataFrame:
    all_relationships = pd.concat(relationship_dfs, ignore_index=False)
    return (
        all_relationships.groupby(["source", "target"], sort=False)
        .agg(
            description=("description", list),
            text_unit_ids=("source_id", list),
            weight=("weight", "sum"),
        )
        .reset_index()
    )

entity_dfs = []
relationship_dfs = []
for result in results:
    if result:
        entity_dfs.append(pd.DataFrame(result[0]))
        relationship_dfs.append(pd.DataFrame(result[1]))

entities = _merge_entities(entity_dfs)
relationships = _merge_relationships(relationship_dfs)
```

合并后实体和关系的description会组合成列表，然会对其进行总结。

最终，执行`extract_graph`的`workflow`后，在`output`文件夹下会生成`entities.parquet`文件和`relations.parquet`文件。

## 合并实体与关系信息与向量化

在提取出实体和关系后，会执行`finalize_graph`的，分别对`entities.parquet`文件和`relations.parquet`文件进行进一步处理，将执行`finalize_graph`的`workflow`。在这个过程中，将执行图节点和关系的聚合，并生成图的向量表示（是否执行由`settings.yaml`中的`embed_graph`控制）。

主要通过两个函数`finalize_entities` 和 `finalize_relationships` ，作用如下：

---

### 1. `finalize_entities`

**主要功能：**
- 对实体（entities）表进行最终的格式和特征处理，生成最终可用的实体数据。

**具体做的事情：**
1. **构建图结构**：根据关系（relationships）数据，构建一个图（networkx）。
2. **嵌入（可选）**：如果配置了嵌入（embed_config），则对图进行嵌入（embedding），生成实体的向量表示。
3. **布局（layout）**：对图进行布局计算，生成每个实体的可视化坐标等信息。
4. **度数计算**：计算每个实体在图中的度（degree）。
5. **合并信息**：将布局、度数等信息合并到原始实体表中。
6. **去重和清洗**：去除重复实体，填补缺失的度数为0，重建索引。
7. **生成ID**：为每个实体生成唯一的 `human_readable_id` 和 `id`（UUID）。
8. **筛选列**：只保留最终需要的列（由 `ENTITIES_FINAL_COLUMNS` 决定）。

---

### 2. `finalize_relationships`

**主要功能：**
- 对关系（relationships）表进行最终的格式和特征处理，生成最终可用的关系数据。

**具体做的事情：**
1. **构建图结构**：同样根据关系数据构建图。
2. **度数计算**：计算每个节点的度（degree）。
3. **去重**：去除重复的 source-target 关系。
4. **计算组合度**：为每条边（关系）计算 `combined_degree`，即源节点和目标节点的度数之和（或其他组合方式）。
5. **生成ID**：为每条关系生成唯一的 `human_readable_id` 和 `id`（UUID）。
6. **筛选列**：只保留最终需要的列（由 `RELATIONSHIPS_FINAL_COLUMNS` 决定）。

---

### 总结

- `finalize_entities` 负责对实体表进行特征补充、清洗、ID生成等，输出最终实体表。
- `finalize_relationships` 负责对关系表进行特征补充、去重、ID生成等，输出最终关系表。

这两个函数的目的是将原始抽取出来的实体和关系，转化为结构化、可用、可下游分析的标准格式。


## 社区检测

`Microsoft GraphRAG` 使用的莱顿算法是基于`graspologic`库的实现。它主要做的是对`Nodes`去划分社区，完成后，会生成`communities.parquet`文件。

可以在`setting.yml`中的`max_cluster_size`进行设置最大的社区数量。

最终结果示例：
| id           | human_readable_id | community | level | parent | children | title       | entity_ids       | relationship_ids  | text_unit_ids                                                                 | period     | size |
|--------------|-------------------|-----------|-------|--------|----------|-------------|------------------|-------------------|-------------------------------------------------------------------------------|------------|------|
| 9b5a9920...  | 0                 | 0         | 0     | -1     | []       | Community 0 | ['6db72e62...', '7c1f5f29...', 'ded79d6c...', 'c84e606a...', 'f8f6c7e4...', '91962af7...', 'f3e827a0...', 'b48cd41f...'] | ['4ee465fb...', '64145f1a...', '847704ca...', '97880361...', 'd11345d9...', 'e9af42ce...', 'f5a4ef58...'] | ['1154060a...', '124ef100...', '81edb5ae...', 'c5a95ef3...'] | 2025-03-07 | 8    |
| f1319f9d...  | 1                 | 1         | 0     | -1     | []       | Community 1 | ['13bae174...', '6b4cd36c...', 'f7426ad5...', '7469019e...', 'bf95e3ce...', 'cf2ee829...', 'd7a99dd2...', '05c54108...'] | ['3e89f931...', '521ac84f...', '6272883c...', '6fdcbcb3...', '8d3f818b...', 'd196074f...', 'eeb509e0...', 'f9e9a314...'] | ['363a168e...', '6f94edfd...'] | 2025-03-07 | 8    |



## 文本单元处理

`create_final_text_units` 函数的作用、目的、输入输出和处理逻辑如下：

---

### 1. 作用与目的

该函数的目的是**对原始的 text_units 数据进行整合和增强**，将实体（entities）、关系（relationships）、协变量（covariates，可选）等信息合并到每个 text_unit 上，最终生成一个结构化、便于下游使用的 DataFrame，作为“最终文本单元”表。

---

### 2. 输入

- `text_units: pd.DataFrame`  
  原始的文本单元数据，包含每个单元的基本信息（如 id、text、document_ids、n_tokens）。
- `final_entities: pd.DataFrame`  
  实体信息，每个实体关联了一个或多个 text_unit。
- `final_relationships: pd.DataFrame`  
  关系信息，每个关系关联了一个或多个 text_unit。
- `final_covariates: pd.DataFrame | None`  
  协变量信息（可选），每个协变量关联一个 text_unit。

---

### 3. 输出

- 返回值：`pd.DataFrame`  
  一个新的 DataFrame，包含了每个 text_unit 的基本信息、关联的实体ID、关系ID、协变量ID（如有），并且只保留了 `TEXT_UNITS_FINAL_COLUMNS` 里定义的列。

---

### 4. 处理逻辑

1. **选择基本字段**  
   只保留 text_units 的部分字段（id, text, document_ids, n_tokens），并新增一列 `human_readable_id`。

2. **实体、关系、协变量的处理**  
   - 用 `_entities` 函数处理实体表，得到每个 text_unit 关联的所有 entity_id。
   - 用 `_relationships` 函数处理关系表，得到每个 text_unit 关联的所有 relationship_id。
   - 如果有协变量表，则用 `_covariates` 处理，得到每个 text_unit 关联的 covariate_id。

3. **合并信息**  
   - 先将实体信息合并到 text_units 上，再合并关系信息。
   - 如果有协变量，再合并协变量信息；否则，covariate_ids 设为空列表。

4. **聚合**  
   - 按 text_unit 的 id 分组，聚合（agg("first")）每组的第一条数据，保证每个 id 只保留一行。

5. **输出**  
   - 只保留 `TEXT_UNITS_FINAL_COLUMNS` 里定义的列，作为最终输出。

---

### 5. 总结

**一句话总结：**  
`create_final_text_units` 用于将原始文本单元与其相关的实体、关系、协变量信息整合，生成结构化的最终文本单元表，便于后续知识图谱、检索等任务使用。

如需更详细的每一步代码解释，也可以继续问我！

### 输出示例
以下是Markdown格式的简化表格：

| id               | human_readable_id | community | level | parent | children | title                          | summary                                                                 | full_content                                                                 | rank | rating_explanation                                                                 | findings                                                                 | full_content_json                                                                 | period     | size |
|------------------|-------------------|-----------|-------|--------|----------|--------------------------------|-------------------------------------------------------------------------|------------------------------------------------------------------------------|------|-----------------------------------------------------------------------------------|--------------------------------------------------------------------------|----------------------------------------------------------------------------------|------------|------|
| 4dfd7889fb594... | 0                | 0         | 0     | -1     | []       | 亚马逊及其生态系统的影响分析   | 本报告主要探讨了亚马逊公司及其相关实体在科技行业中的重要地位和影响... | # 亚马逊及其生态系统的影响分析...                                            | 10.0 | 鉴于亚马逊在科技行业中的领导地位及其对全球经济的影响...                          | [{'explanation': 'Jeff Bezos作为Amazon的创始人...', 'summary': '...'}]  | { "title": "...", "summary": "...", "findings": [...] }                          | 2025-03-07 | 8    |
| e8c93819f129...  | 1                | 1         | 0     | -1     | []       | 科技巨头及其未来趋势           | 本报告主要关注苹果和谷歌这两家科技巨头之间的竞争与合作关系...          | # 科技巨头及其未来趋势...                                                     | 8.0  | 考虑到苹果和谷歌在全球科技行业的影响力及其在多个领域的发展...                     | [{'explanation': '苹果与谷歌在多个领域存在竞争关系...', 'summary': '...'}] | { "title": "...", "summary": "...", "findings": [...] }                          | 2025-03-07 | 8    |
| 9dc5e597707a...  | 2                | 2         | 0     | -1     | []       | 微软及其关键产品与服务         | 微软是一家科技巨头，由Paul Allen和Bill Gates在1975年创立...           | # 微软及其关键产品与服务...                                                   | 9.0  | 微软作为全球领先的科技公司，在多个技术领域拥有强大的影响力和市场地位...           | [{'explanation': '微软早期的产品之一是BASIC解释器...', 'summary': '...'}] | { "title": "...", "summary": "...", "findings": [...] }                          | 2025-03-07 | 8    |
| e4ce56468fdd...  | 3                | 3         | 0     | -1     | []       | Apple Inc.及其关键实体         | 该社区围绕科技公司Apple Inc.展开...                                    | # Apple Inc.及其关键实体...                                                   | 8.0  | 由于Apple Inc.在全球科技行业的领导地位及其对创新和技术发展的重大贡献...           | [{'explanation': 'Steve Jobs是Apple Inc.的创始人之一...', 'summary': '...'}] | { "title": "...", "summary": "...", "findings": [...] }                          | 2025-03-07 | 7    |
| fe2d07d9bc46...  | 4                | 4         | 0     | -1     | []       | 硅谷及其关键实体               | 该社区以硅谷为核心，包括斯坦福大学和加州伯克利分校等教育机构...        | # 硅谷及其关键实体...                                                         | 9.0  | 影响严重性评分高，因为硅谷在全球科技创新中扮演着关键角色...                       | [{'explanation': '硅谷是全球科技创新的中心...', 'summary': '...'}]       | { "title": "...", "summary": "...", "findings": [...] }                          | 2025-03-07 | 4    |

## 社区报告生成

报告应包括以下部分：
- 标题：代表其关键实体的社区名称——标题应简短但具体。如果可能，在标题中包含代表性命名实体。
- 摘要：社区整体结构的执行摘要，其实体之间的关系，以及与其实体相关的重要信息。
- 影响严重程度评级：0-10之间的浮动分数，代表社区内实体造成的影响的严重程度。影响力是一个社区的得分重要性。
- 评级解释：用一句话解释影响严重程度评级。
- 详细发现：关于社区的5-10个关键见解列表。每个见解都应该有一个简短的总结，然后是根据以下基础规则制定的多段解释性文本。要全面。

**输入：关系、实体、社区、主张（可选）四类表格数据。
输出：社区报告表格（DataFrame），并写入存储。**

## 向量化

根据`generate_text_embeddings`的实现，具体会对以下数据表中的字段进行向量化（embedding）：

### 1. documents 表
- **text** 字段  
  对每条 document 的 text 字段进行向量化。

### 2. relationships 表
- **description** 字段  
  对每条 relationship 的 description 字段进行向量化。

### 3. text_units 表
- **text** 字段  
  对每条 text_unit 的 text 字段进行向量化。

### 4. entities 表
- **title** 字段  
  对每条 entity 的 title 字段进行向量化。
- **title_description** 字段  
  这是由 title 和 description 拼接而成的新字段（格式为 "title:description"），对其进行向量化。

### 5. community_reports 表
- **title** 字段  
  对每条 community_report 的 title 字段进行向量化。
- **summary** 字段  
  对每条 community_report 的 summary 字段进行向量化。
- **full_content** 字段  
  对每条 community_report 的 full_content 字段进行向量化。

---

#### 总结表格

| 数据表              | 字段名              | 说明                                 |
|---------------------|---------------------|--------------------------------------|
| documents           | text                | 文档正文                             |
| relationships       | description         | 关系描述                             |
| text_units          | text                | 文本单元正文                         |
| entities            | title               | 实体标题                             |
| entities            | title_description   | 实体标题与描述拼接（title:description）|
| community_reports   | title               | 社区报告标题                         |
| community_reports   | summary             | 社区报告摘要                         |
| community_reports   | full_content        | 社区报告完整内容                     |

**注意**：实际会对哪些字段向量化，还取决于 embedded_fields 这个集合的配置（即本次任务需要哪些embedding），但上面列举的是所有可能被支持的字段。

如需了解 embedded_fields 的来源或具体配置，可以继续追问！


## 接入图数据库Neo4j



