# 附录：教学配置与观察前提

[返回总目录](../README.md) · [第一章：场景与观察练习](../chapters/01-from-scene-to-pixel.md)

> **适用版本：UE 5.7.4，CL 51494982；Windows、D3D12、SM6、桌面延迟渲染。** 本附录的设置名称、变量映射和相关条件已按本地源码核对；尚未启动编辑器创建场景、切换配置或采集运行画面。因此下文是可复现的操作规范与预期现象，不是已经完成的运行测试报告。

## A.1 为什么先固定配置

同一个 UE 项目，改变材质系统、阴影算法或间接光照方法后，可能增加、替换或跳过整组渲染工作。先约定配置，才能准确回答“这一帧中为什么出现这个阶段”。

本教材使用两套配置。**A 用来建立基础认知，B 用来解释现代功能如何改变已有流程。** A 是人为选择的教学配置，不是 UE 5.7 新项目的默认配置。A、B 都使用桌面延迟渲染；“关闭 Lumen”不等于“改用前向渲染”。

需要分清三种值：

| 名称 | 含义 | 为什么可能不同 |
|---|---|---|
| 控制变量注册初值 | C++ 注册一个控制变量时给出的数值 | 为兼容旧项目保留的初值，不一定对应新项目 |
| 项目配置值 | 项目生成器、模板或 Project Settings 写入配置的数值 | 不同模板、旧项目升级和平台设置可能不同 |
| 当前运行值／当前视图实际路径 | 当前进程的控制变量，以及结合后处理、平台能力、资源与显示标志作出的最终选择 | 配置优先级、画质设置、命令行、视口覆盖和功能支持条件会继续影响结果 |

**源码已确认：** `r.Substrate` 的注册初值是 `0`，`r.Substrate.ProjectGBufferFormat` 的注册初值是 `1`；但新项目生成逻辑会写入 `r.Substrate=True` 与 `r.Substrate.ProjectGBufferFormat=0`，并采用不覆盖已有模板值的方式。源码注释说明常规模板采用 Blendable GBuffer，高级模板可以采用 Adaptive GBuffer。因此不能得出“UE 5.7 默认不开 Substrate”或“所有 UE 5.7 项目都使用同一种 GBuffer”这样的结论。

来源：[RenderUtils.cpp：Substrate 注册及兼容说明](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/RenderCore/Private/RenderUtils.cpp:1940)，[GameProjectUtils.cpp：新项目 Substrate 配置](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Editor/GameProjectGeneration/Private/GameProjectUtils.cpp:199)。版本依据：[Build.version](G:/UnrealEngineInstalled/UE_5.7/Engine/Build/Build.version:1)。

## A.2 平台和渲染方法的共同前提

设置入口采用英文显示名，便于在不同编辑器语言下搜索。下面的 `Rendering > ...` 均以 **Edit > Project Settings > Engine > Rendering** 为起点；Windows 平台项位于 **Project Settings > Platforms > Windows**。分类与显示名来自设置类声明，实际界面可能把某些项折叠到高级设置中，可搜索表中的英文名。

| 设置 | A 与 B 的共同目标 | 生效与核验方式 |
|---|---|---|
| Platforms > Windows > Default RHI | DirectX 12 | `DefaultGraphicsRHI=DefaultGraphicsRHI_DX12`，重启编辑器。它是配置属性，不是可在运行时切换的 `r.*` 控制变量 |
| Platforms > Windows > D3D12 Targeted Shader Formats | 包含 SM6；本教材的专用练习项目只选择 SM6 | `D3D12TargetedShaderFormats` 包含 `PCD3D_SM6`。重启并等待目标平台 Shader 编译完成 |
| Rendering > Forward Renderer > Forward Shading | 关闭 | `r.ForwardShading=0`。设置声明要求重启；渲染路径变化会涉及不同 Shader 组合 |
| Rendering > Misc Lighting > Allow Static Lighting | 关闭 | `r.AllowStaticLighting=0`。设置声明要求重启；等待 Shader 编译完成 |
| Rendering > Hardware Ray Tracing > Support Hardware Ray Tracing | 关闭 | `r.RayTracing=0`。设置声明要求重启；硬件光追实验留到第 25 章 |
| Rendering > Lumen > Use Hardware Ray Tracing when available | 关闭 | `r.Lumen.HardwareRayTracing=0`。项目硬件光追支持关闭时 UI 可能不可编辑；这时硬件路径已经被更外层条件关闭，核验配置即可 |
| Rendering > Direct Lighting > Ray Traced Shadows | 关闭 | `r.RayTracing.Shadows=0`；各光源不强制启用光追阴影 |
| Rendering > Direct Lighting > MegaLights | 关闭 | `r.MegaLights.EnableForProject=0`，并在观察会话设置 `r.MegaLights.Allowed=0`，确保后处理覆盖也不会启用它 |

