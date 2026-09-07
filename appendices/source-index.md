# 源码索引：从场景入口读到窗口呈现

本索引汇总已完成章节的源码阅读入口，并保留最初一帧主线的详细定位。核对日期：2026-09-07。核对方式为**静态阅读本地源码**；尚未启动本教材场景、设置断点或采集 GPU 帧，因而没有任何“运行观察已验证”的结论。

引擎根目录为 `G:\UnrealEngineInstalled\UE_5.7\Engine`。下文每个文件的路径都相对这一根目录；文件名链接指向本机绝对路径，冒号后的数字为核对时的行号。版本更新后行号可能变化，应优先搜索符号名。链接形式 `G:/.../file.cpp:123` 由支持本地源码定位的阅读器打开；不能定位行号时，在编辑器中打开该绝对路径并跳转到相应行。

## 各章源码导航

各章正文将源码定位、公式和输入输出放在一起解释。下表指向正文中的阅读路线；继续向下可以使用稳定的 `SRC-*` 主线入口。

| 章节 | 代码阅读主线 | 正文位置 |
|---|---|---|
| 01 | 游戏视图、渲染器调度、窗口呈现 | [从场景到像素](../chapters/01-from-scene-to-pixel.md) |
| 02 | LocalPlayer 视图 → FViewMatrices → LocalVertexFactory／DoubleFloat → Base Pass 顶点输出 | [坐标与投影](../chapters/02-coordinate-spaces.md) |
| 03 | Renderer 深度／混合选择 → RHI 状态描述 → D3D12 状态映射 | [覆盖、深度与混合](../chapters/03-raster-depth-blending.md) |
| 04 | 材质表达与编译 → Shader 绑定 → 传统材质数据与 BRDF 求值 | [Shader 与光照](../chapters/04-shaders-materials-lighting.md) |
| 05 | 场景纹理描述 → 颜色表示 → TAA 历史参数、读取与提取 | [资源、颜色与历史](../chapters/05-resources-color-history.md) |
| 06 | 组件注册／更新／注销 → Proxy／SceneInfo → 场景统一更新 | [游戏侧与渲染表示](../chapters/06-scene-representation.md) |
| 07 | GameViewport／LocalPlayer → ViewFamily → FSceneRenderBuilder | [一帧请求的发起](../chapters/07-views-frame-entry.md) |
| 08 | 渲染命令管道 → RHI 记录、翻译、提交 → 分层同步 | [线程与 GPU 协作](../chapters/08-threads-and-gpu.md) |
| 09 | Pass 参数 → 依赖／寿命分析 → 回调命令与历史提取 | [RDG 与真实 TAA 节点](../chapters/09-rdg.md) |
| 10 | RHI 命令 → D3D12 Context／状态 → Payload／Queue → 同步与呈现 | [后端命令与设备执行](../chapters/10-rhi-d3d12.md) |
| 11 | FScene.Update → GPU Scene 脏记录／上传 → CPU 可见性／GPU 实例筛选 | [数据更新与候选集合](../chapters/11-scene-visibility.md) |
| 12 | MeshBatch → Pass Processor → 缓存／可见 MDC → 实例参数／RHI 提交 | [绘制命令的组织](../chapters/12-mesh-draw-commands.md) |
| 13 | 深度政策 → Depth Pass Shader → HZB 归约 → 查询与历史消费 | [预通道、深度层级与遮挡](../chapters/13-depth-prepass-hzb.md) |
| 14 | Base Pass 附件／Uniform → 生成 GBuffer 编码 → DBuffer 阶段与接收 | [表面属性与贴花](../chapters/14-base-pass-gbuffer-decals.md) |
| 15 | 光源投影 → 投射者 → 阴影深度 → 屏幕比较／过滤 → 逐灯消费 | [常规阴影与 CSM](../chapters/15-shadows.md) |
| 16 | 灯光排序 → 每灯 RDG／Uniform → 屏幕位置恢复 → BRDF／衰减／阴影 → 加法写入 | [延迟直接光照](../chapters/16-direct-lighting.md) |
| 17 | SSAO 生成／合成 → SSR 输入历史与屏幕追踪 → 反射来源／材质响应 | [AO、间接光与反射](../chapters/17-indirect-ao-reflections.md) |
| 18 | 透明 MDC／混合 → Separate 合成；天空 LUT；高度雾／体积注入、历史与积分 | [透明与环境介质](../chapters/18-translucency-sky-fog-volume.md) |
| 19 | 速度写入与相机恢复 → Gen4 TAA；TSR 速度膨胀、拒绝、重建与历史提取 | [时间抗锯与重建](../chapters/19-velocity-taa-tsr.md) |
| 20 | 曝光／PreExposure → 后处理与 LUT／Tonemap → Slate → D3D12／DXGI Present | [从场景颜色到屏幕请求](../chapters/20-postprocess-present.md) |
| 21 | 材质拓扑编译与简化 → Blendable 导出／重建 → Tile 分类；Adaptive 容器对照 | [Substrate 的表达与运行表示](../chapters/21-substrate.md) |

