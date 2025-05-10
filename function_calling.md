# Function Calling教程

- [原理](#原理)
- [JsonSchema基本结构](#JsonSchema基本结构)
- 示例
    - [python_inter_tool](#python_inter_tool)
    - [fig_inter_tool](#fig_inter_tool)
    - [sql_inter_tool](#sql_inter_tool)

---
## 原理

大模型通过一个Json Schema来获取工具信息，进而判断调用什么工具、设置什么参数。

## JsonSchema基本结构

一个工具调用的JSON Schema通常包含以下几个主要部分：

```json
{
    "type": "function",
    "function": {
        "name": "function_name",
        "description": "function_description",
        "parameters": {
            "type": "object",
            "properties": {
                // 参数定义
            },
            "required": [
                // 必填参数
            ]
        }
    }
}
```
## 示例
### python_inter_tool
<details>
<summary>具体代码</summary>

```python
def python_inter(py_code, g='globals()'):
    """
    专门用于执行python代码，并获取最终查询或处理结果。
    :param py_code: 字符串形式的Python代码，
    :param g: g，字符串形式变量，表示环境变量，无需设置，保持默认参数即可
    :return：代码运行的最终结果
    """    
    print("正在调用python_inter工具运行Python代码...")
    try:
        # 尝试如果是表达式，则返回表达式运行结果
        return str(eval(py_code, g))
    # 若报错，则先测试是否是对相同变量重复赋值
    except Exception as e:
        global_vars_before = set(g.keys())
        try:            
            exec(py_code, g)
        except Exception as e:
            return f"代码执行时报错{e}"
        global_vars_after = set(g.keys())
        new_vars = global_vars_after - global_vars_before
        # 若存在新变量
        if new_vars:
            result = {var: g[var] for var in new_vars}
            print("代码已顺利执行，正在进行结果梳理...")
            return str(result)
        else:
            print("代码已顺利执行，正在进行结果梳理...")
            return "已经顺利执行代码"

python_inter_tool = {
    "type": "function",
    "function": {
        "name": "python_inter",
        "description": f"当用户需要编写Python程序并执行时，请调用该函数。该函数可以执行一段Python代码并返回最终结果，需要注意，本函数只能执行非绘图类的代码，若是绘图相关代码，则需要调用fig_inter函数运行。\n同时需要注意，编写外部函数的参数消息时，必须是满足json格式的字符串，例如如以下形式字符串就是合规字符串：{python_inter_args}",
        "parameters": {
            "type": "object",
            "properties": {
                "py_code": {
                    "type": "string",
                    "description": "The Python code to execute."
                },
                "g": {
                    "type": "string",
                    "description": "Global environment variables, default to globals().",
                    "default": "globals()"
                }
            },
            "required": ["py_code"]
        }
    }
} 
```
</details>

#### 1. 工具类型声明

```json
"type": "function"
```

表明这是一个函数类型的工具。

#### 2. 函数定义

```json
"function": {
    "name": "python_inter",
    "description": "...",
    "parameters": {...}
}
```

- `name`: 函数名称，用于标识和调用
- `description`: 函数描述，说明用途和限制
- `parameters`: 参数定义

#### 3. 参数定义

```json
"parameters": {
    "type": "object",
    "properties": {
        "py_code": {
            "type": "string",
            "description": "The Python code to execute."
        },
        "g": {
            "type": "string",
            "description": "Global environment variables, default to globals().",
            "default": "globals()"
        }
    },
    "required": ["py_code"]
}
```

- `type`: 参数类型为对象(object)
- `properties`: 定义各个参数
  - `py_code`: 要执行的Python代码(string类型)
  - `g`: 全局变量环境(可选，默认为globals())
- `required`: 必须提供的参数列表(此处只有`py_code`是必需的)

### fig_inter_tool
<details>
<summary>具体代码</summary>

```python
def fig_inter(py_code, fname, g='globals()'):
    print("正在调用fig_inter工具运行Python代码...")
    import matplotlib
    import os
    import matplotlib.pyplot as plt
    import seaborn as sns
    import pandas as pd
    from IPython.display import display, Image

    # 切换为无交互式后端
    current_backend = matplotlib.get_backend()
    matplotlib.use('Agg')

    # 用于执行代码的本地变量
    local_vars = {"plt": plt, "pd": pd, "sns": sns}

    # 相对路径保存目录
    pics_dir = 'pics'
    if not os.path.exists(pics_dir):
        os.makedirs(pics_dir)

    try:
        # 执行用户代码
        exec(py_code, g, local_vars)
        g.update(local_vars)

        # 获取图像对象
        fig = local_vars.get(fname, None)
        if fig:
            rel_path = os.path.join(pics_dir, f"{fname}.png")
            fig.savefig(rel_path, bbox_inches='tight')
            display(Image(filename=rel_path))
            print("代码已顺利执行，正在进行结果梳理...")
            return f"✅ 图片已保存，相对路径: {rel_path}"
        else:
            return "⚠️ 代码执行成功，但未找到图像对象，请确保有 `fig = ...`。"
    except Exception as e:
        return f"❌ 执行失败：{e}"
    finally:
        # 恢复原有绘图后端
        matplotlib.use(current_backend)

fig_inter_tool = {
    "type": "function",
    "function": {
        "name": "fig_inter",
        "description": (
            "当用户需要使用 Python 进行可视化绘图任务时，请调用该函数。"
            "该函数会执行用户提供的 Python 绘图代码，并自动将生成的图像对象保存为图片文件并展示。\n\n"
            "调用该函数时，请传入以下参数：\n\n"
            "1. `py_code`: 一个字符串形式的 Python 绘图代码，**必须是完整、可独立运行的脚本**，"
            "代码必须创建并返回一个命名为 `fname` 的 matplotlib 图像对象；\n"
            "2. `fname`: 图像对象的变量名（字符串形式），例如 'fig'；\n"
            "3. `g`: 全局变量环境，默认保持为 'globals()' 即可。\n\n"
            "📌 请确保绘图代码满足以下要求：\n"
            "- 包含所有必要的 import（如 `import matplotlib.pyplot as plt`, `import seaborn as sns` 等）；\n"
            "- 必须包含数据定义（如 `df = pd.DataFrame(...)`），不要依赖外部变量；\n"
            "- 推荐使用 `fig, ax = plt.subplots()` 显式创建图像；\n"
            "- 使用 `ax` 对象进行绘图操作（例如：`sns.lineplot(..., ax=ax)`）；\n"
            "- 最后明确将图像对象保存为 `fname` 变量（如 `fig = plt.gcf()`）。\n\n"
            "📌 不需要自己保存图像，函数会自动保存并展示。\n\n"
            "✅ 合规示例代码：\n"
            "```python\n"
            "import matplotlib.pyplot as plt\n"
            "import seaborn as sns\n"
            "import pandas as pd\n\n"
            "df = pd.DataFrame({'x': [1, 2, 3], 'y': [4, 5, 6]})\n"
            "fig, ax = plt.subplots()\n"
            "sns.lineplot(data=df, x='x', y='y', ax=ax)\n"
            "ax.set_title('Line Plot')\n"
            "fig = plt.gcf()  # 一定要赋值给 fname 指定的变量名\n"
            "```"
        ),
        "parameters": {
            "type": "object",
            "properties": {
                "py_code": {
                    "type": "string",
                    "description": (
                        "需要执行的 Python 绘图代码（字符串形式）。"
                        "代码必须创建一个 matplotlib 图像对象，并赋值为 `fname` 所指定的变量名。"
                    )
                },
                "fname": {
                    "type": "string",
                    "description": "图像对象的变量名（例如 'fig'），代码中必须使用这个变量名保存绘图对象。"
                },
                "g": {
                    "type": "string",
                    "description": "运行环境变量，默认保持为 'globals()' 即可。",
                    "default": "globals()"
                }
            },
            "required": ["py_code", "fname"]
        }
    }
}
```
</details>

### sql_inter_tool
<details>
<summary>具体代码</summary>

```python
def sql_inter(sql_query, g='globals()'):
    """
    用于执行一段SQL代码，并最终获取SQL代码执行结果，\
    核心功能是将输入的SQL代码传输至MySQL环境中进行运行，\
    并最终返回SQL代码运行结果。需要注意的是，本函数是借助pymysql来连接MySQL数据库。
    :param sql_query: 字符串形式的SQL查询语句，用于执行对MySQL中telco_db数据库中各张表进行查询，并获得各表中的各类相关信息
    :param g: g，字符串形式变量，表示环境变量，无需设置，保持默认参数即可
    :return：sql_query在MySQL中的运行结果。
    """
    print("正在调用sql_inter工具运行SQL代码...")
    load_dotenv(override=True)
    host = os.getenv('HOST')
    user = os.getenv('USER')
    mysql_pw = os.getenv('MYSQL_PW')
    db = os.getenv('DB_NAME')
    port = os.getenv('PORT')
    
    connection = pymysql.connect(
        host = host,  
        user = user, 
        passwd = mysql_pw,  
        db = db,
        port = int(port),
        charset='utf8',
    )
    
    try:
        with connection.cursor() as cursor:
            sql = sql_query
            cursor.execute(sql)
            results = cursor.fetchall()
            print("SQL代码已顺利运行，正在整理答案...")

    finally:
        connection.close()

    return json.dumps(results)

sql_inter_tool = {
    "type": "function",
    "function": {
        "name": "sql_inter",
        "description": (
            "当用户需要进行数据库查询工作时，请调用该函数。"
            "该函数用于在指定MySQL服务器上运行一段SQL代码，完成数据查询相关工作，"
            "并且当前函数是使用pymsql连接MySQL数据库。"
            "本函数只负责运行SQL代码并进行数据查询，若要进行数据提取，则使用另一个extract_data函数。"
            "同时需要注意，编写外部函数的参数消息时，必须是满足json格式的字符串，例如以下形式字符串就是合规字符串："
            f"{sql_inter_args}"
        ),
        "parameters": {
            "type": "object",
            "properties": {
                "sql_query": {
                    "type": "string",
                    "description": "The SQL query to execute in MySQL database."
                },
                "g": {
                    "type": "string",
                    "description": "Global environment variables, default to globals().",
                    "default": "globals()"
                }
            },
            "required": ["sql_query"]
        }
    }
}
```
</details>

### extract_data_tool
<details>
<summary>具体代码</summary>

```python
def extract_data(sql_query, df_name, g='globals()'):
    """
    借助pymysql将MySQL中的某张表读取并保存到本地Python环境中。
    :param sql_query: 字符串形式的SQL查询语句，用于提取MySQL中的某张表。
    :param df_name: 将MySQL数据库中提取的表格进行本地保存时的变量名，以字符串形式表示。
    :param g: g，字符串形式变量，表示环境变量，无需设置，保持默认参数即可
    :return：表格读取和保存结果
    """
    print("正在调用extract_data工具运行SQL代码...")
    load_dotenv(override=True)
    
    host = os.getenv('HOST')
    user = os.getenv('USER')
    mysql_pw = os.getenv('MYSQL_PW')
    db = os.getenv('DB_NAME')
    port = os.getenv('PORT')
    
    connection = pymysql.connect(
        host = host,  
        user = user, 
        passwd = mysql_pw,  
        db = db,
        port = int(port),
        charset='utf8',
    )
    
    g[df_name] = pd.read_sql(sql_query, connection)
    print("代码已顺利执行，正在进行结果梳理...")
    return "已成功创建pandas对象：%s，该变量保存了同名表格信息" % df_name

extract_data_tool = {
    "type": "function",
    "function": {
        "name": "extract_data",
        "description": (
            "用于在MySQL数据库中提取一张表到当前Python环境中，注意，本函数只负责数据表的提取，"
            "并不负责数据查询，若需要在MySQL中进行数据查询，请使用sql_inter函数。"
            "同时需要注意，编写外部函数的参数消息时，必须是满足json格式的字符串，"
            f"例如如以下形式字符串就是合规字符串：{extract_data_args}"
        ),
        "parameters": {
            "type": "object",
            "properties": {
                "sql_query": {
                    "type": "string",
                    "description": "The SQL query to extract a table from MySQL database."
                },
                "df_name": {
                    "type": "string",
                    "description": "The name of the variable to store the extracted table in the local environment."
                },
                "g": {
                    "type": "string",
                    "description": "Global environment variables, default to globals().",
                    "default": "globals()"
                }
            },
            "required": ["sql_query", "df_name"]
        }
    }
}
```
</details>