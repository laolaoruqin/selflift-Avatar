# selflift-Avatar

用于 ComfyUI / MiniMax H3 的 **SelfLift 独立实验分支**，重点适配音视频双流遮罩和原始音频保留。与原版节点使用不同的注册 ID，可以并存。

> **实验版本，非官方项目。** 已通过 26 项 CPU 测试，但未完成真实 H3 大模型视频生成验证。测试通过不代表实际成片效果、性能或其他插件组合已验证。

## 安装

需要一套已经能够运行 MiniMax H3 的 ComfyUI，以及该工作流本来需要的模型、VAE 和可选 latent upscaler。此项目不包含模型权重，也不安装新的依赖。

在 `ComfyUI/custom_nodes` 下执行（私有仓库需要 GitHub 访问权限）：

```bash
git clone https://github.com/slmonker/selflift-Avatar.git
```

或下载 Release 中的 ZIP，将其中的 `selflift-Avatar` 文件夹放入 `ComfyUI/custom_nodes`。之后重启 ComfyUI 后端并刷新页面。

## 使用

1. 搜索 `selflift-Avatar`，添加 **selflift-Avatar Sampler (MiniMax H3)**。
2. 把原 SelfLift 采样器的输入和输出改接到新节点。原节点和工作流不会自动替换。
3. 使用标准 **Euler**；带 `noise_mask` 时设置 **`highres_tiling=false`**。
4. 建议复制工作流，以固定 seed、短片段、相同其他参数进行对比。
5. 如果采用外部 H3 latent upscaler，可保留 `rho=0`；`upscaler_model=none` 时，H3 节点仍要求 `rho>0`。

| 注册 ID | 显示名称 |
| --- | --- |
| `SelfLiftAvatarH3Sampler` | selflift-Avatar Sampler (MiniMax H3) |
| `SelfLiftAvatarImageSampler` | selflift-Avatar Sampler (Image) |
| `SelfLiftAvatarH3TST` | selflift-Avatar H3 TST |

音频遮罩仍通过 latent 字典的 `noise_mask` 提供，没有新增单独的 mask 输入端口。

## 遮罩支持

- 普通视频/图片 Tensor：BHW、BCHW、BCTHW；通道为 1 或实际 latent 通道数。
- H3 `NestedTensor(video_mask, audio_mask)`：分别解析音视频，保留多通道权重。
- 严格匹配音视频展平尺寸的 `[B,1,N]` 遮罩。
- 音频 T、BT、BST、BCST 布局；维度为 1 可广播。
- 视频空间尺寸不匹配时进行双线性缩放；不猜测时间轴、不对音频时间轴自动重采样。
- 普通 Tensor 默认只约束视频，缺省音频遮罩按全 1 处理。

**0=保留原内容，1=生成，0~1=软约束。** 必须有对应的已初始化 latent 才能保留实际内容。

### SolidMask 加到音频的兼容处理

`SolidMask → SetLatentNoiseMask → 合并音视频 latent` 可能给音频附加图像尺寸的恒定遮罩，例如 `[1,1,928,1664]` 全零。

本版本确认每个批次/通道在空间上严格恒定后，压缩为 `[B,C,1,1]` 广播到音频 latent。全零保留整段输入音频，全一生成音频，恒定软权重也不变。非恒定图像图案不擅自解释为音频时间遮罩。

## 采样约束与修复

- 低清和高清阶段分别应用对应视频 mask；音频 mask 不随空间分辨率缩放。
- 约束原内容时调用实际采样模型的 `process_latent_in`，包含 H3 的音频尺度转换。
- 返回前仅对 `mask==0` 的位置恢复原始 latent，不重复混合软遮罩。
- 修复纯像素 anchor 模式（`rho=w_min=w_max=1`）带 mask 时的 None 运算问题。
- 报错包含实际 mask 类型、各流形状和 latent 形状。

## 已知限制

- 带 mask 时不支持 `highres_tiling`。
- 只应用静态输入遮罩，不执行上游 `denoise_mask_function` 的动态调度；检测到时会警告。
- `model_hires` 必须使用兼容架构、latent 格式和采样尺度设置；跨架构/尺度切换未验证。
- 保留的是输出 latent 区域，不承诺 VAE 解码后像素或波形逐点相同。
- TST、分块和外部 upscaler 继承上游的实验限制；没有重新验证这些功能的所有组合。

## 测试

从安装在 ComfyUI 下的插件目录运行，使用 ComfyUI 对应的 Python：

```bash
python tests/test_avatar.py
```

26 项 CPU 测试包括音视频打包、真实 H3 音频尺度转换、全零/全一遮罩、前段保留、软遮罩、多通道、恒定图像尺寸音频遮罩和两阶段流程。两阶段测试采用合成 denoiser 与模拟 VAE lift，不加载模型权重或生成视频。

## 来源与许可说明

基于 [facok/comfyui-SelfLift](https://github.com/facok/comfyui-SelfLift) 的本地副本修改，保留[上游说明](docs/upstream/README.md)和[中文说明](docs/upstream/README_CN.md)。来源哈希见 `PROVENANCE.json`。

上游包含 SelfLift、TST 和可选 MiniMax H3 latent upscaler 的参考与致谢；这些研究和权重不由本项目声明所有权。本次上传检查时，上游未声明许可证；本项目未擅自新增开源许可证，不能仅凭本仓库存在推断获得原代码或模型的再分发授权。

## 回退

工作流换回原版节点即可；卸载时关闭 ComfyUI，将本文件夹移出 `custom_nodes`。本项目不会修改原版插件或 ComfyUI 核心文件。