SM6 是 Shader Model 6，即 Shader 编译与执行所依据的一组能力要求；它不是分辨率，也不等于启用了硬件光追。选择 D3D12／SM6 后仍须由显卡、驱动和操作系统满足所用功能的要求。若启动日志显示回退到其他 RHI，或功能可视化报告不支持，应先解决平台前提，不能把回退画面当成 B 的验证结果。

**源码已确认：** Windows 两个属性具有 `ConfigRestartRequired` 标记。新建 Windows 项目生成代码显式选择 DX12，并清空继承的 D3D12 Shader 格式列表后加入 SM6。只在一个旧项目配置片段里看到 SM5，不足以断言新项目运行于 SM5。

来源：[WindowsTargetSettings.h：Default RHI 与 Shader 格式](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Developer/Windows/WindowsTargetPlatformSettings/Classes/WindowsTargetSettings.h:50)，[GameProjectUtils.cpp：Windows 默认设置](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Editor/GameProjectGeneration/Private/GameProjectUtils.cpp:2137)，[RendererSettings.h：硬件光追支持](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Engine/Classes/Engine/RendererSettings.h:639)，[静态光照与前向渲染](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Engine/Classes/Engine/RendererSettings.h:686)。

## A.3 A、B 配置矩阵

表中“可运行时调整”只说明这个开关不是本处声明的重启型项目开关，**不保证尚未生成的资源或缺少的 Shader 可以凭空出现**。从 A 完整切换到 B 包含重启型修改，应统一重启、等候资源准备，再开始比较。

| 功能与设置位置 | 配置 A：基础主线 | 配置 B：现代对照 | 重启、编译与使用条件 |
|---|---|---|---|
| Substrate > Substrate materials | 关闭；`r.Substrate=0` | 开启；`r.Substrate=1` | 重启型项目设置；改变材质编译路径，需要等待相关 Shader 编译。不要通过运行时控制台当作即时显示开关 |
| Substrate > Substrate GBuffer Format (Project) | 不使用 Substrate，此项不决定 A 的 GBuffer | Blendable GBuffer；`r.Substrate.ProjectGBufferFormat=0` | 重启型项目设置；`0` 是 Blendable，`1` 是 Adaptive。不得把 B 的布局直接等同于 A |
| Nanite > Nanite | 项目支持关闭；`r.Nanite.ProjectEnabled=0` | 项目支持开启；`r.Nanite.ProjectEnabled=1` | 重启型、只读项目开关；源码明确它影响 Shader 排列组合，改变会重新编译 Shader |
| Nanite 运行时开关 | `r.Nanite=0` | `r.Nanite=1` | 可运行时调整并重建组件渲染状态；前提是项目、平台和网格资源本来支持 Nanite |
| 静态网格资产 > Nanite Settings > Enable Nanite Support | 地面、立方体、球均不启用 | 对适用的不透明静态网格启用并应用构建 | 属于资产构建；需要生成 Nanite 数据。透明薄片保持普通网格，不纳入 Nanite 对照 |
| Global Illumination > Dynamic Global Illumination Method | None；`r.DynamicGlobalIlluminationMethod=0` | Lumen；`r.DynamicGlobalIlluminationMethod=1` | 方法可按项目或视图选择；B 还需要软件追踪的距离场。None 只表示不选动态 GI，单独这一项并不能关闭已有烘焙数据 |
| Reflections > Reflection Method | Screen Space；`r.ReflectionMethod=2` | Lumen；`r.ReflectionMethod=1` | 方法可以被后处理覆盖；A 需要 SSR 质量和强度未被关闭，B 需要 Lumen 运行条件成立 |
| Software Ray Tracing > Generate Mesh Distance Fields | 关闭；`r.GenerateMeshDistanceFields=0` | 开启；`r.GenerateMeshDistanceFields=1` | 设置声明要求重启；生成静态网格距离场会增加资产构建时间、内存与磁盘用量。这主要是几何派生数据的构建，不应统称为 Shader 编译 |
| Lumen > Software Ray Tracing Mode | 无 Lumen，本项不参与主线 | Detail Tracing；`r.Lumen.TraceMeshSDFs=1` | B 在距离场已具备时选择；`0` 是 Global Tracing。软件追踪不是“只用当前屏幕”，也不是“由 CPU 逐条追踪光线” |
| Direct Lighting > Shadow Map Method | Shadow Maps；`r.Shadow.Virtual.Enable=0` | Virtual Shadow Maps；`r.Shadow.Virtual.Enable=1` | 该控制变量可运行时调整，源码回调重建组件渲染状态；B 仍须满足平台支持。完整 A／B 切换按重启流程执行 |
| Default Settings > Anti-Aliasing Method | Temporal Anti-Aliasing；`r.AntiAliasingMethod=2` | 先保持 Temporal Anti-Aliasing；`r.AntiAliasingMethod=2` | 具备对应 Shader 后可调整。切换后历史数据会变化，等待画面稳定再观察 |
| 第 19 章的 TSR 单项对照 | 不作为 A 首次观察配置 | 只将 `r.AntiAliasingMethod=4`；其他项保持 B | 先在 100% 屏幕比例比较算法，再单独降低屏幕比例解释时间超分辨率；不要一次更改算法与分辨率并把全部差异归给 TSR |