## 先学会怎样使用索引

第一次阅读不需要理解每个模板、宏或分支。对一个入口先完成三件事：找到传入的关键数据；找到它构造、修改或传出的对象；找到下一层负责处理这些数据的函数。遇到条件语句，要记录进入该分支的条件，而不是只抄下函数名。

以下两条路线刻意分开，因为“物体加入渲染场景”通常不是每帧重新执行一次。

```text
路线一：组件首次注册或重建渲染状态
UPrimitiveComponent::CreateRenderState_Concurrent
  -> FScene::AddPrimitive / 批量注册路径
  -> FScene::BatchAddPrimitivesInternal
  -> IPrimitiveComponentInterface::CreateSceneProxy
  -> 组件自己的 CreateSceneProxy
  -> 排入渲染命令，创建渲染线程资源并加入场景

路线二：为一个游戏视口发起渲染
UGameViewportClient::Draw
  -> FRendererModule::BeginRenderingViewFamily / BeginRenderingViewFamilies
  -> FSceneRenderBuilder 创建并登记 SceneRenderer
  -> FSceneRenderProcessor 排入渲染命令
  -> 在命令中创建 FRDGBuilder，调用登记的渲染函数
  -> RenderViewFamily_RenderThread -> Renderer->Render
  -> 本教材配置进入 FDeferredShadingSceneRenderer::Render
  -> FRDGBuilder::Execute -> RHI 命令记录、调度与提交

窗口呈现的后续入口：
Slate 绘制窗口 -> Slate 的 RDG Execute -> PresentWindow_RenderThread
  -> FRHICommandListImmediate::EndDrawingViewport
  -> FD3D12CommandContextBase::RHIEndDrawingViewport
  -> FD3D12Viewport::Present / PresentChecked / PresentInternal
  -> IDXGISwapChain::Present
```

这是**源码导航路线**，并非一条覆盖所有线程的同步调用栈。排入命令的地方是跨线程边界；场景渲染与 Slate 使用的 RDG 也不能合并为一个没有边界的“全帧图”。

## 版本与场景入口

### SRC-VERSION

**源码已确认：本机引擎版本标识。**

- 相对路径：`Build/Build.version`。
- 入口：[Build.version，第 2 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Build/Build.version:2)。第 2、3、4 行分别为主、次、补丁版本，第 5 行为 Changelist，第 9 行为分支名。
- 核实值：`5.7.4`；`Changelist = 51494982`；`CompatibleChangelist = 47537391`；分支 `++UE5+Release-5.7`。

这一文件确认安装目录自述的版本。它不能独立证明每个源文件都未被修改，也不能证明当前启动的编辑器就是这个目录的可执行文件。本教材引用的是当前可读到的本地文件；没有把“已成功运行教学配置”作为版本核验结果。

