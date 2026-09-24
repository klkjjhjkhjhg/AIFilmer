# AIFilmer MVP — 详细设计

> **文档版本**: v1.0  
> **结构**: C4 Level 3（Component）+ 关键算法 + 接口契约 + ADR 完整版  
> **配合阅读**: [需求分析](./requirements-analysis.md) · [概要设计](./high-level-design.md)  
> **目标读者**: 实现工程师 / 算法工程师

---

## 0. 阅读指南

本文档面向实现层，按以下顺序阅读：

1. §1 Director 状态机 — 整个 App 的业务骨架
2. §2 Fast Loop / Creative Loop — 双环内部细节
3. §3 Capability Layer 模块接口 — 每个能力模块的 API
4. §4 Shot Protocol — MCP-like 协议 schema
5. §5 关键算法 — 跟踪平滑、调色 LUT、方向提示
6. §6 Cloud Backend — 服务接口、提示词模板
7. §7 性能预算与监控
8. §8 测试策略
9. §9 端到端错误处理
10. §10 隐私与合规实现
11. §11 ADR 完整版

---

## 1. Director 状态机

### 1.1 状态定义

```
                 ┌───────────────┐
                 │     Idle      │ ◄──────────────────────┐
                 └───────┬───────┘                        │
                         │ 用户输入主题 + 生成脚本           │
                         ▼                                │
                 ┌───────────────┐                        │
                 │  ScriptReady  │                        │
                 └───────┬───────┘                        │
                         │ 用户点"开始"+对准主体           │
                         ▼                                │
                 ┌───────────────┐                        │
            ┌────│   Shooting    │ ──── 镜头时长到 ───┐    │
            │    │   (镜头 N)    │                    │    │
            │    └───────────────┘                    ▼    │
            │                                  ┌───────────────┐
   手动结束镜头 │                                  │  NextShotPrompt │
            │                                  └───────┬───────┘
            │            ┌──── 接受 / 跳过 / 修改 ─────┘
            │            ▼
            └────► (回到 Shooting, N+1)
                                                    
                                                  │ 用户退出
                                                  ▼
                                          (回到 Idle)
```

### 1.2 状态枚举

```swift
enum AppState {
    case idle                              // 初始 / 拍摄完成后
    case generatingScript(ThemeInput)       // 等待脚本生成
    case scriptReady(ScriptSpec)            // 脚本已生成
    case shooting(ShotContext)              // 正在拍摄某镜头
    case nextShotPrompt(NextShotContext)    // 等待用户对下一镜头的决策
    case processing(ProcessingContext)      // 后期处理（如生成缩略图）
}
```

### 1.3 状态转换触发条件

| From | To | 触发事件 | 守卫条件 |
|---|---|---|---|
| Idle | GeneratingScript | 用户点"生成脚本" | 主题非空、网络可用 |
| GeneratingScript | ScriptReady | 后端返回有效 ShotSpec | 通过 schema 校验 |
| GeneratingScript | Idle | 用户取消 / 错误重试 3 次 | — |
| ScriptReady | Shooting | 用户点"开始"+ 已对准主体 | 检测到主体或用户跳过对准提示 |
| Shooting | NextShotPrompt | 当前镜头时长到 ±10% / 用户手动结束 | — |
| NextShotPrompt | Shooting | 用户点"接受" / "跳过" | — |
| NextShotPrompt | GeneratingScript | 用户点"修改"且修改的是当前镜头 | — |
| Shooting | Idle | 用户点"停止拍摄" | — |

### 1.4 Director 内部组件

| 组件 | 职责 |
|---|---|
| **StateStore** | 持有当前 AppState（单一可变源），订阅者收到 @Published 流 |
| **EventBus** | 无锁消息总线（actor / DispatchQueue + 原子计数器） |
| **LoopCoordinator** | 启动/停止 Fast Loop 与 Creative Loop，路由它们的输出 |
| **SharedState** | Fast Loop 与 Creative Loop 之间的只读快照（每 100ms 一份） |

### 1.5 状态机实现关键点

