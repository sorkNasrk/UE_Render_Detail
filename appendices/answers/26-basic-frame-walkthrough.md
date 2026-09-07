# 第 26 章答案：从 `UGameViewportClient::Draw` 追一帧基础画面

[返回本章](../../chapters/26-basic-frame-walkthrough.md) · [返回目录](../../README.md)

## 题 1

`UGameViewportClient::Draw` 收集窗口、玩家和视图族输入；`ULocalPlayer::CalcSceneView` 根据相机和后处理设置创建 `FSceneView`；`FSceneRenderBuilder::Execute` 处理已登记的 Renderer 与命令，并把工作安排到渲染线程；`FRDGBuilder::Execute` 编译、安排并执行当前 RDG 的资源和 Pass。前两个主要是请求与 View 构建，后两个仍是 CPU/RHI 组织边界。它们都不能单独证明 GPU 完成，更不能证明交换链和显示器已经更新。

## 题 2

一条合理链路是：Scene 更新把 P 的 Primitive、Bounds、实例和材质相关数据同步到渲染表示；Depth PrePass 为不透明 P 写 Scene Depth；DBuffer（若有有效贴花）在 Base Pass 前提供可应用属性；Base Pass 读取材质输入和深度，写 P 的 GBuffer、适用颜色和 Velocity；AO、间接光、阴影投影与 `RenderLights` 读取这些表面数据并累积 Scene Color；TAA 读取 Scene Color、Depth、Velocity、History 和 PreExposure，产生当前输出与下一帧历史。

Q 的蓝片在相关性和 Mesh Pass 资格选择时已经分流：它保持 Translucent，不进入单层不透明 GBuffer。本章明确选择 After DOF，之后由 `RenderTranslucency` 绑定适用深度并写 Separate 资源，再在后处理对应位置合成回 Scene Color。它的 `(30,180,300)` 与 `0.35` 是透明材质输入，不是 P 的 GBuffer 属性；本例 Cast Shadow 也已关闭。

## 题 3

一个合理的依赖顺序是：

```text
Scene.Update / View Relevance
 -> Depth PrePass 写 Scene Depth
 -> HZB / Occlusion 读取适用深度
 -> DBuffer（如有）
 -> Base Pass 消费深度并写表面资料
```

即使 C++ 代码按这个顺序调用，RDG 仍会根据资源声明、屏障、任务和队列能力组织实际命令；某些 CPU 准备任务可以并行，Async Compute 也可能与图形队列重叠。源码顺序表达调用关系和依赖意图，不是 GPU 时间戳。

若模式为 `DDM_AllOccluders`，即使 `bIsEarlyDepthComplete` 为 false，本处的 `bOcclusionBeforeBasePass` 仍为 true，因此普通最终颜色分支仍在 Base Pass 前调用遮挡。位置没有因此移到后面；实际深度包含哪些表面则由该模式的绘制集合决定。两项条件都不成立时才沿 Base Pass 后的调用位置继续读。不能从“遮挡提前”反推全部不透明深度已经完整。

## 题 4

主场景 RDG 由 `SceneRenderBuilder` 的 Render 操作在渲染线程创建，管理 SceneRenderer 的资源和 Pass；SceneCapture 或 Planar Reflection 可以拥有自己的观察与 RDG；Slate RDG 在窗口元素绘制时创建，消费场景输出并绘制 UI。它们的资源通过外部注册、提取、依赖和线程/提交顺序连接，不要求整帧只有一个 `FRDGBuilder`。

`GraphBuilder.Execute()` 覆盖当前图的编译、Pass 命令组织、资源屏障和提交安排；RHI Submission Event 表示适用命令已交给提交管道；GPU Fence 表示其协议覆盖的 GPU 进度到达；Present 请求交换链呈现并可能受提交、交换链空间、VSync 或系统合成影响。它们分别跨越不同边界，没有一个名称可以直接等同于“人眼已经看到”。

## 题 5

可以按以下路线阅读 Q：先在 `GameViewportClient::Draw` 和 `CalcSceneView` 确认窗口、ViewRect、相机和透明相关 View 设置；在 `SceneRenderBuilder` 与 `DeferredShadingSceneRenderer::Render` 中确认 `RenderBasePass`、`RenderTranslucency`、`AddPostProcessingPasses` 的调用条件；确认预通道建立方块深度、Base Pass 建立 GBuffer，再向下追到透明阶段；在 `TranslucentRendering.cpp` 确认 Q 所属 Pass、颜色/深度资源与 Separate Translucency 是否有效；在 `PostProcessing.cpp` 确认 PostDOF 资源何时合成；最后沿 SceneViewport、Slate 和 D3D12 Present 到交换链。

按本章另行给定的 `C_b=(96,24,12)`、`p=1/960` 算例，Separate 颜色为 `D=(0.0109375,0.065625,0.109375)`，剩余背景权重为 `T=0.65`。合成得到 `Q=(0.0759375,0.081875,0.1175)`；P 的背景存储值仍为 `(0.1,0.025,0.0125)`。它们都尚非最终显示 RGB。

至少有两处不能靠“看起来更蓝”证明：第一，不能由最终颜色证明 Q 写入了哪张透明中间纹理，必须查看 Pass 绑定和合成资源；第二，不能由窗口颜色证明 EV100、PreExposure 或 Tonemap 之前的线性 `Cs` 等于某个数值，需要检查曝光、色调映射和输出编码。即使 Q 的颜色改变，也不能单独证明它改变了 P 的 GBuffer 或主深度。