### SRC-PRIMITIVE

**源码已确认：可渲染组件如何建立渲染侧表示。**

先读 `Source/Runtime/Engine/Private/Components/PrimitiveComponent.cpp` 的 [UPrimitiveComponent::CreateRenderState_Concurrent，第 643 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Engine/Private/Components/PrimitiveComponent.cpp:643)。它先调用父类并更新包围体，再检查是否应当加入场景；有注册上下文时，第 667 行交给 `Context->AddPrimitive`，否则第 671 行调用 `GetWorld()->Scene->AddPrimitive(this)`。因此，“一定立即直接 AddPrimitive”是过度简化。

然后读 `Source/Runtime/Renderer/Private/RendererScene.cpp` 的 [FScene::AddPrimitive，第 1302 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/RendererScene.cpp:1302)。普通路径在第 1309 行进入 `BatchAddPrimitivesInternal`，该模板函数从第 1343 行开始。第 1407 行经组件接口创建代理，第 1423 行创建 `FPrimitiveSceneInfo`，第 1433 行起整理变换和包围体等创建命令数据。

代理接口如何回到具体组件，可在同一个 `PrimitiveComponent.cpp` 文件的 [FActorPrimitiveComponentInterface::CreateSceneProxy，第 5512 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Engine/Private/Components/PrimitiveComponent.cpp:5512) 看见：第 5515 行调用 `Component->CreateSceneProxy()`。具体网格组件的重载决定生成哪种代理。

最后读 `RendererScene.cpp` 的 [AddPrimitiveCommand，第 1462 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/RendererScene.cpp:1462)：这是排入渲染命令的位置。命令内部第 1468 行设置代理变换，第 1469 行创建渲染线程资源，第 1471 行调用 `AddPrimitiveSceneInfo_RenderThread`。

由此能确认：游戏侧组件与渲染侧代理不是同一个对象；代理创建、命令入队、渲染线程处理是可区分的步骤。不能由此推断代理已被 GPU 看见、物体必然通过可见性判断，或所有组件每帧都会重新创建代理。`Actor` 也不是 GPU 绘制命令本身。

### SRC-VIEWPORT

**源码已确认：游戏视口如何准备视图并发起渲染。**

相对路径为 `Source/Runtime/Engine/Private/GameViewportClient.cpp`。从 [UGameViewportClient::Draw，第 1411 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Engine/Private/GameViewportClient.cpp:1411) 开始，关注第 1463 行的 `FSceneViewFamilyContext` 和第 1687 行的 `LocalPlayer->CalcSceneView`：前者组织同一次渲染的一组视图及共享设置，后者为玩家计算具体视图。

在 [BeginRenderingViewFamily 调用，第 1971 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Engine/Private/GameViewportClient.cpp:1971) 处跨入 Renderer 模块。外层第 1967 行要求没有禁用世界渲染、存在玩家视图且平台允许渲染；不满足时进入清理路径。

这个入口适合教材中的游戏视口。它不是“所有编辑器视口、Scene Capture、反射捕捉都必须通过的唯一入口”，也不能推断一帧只有一个 View 或 ViewFamily。

### SRC-RENDERER-ENTRY

**源码已确认：UE 5.7 的 Renderer 模块入口包含场景构建器。**

相对路径为 `Source/Runtime/Renderer/Private/SceneRendering.cpp`。从 [FRendererModule::BeginRenderingViewFamily，第 5034 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/SceneRendering.cpp:5034) 读起，第 5036 行包装为 `BeginRenderingViewFamilies`，后者定义在第 5039 行。

重点读 [FSceneRenderBuilder 创建位置，第 5167 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/SceneRendering.cpp:5167)：在存在场景的分支内，代码先处理可能存在的延迟更新捕捉，再于第 5176 行创建关联的 SceneRenderer。第 5204 行 `AddRenderer` 登记渲染函数，第 5207 行的函数体调用 `RenderViewFamily_RenderThread`，第 5217 行执行构建器。