- **单写多读**：StateStore 是 Director 内唯一可变状态；Fast/Creative Loop 仅发布只读事件
- **无锁**：所有跨 Loop 通信通过事件，Director 在主线程串行处理事件
- **可回放**：所有事件落本地日志，便于 bug 复现

---

## 2. Fast Loop（端侧实时闭环）

### 2.1 节奏

- 触发：Camera 输出每一帧（30 fps）
- 处理：每帧一次，端侧 NPU 推理
- 输出：每帧一份 `FastLoopFrame` 消息，含场景、主体、调参指令

### 2.2 处理流水线

```
Camera Frame
    │
    ▼
┌─────────────────┐
│  Preprocess     │  · 缩放到模型输入尺寸 (192×192 主体 / 224×224 场景)
│  (Vision Kit)   │  · 归一化
└────────┬────────┘
         │
         ▼
┌─────────────────┐     ┌─────────────────┐
│  SubjectNet     │     │  SceneNet       │
│  (YOLOv8-Nano   │     │  (MobileNet-Eff)│
│   + ByteTrack)  │     │                 │
└────────┬────────┘     └────────┬────────┘
         │                       │
         ▼                       ▼
  SubjectState                 SceneState
         │                       │
         └───────────┬───────────┘
                     ▼
            ┌──────────────────┐
            │  PolicyEngine    │
            │  · 死区控制       │
            │  · 惯性平滑       │
            │  · 调参指令生成   │
            └────────┬─────────┘
                     ▼
           CameraAction (EV/AF/AWB/Zoom)
                     │
                     ▼
            ┌──────────────────┐
            │ CameraDriver     │
            │ (AVFoundation)   │
            └──────────────────┘
```

### 2.3 PolicyEngine 规则（核心算法）

```swift
struct PolicyEngine {
    // 调参死区：参数变化小于阈值不动作
    let exposureDeadZone: Float = 0.3      // EV ±0.3
    let whiteBalanceDeadZone: Float = 200   // K ±200
    let focusDeadZone: Float = 0.05         // 5% 画面范围
    let zoomSmoothingFactor: Float = 0.2    // EMA 系数
    
    func decide(prev: CameraAction?, 
                curr: RawInference,
                sceneHint: SceneHint) -> CameraAction {
        // 1. 调参死区
        var action = curr.suggestedAction
        if let prev = prev {
            action.exposureBias = applyDeadZone(
                prev.exposureBias, curr.suggestedAction.exposureBias, 
                deadZone: exposureDeadZone)
            // ... 其他参数类似
        }
        
        // 2. 场景分类修正：夜景不放低 EV，人像优先肤色
        if sceneHint == .nightPortrait {
            action.exposureBias = max(action.exposureBias, -0.5)
        }
        
        // 3. 跟踪平滑：使用 EMA 平滑 zoom factor
        action.zoomFactor = ema(prev?.zoomFactor ?? 1.0, 
                                 curr.suggestedAction.zoomFactor, 
                                 alpha: zoomSmoothingFactor)
        
        return action
    }
}
```

### 2.4 延迟预算

| 阶段 | 预算 |
|---|---|
| Camera capture → preprocess | 50ms |
| SubjectNet + SceneNet 推理 | 50ms |
| PolicyEngine | < 5ms |
| CameraDriver 应用 | 100ms |
| **总 Fast Loop** | **≤ 500ms（含缓冲）** |

---

## 3. Creative Loop（云云协同闭环）

### 3.1 节奏

- 触发：Director 状态机事件
- 频率：0.3~0.5 Hz（每 2~3s 一次）
- 输出：决策事件（脚本生成、下一镜头推荐、方向提示）

### 3.2 处理流水线

```
Director Event (e.g. ShotDurationReached)
    │
    ▼
┌────────────────────┐
│ CreativeLoopService │
│ · 收集当前帧缩略图 │
│ · 收集最近 SharedState │
│ · 收集 ShotSpec     │
└─────────┬──────────┘
          ▼
┌────────────────────┐
│ ShotProtocolClient │
│ · HTTPS POST       │
│ · 带请求 ID + 重试  │
└─────────┬──────────┘
          │
          ▼
   [Cloud Backend]
          │
          ▼
   ShotSpec / NextShotHint
          │
          ▼
   Director 处理
```

