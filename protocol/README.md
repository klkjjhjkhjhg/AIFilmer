# Shot Protocol

MCP-like 镜头语言协议层。把 LLM 输出（自然语言 / 自由 JSON）转换成可执行的相机动作序列。

## 文件

- `shot-protocol-v1.schema.json` — JSON Schema 定义（待创建）
- `examples/` — 各类工具的请求 / 响应示例（待创建）

## 协议版本

当前版本：**v1**（向后兼容；v2 至少保留 6 个月兼容窗口）

## 工具列表

| 工具名 | 用途 |
|---|---|
| `GENERATE_SCRIPT` | 用户主题 → 多镜头脚本 |
| `REGENERATE_SHOT` | 重新生成某个镜头 |
| `MATCH_SUBJECT` | 主体匹配（MLLM） |
| `COMPUTE_DIRECTION` | 三维方向提示 |
| `MODIFY_NEXT_SHOT` | 修改下一镜头 |

## 设计原则

1. 协议 schema 冻结，所有 LLM 输出必须严格遵守
2. 单条 tool 失败不影响其他 tool
3. 客户端 + 服务端都做 schema 校验
4. 协议变更走版本号（v1 → v2）

详见 [详细设计 §4](../docs/detailed-design.md)。