# 第 14 章答案：Base Pass、GBuffer 与贴花

[返回本章](../../chapters/14-base-pass-gbuffer-decals.md) · [返回目录](../../README.md)

## 题 1

GBuffer 保存的是表面属性编码，例如法线、Base Color、Metallic、Roughness 和 Shading Model ID。之后的不透明光照会读取它们，SSR、反射、曝光、色调映射和输出编码还可能继续改变颜色。因此 P 的 GBuffer C 不是最终显示 RGB；看到红色属性也不能推出显示器一定显示同样的红。

## 题 2

按题设的教学传统编码：

```text
Normal (0,0,1) -> EncodeNormal = (0.5,0.5,1.0)，写入 GBuffer A 的 RGB
Metallic 0.0   -> GBuffer B.R
Roughness 0.4  -> GBuffer B.B
```

这些数值是题设，忽略真实材质、格式量化、AO 和可选布局字段；它们不是运行采样值。默认传统法线目标也不是题设的四通道 8 bit。预曝光作用于适用的颜色输出，不应把法线或 Roughness 也按曝光缩放。真实工程应核对 GBuffer Layout、生成的 `EncodeGBufferToMRT` 及对应解码，不能假设所有平台都只有 A/B/C 三张固定 RGBA8。

## 题 3

DBuffer Decal 在 Base Pass 前把投影覆盖的 Base Color、Normal、Roughness 等写入 DBuffer。Base Pass 读取 Mask 和 DBuffer 数据，再把它们合入当前材质输入，随后编码 GBuffer。GBuffer Decal 在 Base Pass 后直接修改已存在的 GBuffer，供光照读取。Emissive Decal 在适用的后续阶段贡献颜色，不等于修改 Base Color。

因此 DBuffer 能影响 Base Pass，是因为它提供了 Base Pass 明确读取的资源和 Mask；它不是自动叠到最终 Scene Color 的贴图。

普通属性贴花的阶段由平台 DBuffer 条件和材质输出等生成：使用 DBuffer 时进入 BeforeBasePass，否则适用属性路径选择 BeforeLighting；不是任意切换一个 Render Stage 选项。在 DBuffer 条件下，材质还输出 Emissive 可以增加额外的发光阶段。配置 A 与 Substrate 也应按各自宏、布局和绑定解释。

传统颜色应用为 `BaseColor = 原材质颜色 * 剩余权重 + 预乘贴花颜色`。剩余权重为 1 表示保留原材质，为 0 表示完全覆盖。法线混合后还需要归一化，不能把所有属性都当成最终颜色的 Alpha Blend。

## 题 4

```text
像素数 = 1920 × 1080 = 2,073,600
每像素 = 4 个目标 × 4 byte = 16 byte
总量 = 33,177,600 byte
      ≈ 31.64 MiB（除以 1024²）
```

这是忽略压缩、对齐、其他 MRT、深度、Velocity、DBuffer、读写和过度绘制的理想单次写入量。GPU 时间还受带宽、缓存、Shader、覆盖率、同步和其他阶段影响；实际显存流量也可能因压缩和重读写远离该数值。

## 题 5

P 的不透明方块通过 Base Pass 深度测试，若命中 DBuffer 投影，Base Color、Normal 或 Roughness 等可在写 GBuffer 前被修改。Q 的蓝片是 Translucent，普通主不透明 Prepass/Base Pass 不自动为它写 GBuffer；Q 的主 GBuffer 因而仍描述方块背景，薄片的颜色随后与背景经混合或合成组合。普通无折射 Alpha 混合不要求薄片的像素 Shader 自己采样背景。

本场景蓝片还被 `!MATERIALBLENDING_ANY_TRANSLUCENT` 条件排除在常规自动 DBuffer 应用之外；能被贴花修改的是 Q 的不透明背景，并非蓝片本身。

把薄片改成 Masked 改变的是 Blend Mode，Material Domain 可以仍为 Surface。必须重新检查 Opacity Mask/clip、Early-Z 是否包含 Masked、Base Pass 深度比较与写入、DBuffer 响应以及是否使用 Custom Depth。薄片仍可能保持 Unlit，需要检查它的特殊输出，不能自动按 Default Lit 通道解释。不能只把 Opacity 数值当成 Blend Mode 的替换。
