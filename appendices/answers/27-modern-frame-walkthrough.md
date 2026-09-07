# 第 27 章参考答案

[返回正文](../../chapters/27-modern-frame-walkthrough.md) · [全部答案](../exercise-answers.md)

本章答案沿用 B：Blendable、Nanite、VSM、软件 Lumen、TAA，硬件光追和 MegaLights 关闭。源码结论来自静态阅读，算例是假设输入，不是抓帧或显示测量。

## 题 1：主材质之前的卡片光照

`RenderLumenSceneLighting` 的接收表面是 Surface Cache 卡片 texel，使用卡片捕获的材质、法线、深度与相应位置。它可以使用已具备的卡片资源计算出射光，并不需要把本帧主视图的 P GBuffer 提前读取出来。本版构图确实先调用卡片光照，再调用主 Base Pass，见 [DeferredShadingRenderer.cpp:2899](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/DeferredShadingRenderer.cpp:2899)。

主 P 的 Screen Probe 收集与材质积分仍需其深度和材质输入。主表面、卡片表面、探针采样位置是不同计算位置；“先材质后光照”必须先说明是哪一套资源，而不是只比较函数名。

## 题 2：三种缓冲与两种着色选择

VisBuffer 保存 Nanite 当前胜出表面的深度和几何引用，供后续查询可见 Cluster、三角形及属性。Scene Depth 是普通网格与 Nanite 汇合后的主视图深度，供灯光定位、透明深度关系、屏幕追踪与后处理使用。Blendable GBuffer 保存编码后的材质属性，供延迟消费者解码；它既不是 VisBuffer，也不是最终颜色。

深度导出可以因平台能力走 Pixel Shader；这项选择不决定材质着色。Nanite 的材质仍由 `DispatchBasePass` 组织分桶与计算着色，`TBasePassCS` 对应 `MainCS`。参照 [导出能力](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/Nanite/NaniteShared.cpp:474)与[材质计算入口](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/Nanite/NaniteShading.cpp:1178)。普通与 Nanite 的更近深度还必须正确决定哪一份材质有效。

## 题 3：相机可见与光源可见

相机看不见的方块背面或离屏物体仍可能位于光源与接收点之间。VSM 必须按灯光视图与页面范围组织阴影投射者，不能只复制主相机最终可见三角形集合。Nanite 阴影使用 Shadows 管线与 DepthOnly 输出，参照 [VSM Nanite 上下文](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/VirtualShadowMaps/VirtualShadowMapArray.cpp:3808)。

Nanite 驻留页缓存几何数据；VSM 页缓存特定灯光投影区域的遮挡深度及相关状态。二者的预算、缺页、失效和更新原因不同。方向光 Receiver Mask 可使动态部分每帧 uncached，静态层仍可缓存，因此几何没有流送也不能证明没有阴影重画。[页更新条件](G:/UnrealEngineInstalled/UE_5.7/Engine/Shaders/Private/VirtualShadowMaps/VirtualShadowMapPhysicalPageManagement.usf:333)

## 题 4：剩余步骤与反射模式

继续查看 `AsyncLumenIndirectLightingOutputs.StepsLeft`，确认 ScreenProbeGather、Reflections、Composite 中哪些已完成、哪些留待后续调用。再跟踪输出纹理与后置合成，不能把每次调用都当作三项完整重算。主调度将普通 Lumen 合成放在 Lights 后，为适用异步工作保留重叠机会。[剩余步骤判断](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/IndirectLightRendering.cpp:1060)

软件 B 的 Lumen 反射依赖有效 Lumen GI；关闭 GI 后不能无条件保留同一软件路线。独立 Lumen 反射需要额外硬件追踪支持和相应 Hit Lighting 条件，属于第 25 章的另一配置。主 B 的 GI 与 Lumen 反射已在间接合成中接入时，传统反射阶段还会跳过对应镜面合成，避免重复。[镜面合成跳过条件](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/IndirectLightRendering.cpp:2044)

## 题 5：背景改变与实际显示

前景项 `0.35*S=(0.035,0.21,0.35)`。旧背景项 `0.65*(0.8,0.1,0.05)=(0.52,0.065,0.0325)`，所以旧 Q 为 `(0.555,0.275,0.3825)`。新背景项是 `(0.585,0.0975,0.052)`，所以新 Q 为 `(0.62,0.3075,0.402)`。

差值为 `(0.065,0.0325,0.0195)`，恰好是背景差值乘 0.65。Unlit 仅限制薄片自身的受光方式，不会冻结后方已完成的颜色，也不会跳过时间处理、曝光和显示输出。本例 S 是假设阶段颜色，不能直接当作实际 EmissiveStrength=300 的观测结果。

Present 是呈现链中的请求与调度边界，返回不证明显示器已经扫描到 Q。还要区分命令提交、GPU 完成、窗口合成、刷新与扫描位置；必须实际测量才能给出可见时刻。参照 [D3D12 Present](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/D3D12RHI/Private/D3D12Viewport.cpp:598)。
