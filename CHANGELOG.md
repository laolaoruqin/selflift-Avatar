# Changelog

## v0.1.0-experimental

- 独立注册 Avatar H3、Image、TST 节点，不覆盖原版。
- 支持 H3 双流遮罩、多通道视频 mask、打包 mask 及音频静态约束。
- 修正 H3 音频约束的采样尺度转换。
- 支持附加到音频的恒定图像尺寸 SolidMask，拒绝非恒定图像网格。
- 修复带遮罩纯像素 anchor 分支。
- 26 项 CPU 测试通过；真实大模型生成未验证。