### 3.3 Creative Loop 调用类型

| 类型 | 输入 | 输出 | 触发 |
|---|---|---|---|
| **GENERATE_SCRIPT** | ThemeText | ScriptSpec | 用户点"生成脚本" |
| **REGENERATE_SHOT** | ShotIndex + ModifyHint | SingleShotSpec | 用户点"修改镜头" |
| **MATCH_SUBJECT** | currentFrame + targetDescription | SubjectMatchResult | 镜头切换前 |
| **COMPUTE_DIRECTION** | prevFrame + currFrame + target | DirectionHint | 镜头切换前 |
| **MODIFY_NEXT_SHOT** | ShotSpec + ModifyHint | SingleShotSpec | 用户修改下一镜头 |

---

## 4. Shot Protocol（MCP-like 协议）

### 4.1 设计原则

- **协议 schema 冻结**：协议版本号 `v1`，向后兼容
- **工具抽象**：每条能力 = 一个 tool，LLM 输出是 tool calls 序列
- **JSON Schema 严格校验**：服务端校验，客户端再校验
- **失败可恢复**：单条 tool 失败不影响其他 tool

### 4.2 Protocol Version 1

```json
{
  "protocol_version": "v1",
  "request_id": "uuid-v4",
  "tool_call": {
    "tool_name": "GENERATE_SCRIPT",
    "arguments": { ... }
  }
}
```

### 4.3 工具定义

#### 4.3.1 GENERATE_SCRIPT

**输入**：
```json
{
  "theme": "夕阳下的咖啡店",
  "shot_count": 4,
  "style_profile": "warm_warm"
}
```

**输出 ScriptSpec**：
```json
{
  "script_id": "uuid",
  "style_profile": {
    "name": "warm_warm",
    "color_temp_shift": 250,
    "contrast": 1.05,
    "saturation": 1.1
  },
  "shots": [
    {
      "index": 0,
      "type": "wide",                    // wide / medium / close-up / extreme-close-up
      "subject": "咖啡店门头全景",
      "subject_descriptor": "咖啡店入口 + 招牌",
      "duration_sec": 4.0,
      "camera_motion": "static",          // static / pan / tilt / zoom-in / zoom-out
      "color_intent": "warm_sunset",
      "composition_hint": {
        "rule_of_thirds": "subject_lower",
        "focal_point": "center-bottom"
      },
      "ai_capability_request": {
        "tracking": "off",
        "exposure_strategy": "ev_locked",
        "color_strategy": "warm"
      }
    }
  ]
}
```

**JSON Schema**：见附录 A（详细 schema 在单独文件 `shot-protocol-v1.schema.json`）。

#### 4.3.2 REGENERATE_SHOT / MODIFY_NEXT_SHOT

输入：
```json
{
  "shot_index": 2,
  "current_shot": { ... },
  "modify_hint": "这个镜头再近一点，拍特写"
}
```

输出：`SingleShotSpec`（同 ScriptSpec.shots[0] 结构）。

#### 4.3.3 MATCH_SUBJECT

输入：
```json
{
  "current_frame_url": "https://...",        // 仅主体裁剪后的小图，< 50KB
  "target_description": "咖啡店门口的猫"
}
```

输出：
```json
{
  "match_status": "in_frame" | "not_in_frame",
  "matched_subject": {
    "id": "subject-uuid",
    "bbox_norm": [0.1, 0.2, 0.4, 0.6],       // [x, y, w, h] 归一化坐标
    "confidence": 0.92
  }
}
```

#### 4.3.4 COMPUTE_DIRECTION

输入：
```json
{
  "prev_frame_bbox": [0.3, 0.4, 0.5, 0.6],
  "curr_frame_bbox": [0.35, 0.45, 0.55, 0.65],
  "target_bbox": [0.7, 0.5, 0.8, 0.6],
  "target_in_frame": true
}
```