**为什么 Nanite 需要三层条件？** `r.Nanite.ProjectEnabled` 决定是否为项目提供这种能力，网格资产的 Nanite 设置决定是否生成这种表示，`r.Nanite` 决定当前是否允许使用它。项目支持为 `0` 时只输入 `r.Nanite 1`，不会补齐项目 Shader 与资产数据；反过来，运行时关闭也不等于从项目中移除了 Nanite 的所有构建成本。

本教材 B 用少量简单网格来识别路径，并不据此证明 Nanite 一定更快。小场景、低面数和现代功能附加工作可能使总开销上升；性能结论需要在相同硬件与代表性场景中另行测量。

**源码已确认：** `r.Nanite.ProjectEnabled` 带 `ECVF_ReadOnly`；`r.Nanite` 的修改回调使用 `FGlobalComponentRecreateRenderStateContext`；`UseNanite` 还会合并运行时支持条件。`r.Shadow.Virtual.Enable` 也有重建渲染状态的回调，不能把热切换成本当成稳定每帧成本。

矩阵来源：

- [RendererSettings.h：GI、反射、Lumen 与阴影的设置映射](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Engine/Classes/Engine/RendererSettings.h:568)，[软件距离场与 Nanite 项目设置](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Engine/Classes/Engine/RendererSettings.h:666)，[抗锯齿](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Engine/Classes/Engine/RendererSettings.h:828)，[Substrate 设置](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Engine/Classes/Engine/RendererSettings.h:1207)。
- [EngineTypes.h：GI 与反射枚举数值](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Engine/Classes/Engine/EngineTypes.h:449)，[Substrate 格式枚举](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Engine/Classes/Engine/EngineTypes.h:778)，[RendererSettings.h：软件追踪枚举](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Engine/Classes/Engine/RendererSettings.h:260)。
- [RenderUtils.cpp：Nanite 项目开关](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/RenderCore/Private/RenderUtils.cpp:29)，[UseNanite 的合并判断](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/RenderCore/Private/RenderUtils.cpp:1414)，[StaticMeshSceneProxy.cpp：运行时开关与回调](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Engine/Private/StaticMeshSceneProxy.cpp:108)。
- [VirtualShadowMapArray.cpp：VSM 开关与回调](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/VirtualShadowMaps/VirtualShadowMapArray.cpp:84)，[SceneView.cpp：抗锯齿枚举说明](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Engine/Private/SceneView.cpp:219)。

### A.3.1 从第 13 章开始固定深度观察条件

配置 A 补充以下选择，便于沿一条明确的完整深度主线读到 Base Pass。它们是教材的观察选择，不能替代项目运行值。第 13 章同时解释其他模式及其执行条件。

| Rendering 下的设置 | 教学值 | 生效与限制 |
|---|---|---|
| Misc Lighting > DBuffer Decals | 开启，`r.DBuffer=1`；先不添加贴花 | 项目设置要求重启；等待 Shader 编译。适用 DBuffer 支持本身就可强制完整预通道 |
| Optimizations > Early Z-pass | 使用默认政策，`r.EarlyZPass=3` | CVar 帮助明确不能运行时修改。保存项目后统一重启；功能条件可覆盖这一请求 |
| Optimizations > Mask material only in early Z-pass | 关闭，`r.EarlyZPassOnlyMaterialMasking=0` | 重启型设置，影响相关材质 Shader；不要把只读变量当热切换开关 |
| Optimizations > Velocity Pass | Write during base pass，`r.VelocityOutputPass=1` | 重启型设置并等待 Shader 编译；避免先进入“深度与速度一起补齐”的另一条早期路径 |

