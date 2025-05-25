# GraphRAG从0到1

## 目录
- [创建基本文本单元(create_base_text_units)](#创建基本文本单元)


- [提取图谱(extract_graph)](#提取图谱)


## 工作流
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
        - description: 描述

- relationship.parquet
    - columns包括:
        - source: 关系的起始实体
        - target: 关系的目标实体
        - text_units_id: 引用的文本单元id
        - weight: 关系权重
        - description: 描述
        


具体工作原理如下：

1. 首先，系统使用语言模型（如GPT）对文档进行一次初始提取，获取实体和它们之间的关系。

2. 如果`max_gleanings`设置为大于0的值，系统会继续进行额外的提取轮次，最多进行`max_gleanings`次额外提取：
   - 系统会向语言模型发送`CONTINUE_PROMPT`提示，要求它继续提取可能遗漏的实体和关系
   - 然后系统会向语言模型发送`LOOP_PROMPT`提示，询问是否还有更多实体和关系需要提取
   - 如果语言模型回答"Y"，则继续下一轮提取；如果回答"N"或已达到`max_gleanings`次数，则停止提取

3. 默认情况下，`max_gleanings`的值为1，表示除了初次提取外，最多再进行一次额外提取。

### 总结描述