输出：
```json
{
  "direction_hint": "right",                // left / right / up / down / forward / back / center
  "confidence": 0.88,
  "natural_language": "请把镜头向右移一点",
  "converge_indicator": true               // 是否在收敛
}
```

### 4.4 错误处理

| 错误码 | 含义 | 客户端处理 |
|---|---|---|
| `INVALID_SCHEMA` | LLM 输出违反 schema | 自动 retry 1 次，第二次失败用预制模板 |
| `TIMEOUT` | LLM 推理超时（> 5s） | 用预制模板 + 提示用户 |
| `LLM_RATE_LIMIT` | 限流 | 退避 2s 后 retry |
| `SUBJECT_NOT_FOUND` | MATCH_SUBJECT 未找到 | 提示"请旋转手机寻找" |

### 4.5 协议版本管理

- 协议变更走 `v2`，旧 `v1` 至少保留 6 个月兼容
- 服务端必须支持 `v1` + `v2` 双版本同时返回
- 客户端升级到 `v2` 后再放弃 `v1`

---

## 5. 关键算法

### 5.1 主体跟踪平滑（Kalman Filter）

```swift
class SubjectTracker {
    private var kf: KalmanFilter2D
    private var lastUpdate: TimeInterval = 0
    
    func update(detections: [BBox]) -> BBox? {
        // 1. 数据关联：IOU 匹配
        let matched = matchByIOU(detections, lastDetection, threshold: 0.3)
        
        // 2. 卡尔曼预测 + 更新
        kf.predict(dt: now - last)
        if let m = matched {
            kf.update(measurement: m.center)
        } else {
            // 丢失超过 1s：丢失处理（不再预测）
            return nil
        }
        
        lastUpdate = now
        return kf.getStateBBox()
    }
}
```

### 5.2 主动调色（Metal Shader LUT）

```metal
// ColorEngine 核心 shader
fragment half4 colorLUTFragment(
    VertexOut in [[stage_in]],
    texture2d<float, access::sample> inputTex [[texture(0)]],
    texture2d<float, access::sample> lutTex [[texture(1)]],
    constant ColorParams &params [[buffer(0)]]
) {
    half4 color = inputTex.sample(s, in.texCoord);
    
    // 色温偏移（暖/冷）
    color.r += params.tempShift * color.r;
    color.b -= params.tempShift * color.b;
    
    // 对比度
    color.rgb = (color.rgb - 0.5) * params.contrast + 0.5;
    
    // 饱和度
    half gray = dot(color.rgb, half3(0.299, 0.587, 0.114));
    color.rgb = mix(half3(gray), color.rgb, half(params.saturation));
    
    // 肤色保护
    if (params.skinProtect > 0) {
        half skinMask = computeSkinMask(color);
        color.rgb = mix(color.rgb, color.rgb * params.skinSafe, skinMask * params.skinProtect);
    }
    
    return color;
}
```

**参数范围（防失真）**：
- 色温偏移：±15%（params.tempShift ∈ [-0.15, 0.15]）
- 饱和度：0.8 ~ 1.2
- 对比度：0.9 ~ 1.1
- 肤色保护权重：0.0 ~ 0.5

### 5.3 三维方向提示生成

```swift
struct DirectionComputer {
    func compute(
        prev: BBox?, 
        curr: BBox, 
        target: BBox?, 
        targetInFrame: Bool
    ) -> DirectionHint {
        // 目标不在画面内
        guard targetInFrame else {
            return DirectionHint(
                direction: .search,
                naturalLanguage: "请旋转手机寻找：\(targetDescription)",
                confidence: 0.0
            )
        }
        
        guard let target = target else {
            return DirectionHint(direction: .center, ...)
        }
        
        // 目标在画面内：基于偏差给方向
        let dx = (target.center.x - 0.5) * 2  // [-1, 1]
        let dy = (target.center.y - 0.5) * 2
        
        let direction: Direction
        if abs(dx) > abs(dy) {
            direction = dx > 0.05 ? .right : (dx < -0.05 ? .left : .center)
        } else {
            direction = dy > 0.05 ? .down : (dy < -0.05 ? .up : .center)
        }
        
        // 收敛性判断：与上一帧方向相反 → 已收敛
        let converging = prev.map { 
            oppositeDirection($0.direction, direction) 
        } ?? false
        
        return DirectionHint(
            direction: direction,
            naturalLanguage: directionText(direction),
            confidence: 0.9,
            converge_indicator: converging
        )
    }
}
```

