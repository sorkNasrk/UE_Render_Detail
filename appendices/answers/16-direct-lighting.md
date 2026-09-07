# 第 16 章理解检查参考答案

[返回第 16 章](../../chapters/16-direct-lighting.md) · [返回答案目录](../exercise-answers.md)

## 题 1：方向光与局部光

方向光的方向和强度在观察范围内可近似统一，因此可以用全屏矩形让每个候选像素执行一次方向光判断。点光源和聚光灯的影响范围有限，使用光源体积可减少无关屏幕像素；Pixel Shader 仍要用恢复的世界位置计算距离、方向和衰减。全屏 Pass 并不保证每个像素执行完整 BRDF，天空、Unlit 或不适用 ShadingModelID 会被跳过，阴影和分支也会改变工作。

## 题 2：排序范围与 clustered 兼容性

`GatherAndSortLights` 用排序键记录类型、阴影、Light Function、Lighting Channel 和聚类支持能力，再扫描出 `ClusteredSupportedEnd`、`UnbatchedLightStart` 等边界。`RenderLights` 可把兼容范围交给 `AddClusteredDeferredShadingPass`，其余灯走批量或逐灯路径。带阴影的点光源可能需要屏幕阴影遮罩、额外采样或特殊功能，而本版排序条件会把这类灯标为不适合最简单的 clustered 路径；具体是否由 VSM 或其他路径支持还需结合配置。

本版为 `bShadowedLightsInClustered` 合并聚类启用、VSM One Pass Projection 和有效 VSM Array 条件；A 使用常规阴影且关闭聚类，不满足它。旧聚类变量名在 5.7 已弃用，读取源码或做独立实验时应查询正文给出的实际开关，不能只复制旧教程的设置。

## 题 3：三种输入

GBuffer 保存法线、材质颜色、粗糙度、金属度和 ShadingModelID 等表面描述；Scene Depth 用屏幕坐标恢复世界位置，并帮助局部光体积的深度测试；Shadow Mask 表示光线从光源到表面是否被遮挡。没有深度时，Shader 难以从当前像素恢复可靠位置，也无法正确计算点光源距离、光源方向和局部几何约束。只凭 GBuffer 不能补出这些观察空间信息。

## 题 4：数值

```text
(0.8,0.1,0.05) / π * 0.5 * 0.8
= (0.8,0.1,0.05) * 0.4 / π
≈ (0.101859, 0.012732, 0.006366)
```

这是忽略镜面、光色、预曝光、能量守恒和色调映射的线性教学值。它不是显示器 RGB，也不能作为 UE 的实测像素值。

## 题 5：透明 Q 的路线

配置 A 的薄片是 Translucent、Unlit。后方方块先写入主不透明深度和 GBuffer，再接受直接光照；薄片在透明阶段产生自己的颜色和 Opacity，并与已经形成的背景 Scene Color 混合。若把薄片当成普通不透明 GBuffer 表面，会改变深度、排序和背景关系，也会错误地把它套到传统不透明延迟光照的材质模型上。特殊透明材质可以有其他资源和 Pass，但不能把它们当作本章案例的默认路径。

[返回第 16 章](../../chapters/16-direct-lighting.md)
