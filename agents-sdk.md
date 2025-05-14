# OpenAI Agents SDK学习笔记

## 目录
- [Guardrail](#guardrail)


## Guardrail

### input guardrail
输入护栏, 当input传递给大模型时，同步传递给`input guardrail`。用于对输入进行安全判断。

示例：

#### 1. 定义护栏Agent
传入`output_type`只能自动解析为该格式，但是无法保证输出为该格式的json文本。目前仅发现了使用提示词强制限制输出的方法。

```python 
### 1. An agent-based guardrail that is triggered if the user is asking to do math homework
class MathHomeworkOutput(BaseModel):
    reasoning: str
    is_math_homework: bool
### 格式化输出需要设计提示词保证输出可以解析为json
instructions = """Check if the user is asking you to do their math homework.
Your response should be in the form of JSON with the following schema: {\"reasoning\": \"Your reasoning for why is math homework.\", \"is_math_homework\": \"true or false\"}. 
Do not include any other text and do not wrap the response in a code block - just provide the raw json output.
If need to use \' in your response, use \\\' instead.
"""
guardrail_agent = Agent(
    name="Guardrail check",
    instructions=instructions,
    output_type=MathHomeworkOutput,
    model=model
)
```

#### 2. 定义护栏函数
由于`input_guardrails`参数只能传递`list[InputGuardrail[TContext@Agent]]`, 因此需要定义一个`input_guardrail`函数。

被装饰函数的输入值必须是：
- context：运行上下文，类型为`RunContextWrapper[TContext]`，用于传递上下文信息。
- agent：当前的 `agent` 实例。
- input：输入内容，可以是字符串或 `TResponseInputItem` 的列表。

返回值必须是:
- `GuardrailFunctionOutput`实例
```python 
@input_guardrail
async def math_guardrail(
    context: RunContextWrapper[None], agent: Agent, input: str | list[TResponseInputItem]
) -> GuardrailFunctionOutput:
    """This is an input guardrail function, which happens to call an agent to check if the input
    is a math homework question.
    """
    result = await Runner.run(guardrail_agent, input, context=context.context)
    final_output = result.final_output_as(MathHomeworkOutput)

    return GuardrailFunctionOutput(
        output_info=final_output,
        tripwire_triggered=final_output.is_math_homework,
    )
```

#### 传入护栏
`input_guardrails`可以是多个，并行运行所有护栏。
```python
agent = Agent(
    name="Customer support agent",
    instructions="You are a customer support agent. You help customers with their questions.",
    input_guardrails=[math_guardrail],
    model=model
)
```
#### 调用
如果触发护栏,会返回`InputGuardrailTripwireTriggered`类，其属性包含了`InputGuardrailResult`
```python
try:
    result = await Runner.run(agent, '1+1=?')
except InputGuardrailTripwireTriggered as e:
    error_result = e.guardrail_result
    print("Guardrail triggered")
    print(f"Reasoning: {error_result.output.output_info.reasoning}")
else:
    print("Guardrail not triggered")
```