### 5.4 风格一致性（StyleProfile）

```swift
struct StyleProfile: Codable {
    let name: String                // "warm_sunset" / "cool_morning" 等
    let colorTempShift: Float       // ±0.15
    let contrast: Float             // 0.9~1.1
    let saturation: Float           // 0.8~1.2
    let lutPath: String?            // 可选 LUT 文件
    
    // 整段拍摄期间锁定；切镜头不重新调
    var isLocked: Bool = true
}
```

---

## 6. Cloud Backend 详细设计

### 6.1 API 端点

| 方法 | 路径 | 用途 |
|---|---|---|
| POST | `/v1/scripts/generate` | 生成整组脚本 |
| POST | `/v1/scripts/{id}/shots/{idx}/regenerate` | 修改单个镜头 |
| POST | `/v1/subjects/match` | 主体匹配 |
| POST | `/v1/hints/direction` | 方向提示 |
| GET  | `/v1/templates/{theme_class}` | 预制模板（离线降级） |
| GET  | `/v1/health` | 健康检查 |

### 6.2 错误响应格式

```json
{
  "error": {
    "code": "INVALID_SCHEMA",
    "message": "shots[2].duration_sec must be in [1, 10]",
    "retry_safe": true
  }
}
```

### 6.3 提示词模板（核心）

#### GENERATE_SCRIPT 提示词骨架

```
你是 AIFilmer 的镜头语言导演。用户给出一段主题描述，请将其拆分为 N 个镜头。

【输出要求】
- 严格输出 JSON，遵守 ShotSpec schema
- 每个镜头字段必须完整：type / subject / duration_sec / camera_motion / color_intent / ai_capability_request
- 镜头时长建议 3~6 秒；切镜头时长不要相同（避免单调）
- 镜头之间要有叙事节奏（建立 → 发展 → 高潮 → 收尾）
- 调色意图词汇：warm_sunset / cool_morning / neutral_daylight / night_warm / moody_cool 等

【主题】
{theme}

【风格】
{style_profile_name}

【输出 N】
{shot_count}
```

### 6.4 预制模板（MVP 3 套）

| 主题类 | 镜头数 | 风格 | 用途 |
|---|---|---|---|
| 人物特写 | 4 | warm_portrait | "拍一段人像" |
| 风景静物 | 4 | cool_morning | "拍一段风景" |
| Vlog 中景 | 5 | neutral_daylight | "拍一段 Vlog" |

> 离线时直接返回预制模板，UX 与在线接近。

### 6.5 服务层实现要点

- **LLM 调用**：统一通过 `LLMProviderAdapter`，支持 OpenAI / Anthropic / Qwen
- **重试策略**：INVALID_SCHEMA → 1 次自动 retry；TIMEOUT → 退避 2s 后 retry；连续 3 次失败返回错误
- **降级**：连续失败 → 走预制模板
- **缓存**：相同 theme + style_profile 命中缓存 5 分钟
- **限流**：每用户 100 次/小时（MVP 简化配额）

---

## 7. 性能预算与监控

### 7.1 端到端性能预算

| 链路 | 预算 |
|---|---|
| Fast Loop 总延迟 | ≤ 500ms |
| Creative Loop 拆镜本 | ≤ 5s |
| Creative Loop 主体匹配 | ≤ 2s |
| Creative Loop 方向提示 | ≤ 1s |
| 视频录制连续时长 | ≥ 10 min |
| 录制掉帧率 | ≤ 0.1% |

### 7.2 客户端监控指标