这里的登记函数并不在 `AddRenderer` 那一行立即把整个画面算出来。应继续到下一条索引，找出回调何时被调用。另一方面，捕捉、平面反射和主视图会影响本次安排的渲染节点，不能把这里只有主视图的情况推广到所有场景。

### SRC-SCENE-BUILDER

**源码已确认：场景构建器如何选择渲染器、安排渲染线程回调与 RDG。**

相对路径为 `Source/Runtime/Renderer/Private/SceneRenderBuilder.cpp`。内部 `FSceneRenderProcessor` 从第 326 行开始。其 [CreateSceneRenderers 的选择分支，第 511 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/SceneRenderBuilder.cpp:511) 在 `EShadingPath::Deferred` 时，第 513 行构造 `FDeferredShadingSceneRenderer`；另一个分支构造移动渲染器。

接着读 [FSceneRenderBuilder::Execute，第 1079 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/SceneRenderBuilder.cpp:1079)，它在第 1084 行调用 `Processor->Execute()`。内部处理器的 `Execute` 定义在第 759 行，其中 [SceneRenderBuilder_Render 入队位置，第 829 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/SceneRenderBuilder.cpp:829) 是重要的线程边界。

在这个渲染命令的函数体中，第 872 行为当前渲染节点创建 `FRDGBuilder`；第 891 行在开启渲染的条件下调用先前登记的 `RenderNode.Function`；第 915 行执行该 RDG。这里的 `ERDGBuilderFlags::Parallel` 表示允许相关并行处理，不意味着所有 Pass 能同时执行。

一个节点一个局部 GraphBuilder 的这段实现，以及后面的 Slate 图，足以说明“一帧可能涉及多个 RDG”。它不能证明每次运行到底产生几个图。也不要仅凭 `FDeferredShadingSceneRenderer` 的类名就判定项目已采用传统 GBuffer 延迟光照：桌面前向配置在这一架构内仍有其他分支，必须结合配置和 Pass 条件判断。

## 从 Render 到资源和 Shader

### SRC-DEFERRED

**源码已确认：普通场景渲染分派及主要阶段的构图入口。**

先看 `Source/Runtime/Renderer/Private/SceneRendering.cpp` 的 [RenderViewFamily_RenderThread，第 4895 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/SceneRendering.cpp:4895)。它在 Hit Proxy 模式下调用专门绘制函数，普通分支第 4909 行才是 `Renderer->Render(GraphBuilder, SceneUpdateInputs)`。

桌面教材配置继续到 `Source/Runtime/Renderer/Private/DeferredShadingRenderer.cpp` 的 [FDeferredShadingSceneRenderer::Render，第 1736 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/DeferredShadingRenderer.cpp:1736)。阅读下面这些导航位置时，务必同时查看其外层条件和传入资源。

| 核对位置 | 代码中的工作 | 当前能够确认的内容 |
|---|---|---|
| 第 2208 行 `bRenderDeferredLighting` | 决定是否做延迟光照 | 光照开关和渲染设置参与分支判断 |
| 第 2284 行 `ShouldRenderPrePass`；第 2384 行 `RenderPrePass` | 深度预通道相关工作 | 存在预通道条件及调用；第 2384 行在局部函数体中，不能单凭其文本位置判断实际调用时刻 |
| 第 2823、2849、3130 行 `RenderShadowDepthMaps` | 阴影深度工作 | 同名调用有多个条件位置，不存在仅靠搜索第一处就能确定的统一阴影时序 |
| 第 2905 行 `RenderBasePass` | 主视图 Base Pass | 调用接收 SceneTextures、DBuffer、深度访问和 Nanite 等相关参数 |
| 第 3314 行 `RenderLights` | 灯光相关 Pass | 构图使用场景纹理、光照通道纹理和已整理的灯光集合 |
| 第 3520、3654 行 `RenderTranslucency` | 透明物体相关工作 | 水上、水下及其他渲染条件存在不同路径 |
| 第 3943 行 `AddPostProcessingPasses` | 按视图加入后处理 | 后处理使用本帧场景结果，后面仍有窗口呈现流程 |