**[源码已确认]**设置映射和重启说明见 [RendererSettings.h](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Engine/Classes/Engine/RendererSettings.h:886)。`r.EarlyZPass` 的帮助和政策值见 [RendererScene.cpp](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/RendererScene.cpp:137)。完整深度的合并条件、`DepthPassCanOutputVelocity`、Base Pass 深度访问与比较函数的区别，见 [第 13 章](../chapters/13-depth-prepass-hzb.md)。

观察会话还应查询 `r.HZBOcclusion`、`r.AllowOcclusionQueries`、`r.SceneDepthHZBAsyncCompute`。不在本附录强制把它们全部设为 1：CPU primitive 遮挡、GPU 实例筛选、HZB 构建与消费者的条件不同。SSR 已经可以要求 HZB，不能用 `r.HZBOcclusion=0` 推出不存在 HZB。没有实际抓帧时，不将这些条件检查标为运行验证。

### A.3.2 MegaLights 为什么单独关闭

`r.MegaLights.EnableForProject` 是项目默认请求，后处理可以覆盖；`r.MegaLights.Allowed` 是许可门槛。为保证基础直接光照主线不会被这条路径替换，A、B 都将前者设为 `0`，并在观察会话将后者设为 `0`。本地源码使用的名称不是 `r.MegaLights.Enable`。

MegaLights 使用随机采样组织直接光照，和 Lumen 间接光照并非同一功能。它与硬件光追、平台支持、每灯设置和后处理的组合留到专章按实际条件核对；这里不把设置提示里对支持范围的简述当作所有内部实验分支的完整规范。

来源：[MegaLights.cpp：两个开关的注册](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/MegaLights/MegaLights.cpp:13)，[IsRequested：后处理与许可门槛](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/MegaLights/MegaLights.cpp:480)。

### A.3.3 第 16～17 章的直接光照与屏幕空间效果

为了固定一条可阅读的分支，第 16 章在 A、B 均保持 `r.UseClusteredDeferredShading_ToBeRemoved=0`。这是本版仍存在且标为待移除的运行开关；不使用旧名称推断未来支持。两盏灯保留 Cast Shadows，Contact Shadow Length 为 0，不使用 Light Function、IES 或非默认 Lighting Channel。具体排序与条件见 [第 16 章](../chapters/16-direct-lighting.md)。

下表只补充 **配置 A** 的第 17 章观察，不覆盖 B 的 Lumen 方法。选择画质档后再应用会话值，并核对最终 View 的后处理设置。

| 项目 | 固定值 | 生效条件与用途 |
|---|---|---|
| AO 方法／实现／层数 | `r.AmbientOcclusion.Method=0`、`r.AmbientOcclusion.Compute=0`、`r.AmbientOcclusionLevels=1` | 运行时选择传统 Pixel Shader SSAO，一级全分辨率；不用异步 AO 推断 A 的位置 |
| PPV Ambient Occlusion | Intensity=1、Radius=100、Radius in WorldSpace 开启、Static Fraction=1 | 启用对应覆盖；世界空间半径按 UE 厘米解释。静态光照关闭仍须查看 AO 合成的实际颜色输入 |
| SSR 方法／质量 | `r.ReflectionMethod=2`、`r.SSR.Quality=3`；PPV Screen Space，Intensity=100、Quality=100、Max Roughness=0.8 | View 方法、显示标志、质量、历史与材质粗糙度共同约束实际执行 |
| SSR 实现 | `r.SSR.Compute=0`、`r.SSR.TiledComposite=0`、`r.SSR.Stencil=0` | 固定普通屏幕 Pixel Shader 主线，优化分支单独对照 |
| SSR 时间处理 | `r.SSR.Temporal=0`、`r.SSR.ExperimentalDenoiser=0` | 配合 TAA 时不额外请求独立 SSR 时间滤波；仍可使用 TAA 颜色历史作为追踪颜色输入 |

这些会话开关不是要求重启的项目编译开关；仍须具备已编译 Shader 和有效资源。PPV 与控制变量的设置方式、源码注册和最终选择位置见 [第 17 章](../chapters/17-indirect-ao-reflections.md)。**[尚未验证]**没有运行教材工程确认实际 Pass 列表或可见差异，预期不能改写成实测结果。

## A.4 贯穿场景的观察约定

