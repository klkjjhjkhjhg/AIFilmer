# Cloud Backend

AIFilmer 的云侧后端服务。

## 技术栈（建议）

- Python 3.11+ / FastAPI
- LLM Provider: OpenAI / Anthropic / Qwen（统一 Adapter）
- 部署：单云厂商（建议 Aliyun / 阿里云函数计算）
- MVP 不需要多区域 / K8s

## 目录约定

```
backend/
├── cloud/
│   ├── app/                ← FastAPI 应用
│   ├── tests/              ← pytest
│   ├── prompts/            ← 提示词模板
│   └── templates/          ← 预制脚本模板（离线降级）
└── ...
```

## API 端点

| 方法 | 路径 | 用途 |
|---|---|---|
| POST | `/v1/scripts/generate` | 生成整组脚本 |
| POST | `/v1/scripts/{id}/shots/{idx}/regenerate` | 修改单个镜头 |
| POST | `/v1/subjects/match` | 主体匹配 |
| POST | `/v1/hints/direction` | 方向提示 |
| GET | `/v1/templates/{theme_class}` | 预制模板 |
| GET | `/v1/health` | 健康检查 |

详见 [概要设计 §4](../docs/high-level-design.md) 与 [详细设计 §6](../docs/detailed-design.md)。