这些是 C++ 组织工作的位置，并非已经执行完的 GPU 事件。相邻两行之间可能只建立依赖或安排任务。更不能把表格的行序当成适用于 Nanite、Lumen、前向渲染、Scene Capture 和编辑器模式的统一时间线。后续章节会沿教学配置分别展开。

### SRC-RDG

**源码已确认：RDG 的 Execute 仍是 CPU 侧图执行与命令安排过程。**

相对路径为 `Source/Runtime/RenderCore/Private/RenderGraphBuilder.cpp`。从 [FRDGBuilder::Execute，第 1755 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/RenderCore/Private/RenderGraphBuilder.cpp:1755) 读起：第 1782 行创建图尾 Pass；第 1797 行区分非 Immediate Mode 路径；第 1800 行等待并行 Setup 任务；第 1891 行调用 `Compile()`。`Compile` 自身从第 1316 行开始。

随后第 2036 行附近进入执行 Pass 的组织过程，第 2082 行可见 `QueueAsyncCommandListSubmit`。继续到 [FRDGBuilder::ExecutePass，第 3482 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/RenderCore/Private/RenderGraphBuilder.cpp:3482)，可以看到第 3490 行执行前置处理、第 3492 行调用 `Pass->Execute(RHICmdListPass)`、第 3494 行执行后置处理。回调拿到的是 RHI 命令列表，不是屏幕像素数组。

这组代码支持将“添加 Pass”“编译依赖并安排资源”“调用 Pass 回调记录命令”“底层提交”“GPU 执行”分开解释。`GraphBuilder.Execute()` 返回不能作为所有 GPU 工作完成或画面已显示的证据；调试 Immediate Mode、任务并行和实际同步路径也需要单独考虑。跨帧资源的提取和历史数据复用将在 RDG 正文章节展开。

### SRC-BASEPASS

**源码已确认：传统材质参数进入 GBuffer，光照再读取这些数据。**

1. CPU 到 Shader 的连接：`Source/Runtime/Renderer/Private/BasePassRendering.cpp` 的 [材质 Shader 注册宏，第 141 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/BasePassRendering.cpp:141) 把对应 Base Pass 像素 Shader 类型注册到虚拟路径 `/Engine/Private/BasePassPixelShader.usf` 的 `MainPS` 入口。虚拟 Shader 路径对应实际 `Shaders/Private` 文件夹，不是本机磁盘根目录。
2. 读取材质结果：`Shaders/Private/BasePassPixelShader.usf` 的 [传统参数读取分支，第 992 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Shaders/Private/BasePassPixelShader.usf:992) 由 `!SUBSTRATE_ENABLED` 保护，第 994、995、996、998 行分别读取 Base Color、Metallic、Specular、Roughness。第 1138 行调用 `SetGBufferForShadingModel`。这些值来自材质求值结果，不是显示器上的最终 RGB。
3. 输出多个目标：同文件 [GBuffer 输出分支，第 2299 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Shaders/Private/BasePassPixelShader.usf:2299) 由 `USES_GBUFFER` 保护；第 2319 行调用 `EncodeGBufferToMRT`。MRT 是 Multiple Render Targets，多渲染目标，意味着一次绘制可以向多个已绑定的纹理写入不同结果。Base Pass 还可能产生 Scene Color 的初始贡献，不能概括为“只写材质、不写颜色”。
4. 消费材质结果：`Shaders/Private/DeferredLightPixelShaders.usf` 的 [传统延迟光照分支，第 368 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Shaders/Private/DeferredLightPixelShaders.usf:368) 读取屏幕空间数据，第 376 行读取场景深度，第 393 行得到阴影衰减，第 394 行把 GBuffer、灯光、视线等交给 `GetDynamicLighting`，第 396 行将计算的 Radiance 累加到输出。Radiance 指辐亮度，在这里可先理解为沿观察方向得到的光照颜色贡献。

