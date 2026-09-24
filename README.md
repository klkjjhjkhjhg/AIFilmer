# 🎬 AIFilmer

> **让 AI 接管"拍摄决策"，把 iPhone 变成自主摄影师。**

AIFilmer 是一款 iOS 旗舰 App（MVP 阶段）。用户输入一段主题，AI 自主将主题拆分为多镜头脚本，每个镜头由 AI 实时控制拍摄参数（曝光、白平衡、对焦、变焦、调色），并在镜头切换时通过**三维方向提示**引导用户对准下一个拍摄主体。

**核心差异化**：AI 自主拍摄 + 用户保留指向权（手持移动设备），LLM 输出通过 **Shot Protocol**（MCP-like 协议）转换成可执行的相机动作序列。

---

## 📚 文档

完整的项目文档位于 [`docs/`](./docs/) 目录：

| 文档 | 说明 | Markdown | HTML |
|---|---|---|---|
| 📋 **需求分析** | User Story + 验收标准 + 非功能需求 + 风险 | [requirements-analysis.md](./docs/requirements-analysis.md) | [requirements-analysis.html](./docs/requirements-analysis.html) |
| 🏗️ **概要设计** | C4 模型（System Context + Container）+ 数据流 + 部署 | [high-level-design.md](./docs/high-level-design.md) | [high-level-design.html](./docs/high-level-design.html) |
| 🔧 **详细设计** | Director 状态机 + 双环 + Shot Protocol + 算法 + ADR | [detailed-design.md](./docs/detailed-design.md) | [detailed-design.html](./docs/detailed-design.html) |
| 🎬 **场景链路** | 夕阳下的咖啡店：端云 4 层协同 + 6 个模型嵌入点交互拆解 | —— | [scenario-walkthrough.html](./docs/scenario-walkthrough.html) |

> 🌐 在浏览器中阅读：[GitHub Pages 站点](https://<user>.github.io/AIFilmer/)（部署后启用）

---

## 🚀 MVP 范围

### ✅ In-Scope
- iOS 旗舰机（iPhone 14+）原生 App
- 端侧实时感知（场景识别 / 主体检测 / 跟踪）+ 系统级相机 API 自动调参
- 端侧主动调色（Metal Shader LUT）
- 云侧 LLM 拆镜本 + MLLM 主体匹配
- 多镜头连续拍摄 + 下一镜头三维方向提示
- 演示三类场景：**人物特写 / 风景静物 / Vlog 中景**

### ❌ Out-of-Scope（YAGNI）
- 语音交互（v1.1）
- 云台控制（v1.2）
- Android（v1.3）
- 手动接管模式、复杂运镜决策、后期剪辑

---

## 🏛️ 架构总览

```
┌────────────────────┐        ┌────────────────────┐
│   iOS App (Swift)  │ HTTPS  │  Cloud Backend     │
│   · Presentation   │ ◄────► │  · FastAPI         │
│   · Director       │        │  · Shot Protocol   │
│   · Fast Loop      │        │    Registry        │
│   · Creative Loop  │        │       │            │
│   · Capability     │        │       ▼            │
└────────────────────┘        │  LLM Provider      │
                              │  (GPT-4o / Qwen)   │
                              └────────────────────┘
```

**双环分层**：
- **Fast Loop**（≤ 500ms）：端侧 NPU 推理 + 系统相机 API
- **Creative Loop**（1~3s）：云侧 LLM 决策 + Shot Protocol 协议转换

详细架构见 [概要设计](./docs/high-level-design.html)。

---

## 📁 目录结构

```
AIFilmer/
├── README.md                       ← 项目说明
├── .gitignore                      ← Git 忽略规则
├── index.html                      ← GitHub Pages 首页（马卡龙配色）
├── docs/                           ← 设计文档（md + html）
│   ├── requirements-analysis.md / .html
│   ├── high-level-design.md / .html
│   ├── detailed-design.md / .html
│   └── scenario-walkthrough.html   ← 场景链路图（夕阳下的咖啡店）
├── protocol/                       ← Shot Protocol JSON Schema 与示例
├── models/                         ← 端侧模型存储（按需下载，不入仓）
├── app/                            ← iOS 工程（待创建）
│   ├── AIFilmer/
│   └── AIFilmerTests/
├── backend/                        ← Cloud Backend 工程（待创建）
│   └── cloud/
├── scripts/                        ← 构建 / 部署 / 工具脚本
├── .github/                        ← GitHub 配置
│   └── workflows/
└── docs/assets/                    ← 文档配图
```

---

## 📅 实施里程碑

| 里程碑 | 内容 | 可演示能力 |
|---|---|---|
| **M0** | App 骨架 + Camera 预览 + 录制 | 能录视频 |
| **M1** | 端侧 Vision Pipeline + PolicyEngine + Camera Driver | 单镜头 AI 调参与调色 |
| **M2** | Director 状态机 + UI（HUD + 下一镜头卡） | 单镜头完整闭环 |
| **M3** | Cloud Backend + Shot Protocol Client | LLM 拆镜本可用 |
| **M4** | Creative Loop + 主体匹配 + 方向提示 | 多镜头连续拍摄 |
| **M5** | 演示三类场景 + Golden Path | **MVP 完成** |

---

## 🤝 贡献

提交规范、PR 流程、ADR 模板等将在项目活跃开发阶段补充。

---

## 📄 许可证

待定。