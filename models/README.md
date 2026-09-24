# 端侧模型

本目录用于存放 AIFilmer 端侧推理所需的模型文件。

> ⚠️ 模型文件**不入仓**（已在 `.gitignore` 中排除）。模型通过首次启动按需下载，并在 `ModelManager` 中支持热更新。

## MVP 模型清单

| 模型 | 用途 | 大小 | 备注 |
|---|---|---|---|
| `subject_detector.mlmodel` | 主体检测 + 跟踪 | ~15MB | YOLOv8-Nano + ByteTrack |
| `scene_classifier.mlmodel` | 场景分类 | ~5MB | MobileNet-EfficientNet 蒸馏 |
| `face_landmarker.mlmodel` | 人脸关键点 | ~3MB | MediaPipe FaceLandmarker |
| `light_match_net.mlmodel` | 主体匹配（轻量，离线兜底） | ~10MB | 仅离线降级使用 |

**合计 ≤ 50MB**（按场景下载）

## 模型来源

- 自训练 / 蒸馏 / 第三方开源
- 不存原始模型权重，只记 Source 引用

详见 [详细设计 §2-§3](../docs/detailed-design.md)。