| 指标 | 采集方式 | 上报 |
|---|---|---|
| Fast Loop 延迟分布 | Director 内部计时 | 本地日志 |
| 调参指令生效延迟 | CameraDriver 回调 | 本地日志 |
| 跟踪丢失率 | SubjectTracker 内部 | 本地日志 |
| 应用启动时间 | OS 钩子 | 本地日志 |
| 录制掉帧数 | AVAssetWriter 回调 | 本地日志 |

> MVP 不上报任何指标到云端（保护隐私）。

### 7.3 服务端监控指标

| 指标 | 采集方式 | 告警阈值 |
|---|---|---|
| API P95 延迟 | 中间件 | > 5s |
| LLM token 消耗 | Provider 响应 | 异常突增 |
| INVALID_SCHEMA 比率 | 后端埋点 | > 20% |
| 错误率 | 后端埋点 | > 5% |

---

## 8. 测试策略

### 8.1 测试矩阵

| 层级 | 类型 | 工具 | 覆盖目标 |
|---|---|---|---|
| 单元 | Director 状态机 | XCTest | 状态转换 100% |
| 单元 | PolicyEngine | XCTest | 死区 / 平滑 / 场景修正 |
| 单元 | SubjectTracker | XCTest | 跟踪稳定性 / 丢失恢复 |
| 单元 | DirectionComputer | XCTest | 收敛性判断 |
| 集成 | Shot Protocol | XCTest + 后端 fixture | schema 校验 / retry |
| 集成 | Fast Loop pipeline | XCTest | 端到端延迟 |
| 集成 | Cloud Backend | pytest | API 正确性 |
| 系统 | 三类演示场景 | XCTest UI | 连续拍摄成功率 ≥ 90% |
| 系统 | 设备矩阵 | TestFlight | iPhone 14/15/16 各 2 台 |

### 8.2 关键测试用例

- **TC-01**：状态机 ScriptReady → Shooting 的守卫条件
- **TC-02**：PolicyEngine 在 EV 死区内不动作
- **TC-03**：跟踪器在主体丢失 1s 后正确返回 nil
- **TC-04**：调色参数饱和度上限保护肤色不失真
- **TC-05**：LLM 返回 INVALID_SCHEMA 自动 retry 1 次
- **TC-06**：离线时拆镜本走预制模板
- **TC-07**：方向提示的收敛性（连续 3 次提示方向相反）

### 8.3 Golden Path 录制

三类演示场景各录制 1 段标准视频，作为回归基准。

---

## 9. 端到端错误处理

### 9.1 错误分类

| 类别 | 例子 | 处理 |
|---|---|---|
| **可恢复网络错误** | 超时 / 5xx | 自动 retry + 退避 |
| **可恢复 LLM 错误** | INVALID_SCHEMA | retry 1 次 + 模板兜底 |
| **不可恢复错误** | 用户取消 / API Key 失效 | 显示明确错误 |
| **设备能力错误** | NPU 不可用 | 降级 + 提示 |

### 9.2 用户可见错误文案规范

- **不出现技术术语**："JSON 解析失败" → "AI 暂时没想好，我们再试一次？"
- **永远给下一步动作**：除"取消"外必须有"重试 / 使用备选方案"
- **永远可回退**：每个错误状态都能返回到 Idle

### 9.3 Director 错误状态机扩展

```swift
enum AppState {
    // ...
    case error(ErrorContext, recoveryAction: RecoveryAction)
}

enum RecoveryAction {
    case retry
    case useTemplate
    case exit
}
```

---

## 10. 隐私与合规实现

### 10.1 数据流隐私分级

| 数据 | 存储位置 | 是否上传 |
|---|---|---|
| 用户拍摄的视频 | App 沙盒（仅本地） | ❌ 不上传 |
| 主题文本 | 仅在请求体内 | ✅ 上传（脱敏） |
| 主体匹配用图片 | 临时文件，请求后删除 | ✅ 上传（裁剪至 < 50KB） |
| 脚本 JSON | 本地缓存 | ❌ 不上传 |
| 端侧模型 | App 沙盒 | ❌ 不上传 |

### 10.2 App Store 隐私声明要点

