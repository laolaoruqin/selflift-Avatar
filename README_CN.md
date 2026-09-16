# selflift-Avatar

**当前公开开发版本：`v0.1.2-experimental`** · [English README](README.md) · [更新记录](CHANGELOG.md) · [v0.1.2 发布页](https://github.com/slmonker/selflift-Avatar/releases/tag/v0.1.2-experimental)

这是一个面向 ComfyUI / MiniMax H3 的 SelfLift 独立实验分支，重点处理 H3 音视频 latent 遮罩、采样过程中的原始音频保留，以及一个有限制的高分辨率自动分块路径。它使用独立的节点 ID，可以和原版 SelfLift 共存。

> **实验版本，非官方项目。** v0.1.2 已通过 34 项 CPU 测试，并在维护者的 H3 工作流中试用。尚未进行系统性的口型准确率、速度和所有模型/插件组合测试。不同角色、音频、提示词、采样器、latent upscaler 和 seed 可能产生不同结果。

## v0.1.2 更新内容

- 当视频全量生成（`video mask` 全为 1）、输入音频全量保留（`audio mask` 全为 0）时，允许开启高分辨率分块。
- 根据可用工作区自动选择分块方向和数量，当前规划器可选择 1～8 块。
- 每个视频块都会收到完整音频 latent 和完整 H3 音频条件；音频不会按画面分块。
- 仍然拒绝视频局部保留、音频局部/软遮罩，以及尚未验证的其他遮罩组合。
- 新增分块拼接、完整音频保留、H3 音频遮罩传递、非法组合和无遮罩路径测试。

## Demo

[![H3 音频驱动 Demo 预览](docs/assets/demo-preview.gif)](https://github.com/slmonker/selflift-Avatar/releases/download/v0.1.2-experimental/selfliftAvatar_00011-audio.mp4)

**H3 音频驱动 Demo：** [下载 / 查看完整 13.67 秒 MP4](https://github.com/slmonker/selflift-Avatar/releases/download/v0.1.2-experimental/selfliftAvatar_00011-audio.mp4) · [打开 v0.1.2 发布页](https://github.com/slmonker/selflift-Avatar/releases/tag/v0.1.2-experimental)

上方是静音 GIF 片段，点击即可获取带声音的完整视频。该视频由维护者提供，作为公开 Release 附件发布。

[![SelfLift Avatar 工作流 Demo 预览](docs/assets/demo-selflift-avatar-preview.gif)](https://github.com/slmonker/selflift-Avatar/releases/download/v0.1.2-experimental/selflift-Avatar-demo-selflift-avatar.mp4)

**SelfLift Avatar 工作流 Demo：** [下载 / 查看完整 10.05 秒 MP4](https://github.com/slmonker/selflift-Avatar/releases/download/v0.1.2-experimental/selflift-Avatar-demo-selflift-avatar.mp4)

第二个预览同样是静音 GIF，点击即可获取带声音的完整视频。两个视频都只是维护者工作流的示例，不代表所有角色和音频都能达到相同的口型效果。仓库不包含模型权重。

## 安装

你需要已有可运行的 MiniMax H3 ComfyUI 环境，以及工作流本身需要的模型、VAE、LoRA 和可选 latent upscaler。本项目不新增依赖，也不包含模型权重。

### Git 安装

```bash
git clone https://github.com/slmonker/selflift-Avatar.git ComfyUI/custom_nodes/selflift-Avatar
```

更新已有 Git 安装：

```bash
cd ComfyUI/custom_nodes/selflift-Avatar
git pull --ff-only
```

更新后必须重启 ComfyUI 后端；仅刷新浏览器不会重新加载 Python 节点。

### ZIP 安装

从 [Releases 页面](https://github.com/slmonker/selflift-Avatar/releases) 下载最新 ZIP，将旧的 `selflift-Avatar` 文件夹备份到 `custom_nodes` 之外，用新版文件夹替换后重启 ComfyUI。不要在 `custom_nodes` 中同时放置两个本插件副本。

原版 `comfyui-SelfLift` 可以保留；本项目使用不同的节点 ID。

## 节点

| 节点 ID | 显示名称 |
| --- | --- |
| `SelfLiftAvatarH3Sampler` | selflift-Avatar Sampler (MiniMax H3) |
| `SelfLiftAvatarImageSampler` | selflift-Avatar Sampler (Image) |
| `SelfLiftAvatarH3TST` | selflift-Avatar H3 TST |

采样器使用标准的 `sampler` 和 `sigmas` 输入。请使用标准 Euler，并保持 `s_churn=0`。

## 音频驱动 H3 工作流

典型的输入音频连接方式：

```text
输入音频 → 裁剪到目标片段 → H3 音频 VAE 编码
                                  ↓
SolidMask(value=0) → SetLatentNoiseMask
                                  ↓
H3 视频 latent ───────────────→ 合并 AV latent
                                  ↓
                       selflift-Avatar H3 采样器
```

- 音频遮罩为 `0` 表示保留输入音频，为 `1` 表示允许生成音频，0～1 表示软约束。
- 编码音频、视频帧范围和最终输出 FPS 必须对应同一段素材。
- 如果最终视频节点直接使用原始音频，听到正确的声音并不能证明 H3 在生成画面时真正用它驱动了嘴部；需要检查生成的视频帧。
- 本采样器不是独立的后期口型同步工具。

## 高分辨率分块

打开 `highres_tiling=true` 后，插件会自动规划高分辨率空间分块。规划器根据可用工作区选择方向和块数；也可能选择 1 块，这表示当前不需要实际拆分。

### 当前支持的带遮罩分块模式

目前经过验证的带遮罩模式被有意限制为：

```text
视频遮罩：全 1 → 生成完整视频
音频遮罩：全 0 → 保留完整输入音频
```

每个视频块都会获得完整音频 latent 和完整 H3 音频条件。分块只发生在视频空间方向，音频不会按画面空间切分。

### 开启分块时仍会拒绝

- 视频遮罩中存在 0 或软值；
- 音频局部遮罩或软遮罩；
- 动态 `denoise_mask_function`；
- H3 分块路径与 ControlNet 组合。

这些工作流请关闭 `highres_tiling`。直接删除校验并不能让它们正确运行，因为对应遮罩还需要按 tile 裁剪、变换并对齐。

在控制台中看到以下日志即可确认规划器：

```text
[selflift-Avatar plan] automatic high-resolution tiling enabled
[SelfLift tiling plan] axis=W tiles=4
```

`axis=H` 或 `axis=W` 表示分块方向，`tiles=N` 表示自动选择的分块数量。

## 不使用分块时的遮罩支持

- 普通图像/视频 Tensor：BHW、BCHW、BCTHW；支持单通道或实际 latent 通道数。
- H3 `NestedTensor(video_mask, audio_mask)`：分别处理视频和音频遮罩。
- 与音视频 latent 展平尺寸严格匹配的 packed mask。
- 音频 T、BT、BST、BCST 布局；维度为 1 时可广播。
- 视频空间遮罩会缩放到 latent 网格；不会猜测音频时间含义，也不会自动重采样音频时间轴。
- 附加到音频上的恒定图像尺寸 `SolidMask`，例如 `[1,1,928,1664]`，会被识别为恒定音频约束，并广播到音频 latent。

遮罩语义：**0=保留，1=生成，0～1=软约束**。要产生有意义的保留效果，输入 latent 中必须确实包含要保留的内容。

## H3 原生音频条件

采样器会在采样前把打包后的遮罩传入 ComfyUI 原生 H3 条件路径，使用 H3 的 `audio_denoise_mask` 和输入侧 `scale_latent_inpaint`，而不只是模型预测后再恢复音频。高清阶段的带噪恢复状态和干净音频 anchor 分开处理，避免把残余采样噪声当成原始音频。

这修复的是遮罩和条件的传递问题，并不保证口型绝对准确。外部 H3 latent upscaler 包含时间卷积，而且不会直接接收目标音频；低分辨率前缀和高清修正步数也可能影响嘴部细节。

做对比时，请固定模型、音频片段、提示词、帧数、FPS 和 seed。一次只改变一个因素，对比原生基准、当前 SelfLift 路径、不同 `transition_step`，以及 `upscaler_model=none` 的 pixel-anchor 路径。

## 测试

使用 ComfyUI 对应的 Python 运行：

```bash
python tests/test_avatar.py
```

当前共有 34 项 CPU 测试，覆盖音视频打包、H3 音频尺度、全量/局部遮罩、恒定音频 SolidMask 转换、原生音频条件、两阶段采样、自动分块规划、分块拼接、每个分块获得完整音频、非法遮罩组合和无遮罩分块路径。测试不会加载大型 H3 权重，也不衡量最终视频的口型质量。

## 来源与许可

本项目基于 [facok/comfyui-SelfLift](https://github.com/facok/comfyui-SelfLift) 的本地快照修改。上游说明保存在 [`docs/upstream/`](docs/upstream/)，来源哈希记录在 [`PROVENANCE.json`](PROVENANCE.json)。

准备本项目时，上游仓库没有声明许可证。本项目没有代上游作者添加许可证，也不代表获得重新分发上游代码或模型权重的授权。论文、第三方代码和模型权重仍由各自作者负责。

## 回退

[v0.1.1-experimental](https://github.com/slmonker/selflift-Avatar/releases/tag/v0.1.1-experimental) 仍可用于对比。也可以在工作流中重新连接原版 SelfLift 节点。卸载时先停止 ComfyUI，再将本文件夹移出 `custom_nodes`。