注意两种“延后”：传统延迟渲染把很多不透明表面的光照工作放在材质数据写出以后；这与“GPU 比 CPU 晚执行”是两个不同概念。上述 Shader 同时有 Substrate 和其他编译分支，不能截取几行后宣称所有材质都写相同布局。

**同一文件里的透明绘制状态，是第一章混色算例的实现对照。** `BasePassRendering.cpp` 的 `SetTranslucentRenderState` 从第 235 行开始，在 [传统 Translucent 分支，第 355 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/BasePassRendering.cpp:355) 中，非 Holdout 情况的第 361 行使用 RGB 的 `BO_Add`、`BF_SourceAlpha`、`BF_InverseSourceAlpha`。这对应简单的 `C_out = C_src * alpha + C_dst * (1 - alpha)`。

该式只解释这一混合状态的 RGB 部分。目标 Alpha 使用独立混合因子；Additive、AlphaComposite、Holdout、Substrate 的有色透射以及分离透明合成不能直接套用同一式子。算例应在线性颜色中进行，且其简单数值不是完整物理玻璃或 UE 截图的逐像素预测。

**不受灯光计算影响，不等于不受曝光影响。** 对第一章的传统 Surface／Translucent／Unlit 薄片，继续读 `Shaders/Private/BasePassPixelShader.usf` 的 [自发光求值，第 1569 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Shaders/Private/BasePassPixelShader.usf:1569)：`GetMaterialEmissive(PixelMaterialInputs)` 取得材质的自发光结果，第 1630 行在对应非 Thin Translucent 分支将其加到 `Color`。第 2258 行的普通透明分支输出颜色与 Opacity。

随后同文件的 [预曝光处理，第 2421 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Shaders/Private/BasePassPixelShader.usf:2421) 从 `View.PreExposure` 取得缩放，第 2427 行把普通 Translucent 纳入 RGB 缩放分支，第 2428 行执行 `Out.MRT[0].rgb *= ViewPreExposure`。这个分支不把透明混合用的 Alpha 同比例缩放。由此可以确认：Unlit 提供的 Emissive 仍进入场景颜色的内部数值处理，不能因为名字叫“无光照”就把它当成无需曝光或显示转换的屏幕颜色。

物理相机参数如何决定手动曝光的计算依据，见 `Source/Runtime/Renderer/Private/PostProcess/PostProcessEyeAdaptation.cpp` 的 [CalculateManualAutoExposure，第 389 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessEyeAdaptation.cpp:389)：第 395 行从最终视图设置的光圈、快门和 ISO 计算 EV100，第 396 行判断是否应用物理相机曝光，第 398 行转换为相应亮度依据。预曝光是内部存储与计算的缩放机制，不能把它和最终曝光、色调映射混为完全相同的一步；单看这几行也不能计算完整屏幕 RGB。

因此第一章实操将薄片颜色与自发光强度分开设置，使其量级适合固定曝光下的观察；这个强度是待实测的教学起点，不是从 Shader 推导出的唯一正确数值。手算混合公式则使用明确给出的同一线性表示输入，不能把实操材质的原始 Emissive 数值直接代入并要求与显示拾色一致。

### SRC-SCENE-TEXTURES

**源码已确认：场景颜色、深度、GBuffer 和速度是不同用途的数据。**