- 相机权限："用于拍摄视频"
- （MVP 不需要）麦克风权限：留待 v1.1
- 不收集位置 / 通讯录 / 标识符

### 10.3 实现要点

- 主体匹配上传的图片必须客户端裁剪（仅主体区域）
- 任何上传失败 → 走本地兜底，不阻塞主流程
- 用户可一键清除本地所有数据

---

## 11. ADR 完整版

### ADR-001 双环分层架构
- **状态**：已批准
- **决策**：Fast Loop（≤ 500ms，端侧）与 Creative Loop（1~3s，云云协同）物理隔离，通过 SharedState + EventBus 通信
- **理由**：满足延迟预算；端云分层清晰；扩展性好
- **后果**：Director 必须是无锁的纯消息路由；Capability Layer 接口要稳定

### ADR-002 端侧优先 + 云侧可选
- **状态**：已批准
- **决策**：所有能力标记 Online-Required / Online-Optional / Offline-Only；MVP 不允许 Online-Required 阻塞主流程
- **理由**：断网时核心拍摄能力必须可用
- **后果**：脚本生成、主体匹配、方向提示在断网时降级

### ADR-003 Shot Protocol 协议先行
- **状态**：已批准
- **决策**：定义 MCP-like 协议，所有 LLM 输出必须遵守；协议 schema 冻结
- **理由**：协议稳定后再扩展能力，避免每加一个能力重写 LLM 输出
- **后果**：服务端维护 schema 注册表；客户端解析失败可降级

### ADR-004 设备白名单
- **状态**：已批准
- **决策**：MVP 仅在 iPhone 14 及以上 iOS 17+ 设备上保证完整能力
- **理由**：不同 iPhone 机型 AVFoundation 行为差异巨大
- **后果**：App 启动时检测设备能力；不达标给出明确提示

### ADR-005 状态机驱动而非事件链驱动
- **状态**：已批准
- **决策**：Director 用显式状态机而非回调链
- **理由**：可测试性高；状态转换可枚举；bug 容易复现
- **后果**：所有 UI 必须订阅 StateStore；不能反向写状态

### ADR-006 LLM 输出必须走 schema 校验 + 预制模板兜底
- **状态**：已批准
- **决策**：所有 LLM 输出在客户端和服务端都过 JSON Schema 校验；连续失败走预制模板
- **理由**：LLM 输出不稳定是结构性风险，必须有结构性兜底
- **后果**：服务端必须有完整的 schema 定义；预制模板必须覆盖三大演示场景

### ADR-007 视频与脚本完全本地存储
- **状态**：已批准
- **决策**：用户视频与脚本一律本地存储，不上传云端
- **理由**：隐私优先；MVP 用户接受度依赖此承诺
- **后果**：历史视频管理功能完全本地；不做云端备份（MVP）

---

## 12. 实施顺序（建议）

按以下顺序实现，每个里程碑独立可演示：

| 里程碑 | 内容 | 可演示能力 |
|---|---|---|
| **M0** | App 骨架 + Camera 预览 + 录制 | 能录视频 |
| **M1** | 端侧 Vision Pipeline + PolicyEngine + Camera Driver | 单镜头 AI 调参与调色 |
| **M2** | Director 状态机 + UI（HUD + 下一镜头卡） | 单镜头完整闭环 |
| **M3** | Cloud Backend + Shot Protocol Client | LLM 拆镜本可用 |
| **M4** | Creative Loop + 主体匹配 + 方向提示 | 多镜头连续拍摄 |
| **M5** | 演示三类场景 + Golden Path | MVP 完成 |

---

## 附录 A：JSON Schema 文件

实际 JSON Schema 文件位于：

- `F:\Codes\AIFilmer\protocol\shot-protocol-v1.schema.json`

（待实施时创建）

---

## 附录 B：参考资料

- Apple AVFoundation 文档
- Core ML 性能优化指南
- YOLOv8-Nano 模型卡
- ByteTrack 论文
- MCP（Model Context Protocol）规范

---

**文档结束**。如有歧义或需细化，请在 [issues] 中提出。