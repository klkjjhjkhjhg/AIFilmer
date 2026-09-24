# iOS App

AIFilmer 的 iOS 工程目录。

## 技术栈

- Swift 5.9+
- SwiftUI（Presentation 层）
- AVFoundation（Camera Driver）
- Core ML（端侧推理）
- Metal（GPU 调色 Shader）
- VisionKit（图像预处理）

## 目录约定

```
app/
├── AIFilmer/                ← 主工程源码
│   ├── Presentation/        ← SwiftUI 视图
│   ├── Director/            ← 状态机 + 双环编排
│   ├── Capability/          ← 端侧能力模块
│   └── Resources/
├── AIFilmerTests/           ← 单元测试
└── ...
```

## 工程初始化

待 M0 阶段创建（建议用 Xcode 15+ 生成工程）。

详见 [详细设计 §1-§5](../docs/detailed-design.md)。