相对路径 `Source/Runtime/Renderer/Internal/SceneTextures.h`，注意 5.7 中该头文件位于 `Internal`。从 [FMinimalSceneTextures，第 51 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Internal/SceneTextures.h:51) 阅读：第 74 行 `Color`，第 77 行 `Depth`；派生的 `FSceneTextures` 从第 109 行开始，第 131 行起保存 `GBufferA` 等引用，第 143 行 `Velocity`，第 150 行 `ScreenSpaceAO`。字段存在不代表所有配置都分配并有效写入它。

再看 `Source/Runtime/Renderer/Private/SceneTextures.cpp` 的 [FSceneTextures::InitializeViewFamily，第 684 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/SceneTextures.cpp:684)。第 703 行创建速度纹理引用，第 710 行起按绑定信息有条件创建各 GBuffer 纹理。更早的第 485 行可见 `SceneDepthZ` 的创建，第 514 行可见 `SceneColorMS` / `SceneColor` 的创建。RDG 创建资源描述不等于 GPU 已经把场景渲染进该资源。

同文件 [FSceneTextures::GetGBufferRenderTargets，第 852 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/SceneTextures.cpp:852) 的第 859 行把 `Color.Target` 放入第一个目标槽；第 861 行判断是否使用 GBuffer；第 876 行读取所选布局，再按绑定索引填入目标。这是“Base Pass 同时涉及场景颜色和材质数据”的 CPU 侧连接。

Shader 中的数据视图可看 `Shaders/Private/DeferredShadingCommon.ush` 的 [FGBufferData，第 335 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Shaders/Private/DeferredShadingCommon.ush:335)。第 338 行法线、第 346 行基础颜色、第 348 行金属度、第 361 行粗糙度、第 380 行深度。深度旁的注释明确说明解码时由 Z Buffer 重建，不能把每个结构成员都理解为 GBuffer 某个独立通道的原样拷贝。

此结构描述 Shader 使用的语义数据；实际纹理通道打包、格式、分辨率、可用性由平台和配置决定。Scene Color 也是逐步写入的中间结果，它不会在分配时就“包含完整光照”，更不等同于窗口交换链的 Back Buffer。

## 从窗口绘制到 Present

### SRC-SLATE

**源码已确认：Slate 窗口渲染有自己的 RDG 和呈现入口。**

相对路径 `Source/Runtime/SlateRHIRenderer/Private/SlateRHIRenderer.cpp`。从 [FSlateRHIRenderer::DrawWindows_RenderThread，第 1066 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/SlateRHIRenderer/Private/SlateRHIRenderer.cpp:1066) 看窗口数组处理：第 1087 行创建名为 `Slate` 的 RDG；第 1098 行调用 `DrawWindow_RenderThread` 安排窗口绘制；第 1102 行执行 RDG；第 1112 行调用 `PresentWindow_RenderThread`。

继续读同文件 [FSlateRHIRenderer::PresentWindow_RenderThread，第 920 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/SlateRHIRenderer/Private/SlateRHIRenderer.cpp:920)，第 945 行调用 `RHICmdList.EndDrawingViewport`，传入窗口视口、需要 Present 的标志和垂直同步设置。

这样能把“场景后处理输出”与“应用窗口 UI 绘制和呈现请求”区分开。但不能据此宣称任何 UI 都在最后一个统一 Pass 中绘制，也不能证明所有离屏捕捉都会立即 Present。编辑器窗口、多窗口、HDR 合成、自定义 Present 和 XR 都需要额外路径分析。

### SRC-D3D12-PRESENT

**源码已确认：普通 Windows D3D12 窗口的呈现请求如何走到 DXGI。**