场景搭建与摆放见 [第一章的观察练习](../chapters/01-from-scene-to-pixel.md#1131-准备一个不含额外照明的关卡)。本附录只规定影响解释的环境条件。

| 项目 | 初次观察时固定为 | 目的与注意事项 |
|---|---|---|
| 内容 | 普通 Camera、地面、不透明立方体、金属球、透明薄片、方向光和点光源 | A 不放 Sky Light、Reflection Capture、平面反射、Sky Atmosphere、云或雾；B 最初也复用该场景 |
| 光源移动性 | 两盏光均 Movable，开启 Cast Shadows；A 方向光的 Dynamic Shadow Distance MovableLight 为 `2000 cm` | 排除静态与 Stationary 光源的烘焙／混合路径；保证本例常规动态阴影距离覆盖案例物体 |
| 不透明材质 | A 使用传统 Default Lit；粗糙度与金属度按第一章固定 | 不添加复杂层材质、位移或自定义后处理材质 |
| 透明薄片 | Unlit、Translucent、Two Sided；`Tint=(0.1, 0.6, 1.0)` 乘 `EmissiveStrength=300` 接到 Emissive Color，即 `(30, 180, 300)`；Opacity 为 `0.35`，不启用折射 | Unlit 仍受曝光影响，强度 300 是尚未实测的固定观察起点；手算混色使用另行假设的阶段输入。这是教学表面，不是物理玻璃 |
| 烘焙 | 不使用；项目 Allow Static Lighting 已关闭 | 从空场景开始，避免把旧关卡里已有的光照数据、天空光或反射捕获误当作关闭 GI 后仍产生的反弹光 |
| 视图 | 单摄像机、非立体、非分屏 | 输出目标 1280 × 720，16:9；摄像机取景保持一致 |
| 画质 | A、B 首次比较均使用 Epic，并记录实际值 | 画质档会修改多项控制变量；选择档位后再应用本附录的观察值，不在比较中随意换档 |
| 后处理 | 一个启用的全局 Post Process Volume | Infinite Extent (Unbound) 开启、Blend Weight 为 `1`；不叠加其他体积，相机自定义后处理 Blend Weight 为 `0` |

PPV 是 Post Process Volume，即后处理体积。它给视图提供曝光、反射等设置；Unbound 表示不要求摄像机身处体积边界内。**参数左侧的覆盖勾选框也必须勾选**，否则填入的值可能不参与最终设置合成。

### A.4.1 固定曝光

曝光决定光照计算产生的亮度如何映射到最终画面。在自动曝光下，把摄像机转向暗处会改变整幅图像的明暗，容易被误解为某个光源或材质改变了。因此本教材先使用 Manual，即手动曝光。

在上述唯一 PPV 中按下表设置，各项启用覆盖：

| 位置／字段 | 目标值 | 说明 |
|---|---|---|
| Lens > Exposure > Metering Mode | Manual | `AutoExposureMethod=AEM_Manual`，不使用随场景亮度变化的自动测光 |
| Lens > Exposure > Apply Physical Camera Exposure | 开启 | 在 Manual 下根据快门、ISO 和光圈计算物理相机曝光 |
| Lens > Camera > ISO | `100` | 感光度 |
| Lens > Camera > Shutter Speed (1/s) | `60` | 字段单位是每秒的倒数，表示 `1/60` 秒；不是输入 `0.0167` |
| Lens > Camera > Aperture (F-stop) | `4` | 光圈数值；与本地 `FPostProcessSettings` 初始值相同，但仍明确启用覆盖以固定实验条件 |
| Lens > Exposure > Exposure Compensation | `0` | 不额外加减曝光档数 |
| Lens > Exposure > Exposure Compensation Curve | None | 排除自定义曲线 |

首次实验使用普通 Camera，避免 Cine Camera 依据自身 Current Aperture 重新覆盖光圈。如果复用的项目通过蓝图、相机系统或其他后处理更改了 `DepthOfFieldFstop`，则必须把最终光圈恢复到 `4` 后再比较；不能单凭 PPV 显示值断言最终视图值相同。

**源码已确认：** 实际曝光计算读取 `View.FinalPostProcessSettings`，采用以下相机曝光值：

```text
EV100 = log2(Fstop² × ShutterSpeed × 100 / ISO)
      = log2(4² × 60 × 100 / 100)
      = log2(960)
      ≈ 9.907
```

这里 `EV100` 是用 ISO 100 作为基准的曝光值；`Fstop` 为光圈数值，`ShutterSpeed` 为快门时间的倒数。上式核对的是相机参数如何进入曝光计算，不是“屏幕像素必然等于某个 RGB 值”的完整公式；还存在曝光标定、预曝光、色调映射与输出转换。即使物理光圈参与曝光计算，仍可通过下面的质量开关关闭景深模糊。

光强按第一章给出的方向光与点光源数值开始，然后保持不变。那些数值是搭建起点，尚未在编辑器实测，不保证所有复用场景都会有相同观感。需要调整时一次只调整一个光源或曝光补偿，并记录修改后的值；A、B 对照时保持一致。

来源：[Scene.h：Metering Mode](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Engine/Classes/Engine/Scene.h:1428)，[快门、ISO 与光圈字段](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Engine/Classes/Engine/Scene.h:1835)，[物理曝光开关](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Engine/Classes/Engine/Scene.h:1876)，[Scene.cpp：相机默认参数](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Engine/Private/Scene.cpp:487)，[默认光圈](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Engine/Private/Scene.cpp:592)，[PostProcessEyeAdaptation.cpp：实际相机曝光计算](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessEyeAdaptation.cpp:393)，[CineCameraComponent.cpp：电影相机的光圈覆盖](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/CinematicCamera/Private/CineCameraComponent.cpp:699)。

### A.4.2 把分辨率、后处理和雾固定下来

打开 **Standalone Game 的控制台**后，以下命令每行单独执行。它们写入当前运行会话，不是系统命令，也不会自动成为项目永久配置：

```text
r.SetRes 1280x720w
r.DynamicRes.OperationMode 0
r.ScreenPercentage 100
r.SecondaryScreenPercentage.GameViewport 100
r.BloomQuality 0
r.MotionBlurQuality 0
r.DepthOfFieldQuality 0
r.Fog 0
r.VolumetricFog 0
r.MegaLights.Allowed 0
```

`w` 选择窗口模式；1280 × 720 指目标渲染窗口的客户区域，不把标题栏计入图像。`r.SetRes` 的本地注册说明明确它针对游戏视图、对编辑器视口无效，见 [UnrealEngine.cpp](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Engine/Private/UnrealEngine.cpp:457)。`r.ScreenPercentage` 控制主要场景渲染比例，`r.SecondaryScreenPercentage.GameViewport` 固定后续游戏视口缩放比例，防止 DPI 自动比例参与首次比较；`r.DynamicRes.OperationMode=0` 关闭随负载改变比例的动态分辨率。输出分辨率与内部场景渲染分辨率是两个概念，即使最终窗口相同，内部处理的像素数也可能不同。

全局 PPV 同时将 Bloom Intensity 和 Motion Blur Amount 覆盖为 `0`，Lens Flare Intensity、Vignette Intensity 与 Film Grain Intensity 设为 `0`，不使用颜色分级 LUT 或自定义后处理材质。雾的初次关闭以场景中不放雾和大气对象为前提，`r.Fog=0` 不能泛指关闭一切天空、大气、云与用户编写的雾效果。

上述质量变量可运行时调整，重新启动独立游戏后应再次应用。此处保留普通色调映射与输出颜色转换，**“关闭 Bloom”不等于“关闭全部后处理”**；第一章算出的线性混合值不能直接与显示器上取到的最终 RGB 数值比较。

来源：[UnrealEngine.cpp：动态分辨率模式](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Engine/Private/UnrealEngine.cpp:475)，[LegacyScreenPercentageDriver.cpp：场景比例](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Engine/Private/LegacyScreenPercentageDriver.cpp:36)，[GameViewportClient.cpp：二级比例及 DPI](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Engine/Private/GameViewportClient.cpp:115)，[ConsoleManager.cpp：Bloom 质量](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Core/Private/HAL/ConsoleManager.cpp:3923)，[运动模糊质量](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Core/Private/HAL/ConsoleManager.cpp:3967)，[景深质量](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Core/Private/HAL/ConsoleManager.cpp:4014)，[FogRendering.cpp](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/FogRendering.cpp:19)，[VolumetricFog.cpp](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/VolumetricFog.cpp:40)。

### A.4.3 编辑器视口、Game View、PIE 与 Standalone

| 观察位置 | 可以用来做什么 | 不能直接推断什么 |
|---|---|---|
| 编辑器关卡视口 | 摆放对象、切换缓冲可视化、检查材质 | 其尺寸、曝光与屏幕比例不一定等于游戏窗口 |
| 编辑器 Game View | 隐藏编辑器辅助显示，接近游戏的显示标志组合 | 它仍是编辑器视口；不是启动游戏，也没有自动切换到关卡 Camera 的保证 |
| PIE，即 Play In Editor | 在编辑器中运行游戏逻辑并验证实际取景摄像机 | 当前视口 PIE 与新窗口 PIE 的分辨率、窗口设置不相同；编辑器与游戏共享进程时，控制台修改的影响范围也应记录 |
| Standalone Game | 本教材固定分辨率和配置对照的主要观察入口 | 不保证和 Shipping 打包程序的性能相同；编辑器、资产构建与后台进程仍会影响机器负载 |

**源码已确认：** 编辑器视口的一条屏幕比例设置路径明确注释“按设计忽略 `r.ScreenPercentage` 与后处理 ScreenPercentage”，并使用自己的比例驱动。因此在 Output Log 输入 `r.ScreenPercentage 100` 后，仅看编辑器视口不能证明渲染比例已经变成 100%。编辑器观察时使用其屏幕比例与曝光显示设置；需要固定游戏输出时在实际 Standalone 会话核验。

来源：[EditorViewportClient.cpp：编辑器视口的独立比例驱动](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Editor/UnrealEd/Private/EditorViewportClient.cpp:4738)。

## A.5 反射与金属球的预期现象

A 选择 SSR，即 Screen Space Reflections，屏幕空间反射。它用当前视图可用的信息近似寻找反射内容，无法可靠重建屏幕之外、被其他物体挡住、或没有保存在相关屏幕缓冲中的表面。

为了不因质量设置误判 SSR 已被关闭，在 A 的观察会话输入 `r.SSR.Quality 3`，PPV 的 Reflections > Screen Space Reflections 覆盖为 Intensity `100`、Quality `100`、Max Roughness `0.8`。`r.SSR.Quality` 的 `0～4` 与 PPV Quality 的 `0～100` 是不同刻度；前者会限制后者能够采用的质量等级，并不是“两个地方都设为 3”。

金属球按第一章使用较低粗糙度。粗糙度超过可处理范围时反射会衰减或不参与；即使粗糙度合适，也不能要求球体每一处都显示完整场景。移动摄像机后，反射在画面边缘消失或变化，是识别屏幕空间限制的观察入口，不足以单独证明网格材质出错。

由于 A 没有 Sky Light、反射捕获或烘焙间接光照，金属球除直接光照高光和可获得的 SSR 外，大片区域可能较暗。金属没有像普通非金属那样的漫反射项，**将 Base Color 改亮并不保证整颗球体呈现同样明亮的实体颜色**。这正好展示“材质参数”与“最终像素颜色”的区别。不要为了让球体看起来漂亮而偷偷添加天空光，然后继续声称它仍是 A 的原始实验。

B 使用 Lumen 反射后能使用更多场景表示，但仍有表面表示、软件追踪、材质支持与时间累积方面的限制。B 不是离线真值，Lumen 的屏幕追踪部分也不等同于 A 所选的独立 SSR 反射方法。

**源码已确认：** `ShouldRenderScreenSpaceReflections` 检查显示标志、最终反射方法、视图状态、质量、强度等条件；项目方法选择并不是唯一门槛。PPV 的 GI 与反射方法本身也有对应的覆盖字段。

来源：[ScreenSpaceRayTracing.cpp：SSR 质量注册](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/ScreenSpaceRayTracing.cpp:14)，[ShouldRenderScreenSpaceReflections](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/ScreenSpaceRayTracing.cpp:130)，[Scene.h：SSR 参数](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Engine/Classes/Engine/Scene.h:1799)，[GI 覆盖](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Engine/Classes/Engine/Scene.h:1700)，[反射方法覆盖](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Engine/Classes/Engine/Scene.h:1768)。

## A.6 执行一次配置核验

1. 在专用练习项目中选择 A，记录引擎版本、显卡、驱动和设置值。从空关卡建立第一章场景；关闭项目级功能后重启，等待 Shader 和网格派生数据准备完成。
2. 设置唯一 PPV 与实际游戏摄像机。GI、反射方法在 PPV 中保持未覆盖，使其继承项目矩阵；曝光和本附录明确要求的参数开启覆盖。
3. 选择固定画质档，进入 Standalone，应用 A.4.2 的观察命令；确认游戏画面来自案例 Camera。只切换编辑器 Game View 不能完成这一步。
4. 在同一个运行会话逐行输入下面不带数值的变量名，查看控制台返回的当前值。带数值是写入，不带数值是查询；只读项目变量也用查询来核验，不强行热修改。
5. 检查对应缓冲或功能可视化，并记录预期现象。变量正确但功能可视化不正确时，继续检查视图覆盖、资产支持和平台能力。等待历史稳定后观察，但不要声称一个固定等待帧数适用于所有功能和场景。

```text
r.ForwardShading
r.Substrate
r.Substrate.ProjectGBufferFormat
r.Nanite.ProjectEnabled
r.Nanite
r.AllowStaticLighting
r.DynamicGlobalIlluminationMethod
r.ReflectionMethod
r.GenerateMeshDistanceFields
r.Lumen.TraceMeshSDFs
r.Lumen.HardwareRayTracing
r.Shadow.Virtual.Enable
r.RayTracing
r.RayTracing.Shadows
r.MegaLights.EnableForProject
r.MegaLights.Allowed
r.AntiAliasingMethod
r.SSR.Quality
r.DynamicRes.OperationMode
r.ScreenPercentage
r.SecondaryScreenPercentage.GameViewport
r.BloomQuality
r.MotionBlurQuality
r.DepthOfFieldQuality
r.Fog
r.VolumetricFog
```

查询数值与 A.2、A.3 对照；在 A 中未被使用的子功能值可以保留注册值，例如距离场关闭且 GI 为 None 时，`r.Lumen.TraceMeshSDFs` 为 `1` 不意味着 Lumen 在运行。A 不要求 Substrate 格式变量等于某个数值，因为外层 `r.Substrate=0` 已经关闭该路径。

没有全局 CVar 能替代对所有最终视图参数的核验。例如曝光来自 `FinalPostProcessSettings`，Nanite 是否参与还取决于每个网格。运行日志应同时记录：设置项、当前变量值、观察位置、资产是否启用、所用画质档，以及是否见到预期资源或阶段。控制台返回的 `LastSetBy` 信息如可用也一并记录，用于追踪值由哪类来源设置。

### A.6.1 完整切换与单项对照

完整切换 A 到 B 时，先记录 A 的设置与场景状态，再修改 Substrate、Nanite 项目支持、距离场等重启型设置，重启并等候构建；为 B 的不透明网格生成 Nanite 数据，保持透明薄片为普通网格，然后设置 Lumen、VSM 和相同的 TAA／曝光／分辨率。不要在资产还未准备完成时截图并解释为 B 的稳定效果。

材质系统改变会影响材质资产的兼容表示。A 的传统材质案例和 B 的 Substrate 案例应分别保留可恢复的版本；不要把复杂 Substrate 材质改存后，再通过关闭 Substrate 假定能自动还原出完全等价的传统材质。

研究单个特性时，先在已经具备所需 Shader 和资源的 B 中做单项实验。例如保持 Substrate、Lumen、阴影、曝光和摄像机不变，只切换 TAA／TSR；关闭 Nanite 时记录正在观察普通网格还是回退网格，不能把几何精度差异全部归为光照算法。A、B 整套配置的对比展示的是组合效果，不是任何一个特性的独立性能收益。

### A.6.2 常见偏差与排查顺序

| 现象 | 先检查什么 | 正确的解释边界 |
|---|---|---|
| 画面仍在自动变亮／变暗 | 是否是编辑器曝光覆盖；PPV 是否 Unbound；Manual 覆盖是否勾选；相机与其他体积是否覆盖 | 控制变量默认值不等于最终相机参数 |
| A 仍然看到反弹光或环境亮度 | 烘焙数据、Sky Light、反射捕获、环境立方体、自发光与其他灯是否被带入 | GI None 不是“所有非直接光来源都清空” |
| SSR 似乎没生效 | 当前反射方法、`r.SSR.Quality`、PPV 强度、粗糙度、反射目标是否在屏幕信息内 | 屏幕之外的反射内容缺失可能是算法能力限制 |
| B 的 Lumen 没生效 | 距离场生成是否开启并完成、GI 画质、最终方法、平台与显示标志 | Lumen 软件追踪不要求开启硬件光追，但需要自己的场景表示 |
| 开 `r.Nanite 1` 后没有变化 | 项目支持、资产 Nanite 数据、材质及平台支持；观察是否只是画面相似 | 算法改变不保证肉眼出现显著差异，需要对应可视化验证 |
| VSM 切换卡顿 | 是否正发生渲染状态重建、Shader 或资源准备 | 一次切换停顿不是稳定每帧开销 |
| 同为 720p 却速度和清晰度不同 | 主／次屏幕比例、动态分辨率、窗口客户区域、DPI、AA 方法及画质 | 最终输出像素数不足以唯一确定内部计算量 |
| 所有变量看起来正确但仍走其他路径 | 进程是否相同、后处理覆盖、平台回退、插件、命令行与配置优先级 | 应以实际路径和资源为最后证据，不能只凭一张设置截图 |

## A.7 本附录的验证范围

本批已核对本地版本、设置声明、枚举数值、关键变量注册、部分启用条件以及曝光计算。尚未执行项目创建、UI 操作、Shader 编译、距离场／Nanite 构建、Standalone 运行、截图与 GPU 捕获；因此所有观察现象均为待实践验证的预期，没有“运行观察已验证”的项目。

后续实践章节应附带实际版本、配置记录与捕获证据，再将对应条目标记为运行验证完成。来源中的行号是此次本地 UE 5.7.4 安装源码的定位信息；升级后先搜索符号与变量名，再重新确认条件，不能把旧行号直接当成新版本的证据。

[返回总目录](../README.md) · [源码索引](source-index.md)
