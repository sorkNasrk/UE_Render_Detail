# 第 24 章理解检查题参考答案

[返回正文](../../chapters/24-lumen-software-tracing.md) · [返回答案目录](../exercise-answers.md)

## 1. 三种表示的职责

Mesh SDF 和 Global SDF 是几何距离近似，用于判断射线何处命中；Mesh SDF 精度较高，Global SDF 覆盖范围大但分辨率较低。Surface Cache 是卡片表面的材质、法线、发光和光照图集，命中后从中取得 Radiance。SDF 不含材质颜色，Surface Cache 也不能单独告诉射线在哪里撞到表面。[软件追踪命中后采样](G:/UnrealEngineInstalled/UE_5.7/Engine/Shaders/Private/Lumen/LumenSoftwareRayTracing.ush:588)

## 2. 探针射线顺序

满足条件时先运行屏幕深度/HZB 追踪；仍需处理的射线压缩后，若细节追踪允许且有网格距离场对象则执行 Mesh SDF Compute，再由 Global SDF 等后续阶段处理剩余部分。世界命中后把位置映射到卡片并采样 Surface Cache，结果写入 `TraceRadiance`，经过滤波、方向积分和时域重投影，生成 `DiffuseIndirect` 或 `RoughSpecularIndirect`。不是每条射线必跑三次，也不是只要一个 CVar=1 就满足所有运行条件。[屏幕追踪条件](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/Lumen/LumenScreenProbeTracing.cpp:688)[探针调度](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/Lumen/LumenScreenProbeGather.cpp:2576)

## 3. 更新因子 32 与 64

它们是调度 Surface Cache 图集更新规模的初始因子，不是“每 32 或 64 帧更新一次”。源码把直接光照和间接光照分别传入 `SetLightingUpdateAtlasSize`，再按视锥距离、页面年龄和预算挑选 tile；因此移动光源后的变化可能分多帧传播。[更新上下文](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/Lumen/LumenSceneLighting.cpp:39)[图集尺寸设置](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/Lumen/LumenSceneLighting.cpp:572)

忽略 tile 对齐和最小页要求，两个边长各乘 `1/sqrt(F)`，面积便约为 `1/F`；注册初值仍可被画质和运行时配置覆盖。光照传播中还有 Radiosity、世界 Radiance Cache、Screen Probe 历史，不能仅凭这两个值推算最终像素的固定延迟。

## 4. M 为什么需要反射专用路线

漫反射是宽角度、低频积分，Screen Probe 的 `DiffuseIndirect` 经过下采样和插值后不保留清晰镜面方向。金属球 M 的反射要依据法线、粗糙度和材质闭包生成专门射线，再执行 Resolve、时域和空间去噪。UE 在 `ReflectionsMethod == Lumen` 时单独调用 `RenderLumenReflections`；其 Shader 还拥有自己的 Mesh/Global SDF permutation。[反射调用](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/IndirectLightRendering.cpp:1088)[反射入口](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/Lumen/LumenReflections.cpp:1154)

## 5. Q 为什么不写普通不透明 GBuffer

主不透明 GBuffer 在 Q 位置记录后方方块；薄片在所选透明阶段与背景混合。把薄片改成普通不透明物体会改变深度和材质语义，并非“让同一个透明材质正确使用 Lumen”。本案例薄片是 Unlit，自身不接收漫反射 GI；它覆盖的方块背景有了 GI，Q 仍可随之变化。适用受光透明材质的体积/前层反射属于额外分支，不应直接套到本例。

教学算例中，若假设薄片阶段颜色为 `(0.1,0.6,1)`、Opacity 为 `0.35`、背景为 `(0.8,0.1,0.05)`，则 `Q=(0.555,0.275,0.3825)`。这不是实际 EmissiveStrength=300、预曝光和色调映射共同作用后的测量值。
