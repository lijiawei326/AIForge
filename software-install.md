# 硬件安装

- [MySQL](#MySQL)
- [Postman](#MySQL)
- [GraphRAG](#graphrag源码安装)

---

## MySQL
- [Windows安装包链接](https://downloads.mysql.com/archives/community/)

---

## Postman
- [Postman安装包链接](https://www.postman.com/downloads/)

---

## graphrag源码安装
1. 克隆项目
```terminal
git clone https://github.com/microsoft/graphrag.git
```
2. 创建虚拟环境
```terminal
python -m venv venv
```
3. 进入虚拟环境
```terminal
### windows
.\venv\Scripts\activate
### linux/mac os
source venv/bin/activate
```

4. 安装poetry
microsoft graphrag使用poetry管理环境。
```terminal
pip install poetry
```

5. 使用poetry安装项目依赖
```terminal
# 更换源
# poetry source add --priority=primary mirrors https://pypi.tuna.tsinghua.edu.cn/simple/

poetry install
```

## Neo4j
- [Neo4j源码安装](https://neo4j.com/deployment-center/)
- [apoc插件](https://github.com/neo4j/apoc/releases)

