# 工具治理与权限状态机演示

一个不依赖任何框架的 Python 教学演示：把 LLM Agent 的「工具调用」当作不可信输入，用**参数校验 → 权限决策 → 审批 → 超时与重试 → 结果脱敏 → 审计**一串确定性检查，把模型的能力关进可审计的笼子里。

所有调用都收敛到唯一入口 `ToolRuntime.invoke`，离线演示、CLI、真实模型 Agent Loop 与测试共用同一条链路。

## 核心能力

| 环节 | 做了什么 |
| --- | --- |
| 参数校验 | Pydantic `StrictArgs`（`extra="forbid"` + `strict=True`）把不可信字典转成业务对象，模型注入的 `user_id`、`approved` 等字段直接判非法 |
| 权限决策 | `PermissionEngine.decide` 的固定优先级三态状态机：deny 规则 → plan 只读 → 执行白名单 → RBAC → 业务预检 → 审批 → bypass → allow 规则 → 危险命令兜底 |
| 审批绑定 | `ApprovalStore` 用 `SHA-256(工具名 + 规范化参数)` 生成摘要，一次性消费，并校验 user / tenant / 工具 / 参数，参数被篡改即失效 |
| 超时与恢复 | 只读或幂等工具才重试；非幂等写超时返回 `TIMEOUT_UNKNOWN`，明确告诉模型「结果未知」而不是假装成功 |
| 结果脱敏 | `_redact` 统一屏蔽密钥类字段、邮箱与账号（`ACC-A-****3456`），handler 的原始结果不直接交给模型 |
| 审计 | 决策阶段与执行阶段各落一条 `AuditRecord`，记录 trace、用户、租户、工具、决策码与耗时 |
| 工具发现隔离 | `model_tools` 按白名单向模型投影 JSON Schema，只暴露描述与参数，不暴露 handler 与治理策略 |

## 内置工具

| 工具 | 效果 | 风险 | 权限 | 需审批 | 备注 |
| --- | --- | --- | --- | --- | --- |
| `get_order` | read | medium | `order:read` | 否 | 幂等、可重试，超时 1s |
| `create_refund` | write | high | `refund:create` | 是 | 退款预检订单状态与可退金额 |
| `transfer` | write | high | `transfer:execute` | 是 | 限额 / 余额预检，高额转账故意慢调用 |
| `run_shell` | shell | medium | `shell:run` | 否 | 教学模拟，不创建真实子进程 |

## 运行环境

- Python 3.11+（使用 `StrEnum`、`asyncio.timeout`）
- 依赖：

```bash
pip install pydantic pytest
```

- 可选（真实 Agent Loop）：`pip install openai`，并设置 `DEEPSEEK_API_KEY`

## 使用方式

离线演示（无需网络与密钥，打印每个工具调用返回的 `ToolResult` 与副作用计数）：

```bash
python tool_governance_demo.py
```

真实模型闭环：

```bash
set DEEPSEEK_API_KEY=sk-xxx
python tool_governance_demo.py --agent --input "请查询订单 ord_1001 的状态和可退金额"
```

可用环境变量：`DEEPSEEK_BASE_URL`（默认 `https://api.deepseek.com`）、`DEEPSEEK_MODEL`（默认 `deepseek-v4-flash`）。

运行测试：

```bash
pytest -v
```

## 目录结构

```
tool_governance_demo.py    # 全部实现：参数模型 / 权限状态机 / 运行时 / 工具 / 演示入口
conftest.py                # 让任意目录下运行 pytest 都能导入主模块
tests/test_tool_governance.py  # transfer 工具 5 条链路测试
```

## 演示覆盖的场景

离线演示依次触发：正常只读、写操作需要确认、审批后放行、参数注入被拦、`bypassPermissions` 仍挡不住 `deny` 规则、`plan` 模式拒绝写操作，以及 transfer 的四类结果：需审批、超限拒绝（`EXCEED_LIMIT`）、余额不足（`INSUFFICIENT_BALANCE`）、超时未知（`TIMEOUT_UNKNOWN`）。

## 设计要点

- **顺序即安全**：`decide` 的优先级不可调换。审批放在白名单与 RBAC 之后、allow 规则之前，避免「有审批就能绕过权限」。
- **只读契约不是提示词**：`plan` 模式在决策层直接拒绝写操作与 Shell，而不是靠系统提示词请求模型自觉。
- **执行期重新授权**：发现阶段的工具过滤只是减少模型可见能力，执行阶段必须重查白名单。
- **副作用在最后**：全部确定性检查通过后 handler 才可能产生副作用，超时点刻意安排在扣款之前。
- **正则只是教学兜底**：`DANGEROUS_SHELL_PATTERNS` 仅用于演示，生产应配合窄工具、AST 解析与沙箱。

## 注意

仓库中的订单、账户、审批与 Shell 均为内存中的模拟数据，`ORDERS`、`ACCOUNTS`、`SIDE_EFFECTS` 是模块级可变状态，仅供教学使用。