1. RHI 包装层：`Source/Runtime/RHI/Private/RHICommandList.cpp` 的 [FRHICommandListImmediate::EndDrawingViewport，第 1908 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/RHI/Private/RHICommandList.cpp:1908)，先安排先前工作的调度，然后依 Bypass 状态选择直接调用上下文（第 1918 行）或记录 `FRHICommandEndDrawingViewport`（第 1922 行）。这解释了为什么不能把“调用 RHI 函数”统一画成“已经进入驱动”。
2. 记录命令的执行桥梁：`Source/Runtime/RHI/Public/RHICommandListCommandExecutes.inl` 的 [FRHICommandEndDrawingViewport::Execute，第 562 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/RHI/Public/RHICommandListCommandExecutes.inl:562)，第 565 行经上下文调用 `RHIEndDrawingViewport`。
3. D3D12 上下文：`Source/Runtime/D3D12RHI/Private/D3D12Viewport.cpp` 的 [FD3D12CommandContextBase::RHIEndDrawingViewport，第 753 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/D3D12RHI/Private/D3D12Viewport.cpp:753)，在 `bPresent` 成立时，第 775 行调用 `Viewport->Present`。
4. 命令提交和呈现检查：同文件 [FD3D12Viewport::Present，第 598 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/D3D12RHI/Private/D3D12Viewport.cpp:598)，第 618 行 `FlushCommands(WaitForSubmission)`，第 622 行 `PresentChecked`。`WaitForSubmission` 的文字和用途是等待提交处理，不能等同于等待所有 GPU 工作完成。`PresentChecked` 从第 519 行开始，第 553 行允许自定义 Present 决定是否还需要原生呈现，第 562 行才进入 `PresentInternal`。
5. 平台交换链：`Source/Runtime/D3D12RHI/Private/Windows/WindowsD3D12Viewport.cpp` 的 [FD3D12Viewport::PresentInternal，第 352 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/D3D12RHI/Private/Windows/WindowsD3D12Viewport.cpp:352)，设置同步和撕裂标志后，第 388 行调用 `SwapChain1->Present(SyncInterval, Flags)`。

Present 把待显示缓冲交给呈现系统。源码调用本身不等于显示器已经完成扫描输出；操作系统合成、交换链排队、垂直同步和显示刷新仍会影响可见时刻。这里没有测量输入延迟、帧间差或实际屏幕显示时间，也没有声称 Present 在所有机器上都固定耗时或固定阻塞。

## 编辑器操作定位

### SRC-EDITOR-TRANSFORM

**源码已确认：Transform 旋转字段与 Roll、Pitch、Yaw 的绑定关系。**

相对路径为 `Source/Editor/DetailCustomizations/Private/ComponentTransformDetails.cpp`。在 [旋转输入框绑定，第 578 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Editor/DetailCustomizations/Private/ComponentTransformDetails.cpp:578)，第 578、579、580 行分别将 `Roll`、`Pitch`、`Yaw` 绑定到 `GetRotationX`、`GetRotationY`、`GetRotationZ`。第 593～598 行的值改变与提交回调继续将它们对应到 `EAxisList::X`、`Y`、`Z`。

因此，在按 X、Y、Z 标注的常用旋转字段中，映射是 **X = Roll，Y = Pitch，Z = Yaw**。第一章表格的旋转顺序采用 `(Pitch,Yaw,Roll)`，不能按这个顺序直接填到三个 X/Y/Z 输入框；例如薄片的 `(90,0,0)` 应填为旋转 X=`0`、Y=`90`、Z=`0`，相机的 `(-18,0,0)` 应填为 X=`0`、Y=`-18`、Z=`0`。

这一结论针对字段的语义绑定，不保证所有界面、用户轴显示设置或其他软件都用相同的排列。相邻第 585 行还对显示顺序使用轴显示设置；操作时应按实际字段标签和提示确认，不能只背“从左到右第几个”。本批未运行编辑器界面验证。

## 后续引用的维护规则

教材引用这些条目时使用稳定编号，例如 `SRC-BASEPASS`，而不是只写“见源码”。如果结论超出条目已核对范围，应先补读对应分支和调用关系，再扩充索引。新增运行实验时另记引擎版本、配置、场景、观察工具、具体结果，不能直接把本页的“静态已确认”改成“运行已确认”。
