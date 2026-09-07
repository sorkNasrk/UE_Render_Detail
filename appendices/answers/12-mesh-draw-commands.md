# 第 12 章理解检查参考答案

[返回第 12 章](../../chapters/12-mesh-draw-commands.md) · [返回目录](../../README.md)

> 答案依据 UE 5.7.4、CL 51494982 本地源码静态阅读。没有启动项目或测量 GPU 时间；涉及命令数量和性能的表述是机制解释，不是运行结果。

## 题 1：FMeshBatch 与 FMeshDrawCommand

`FMeshBatch` 是 Scene Proxy 提供给某个 Mesh Pass 的输入描述，包含 Vertex Factory、Material Render Proxy、一个或多个 Batch Element，以及索引范围、实例数、Primitive Uniform Buffer 和 Pass 资格标记。它还没有确定本 Pass 要使用的完整 Shader、Rasterizer、Depth-Stencil、Blend 和 PSO。

`FMeshDrawCommand` 则保存更接近一次绘制所需的绑定：Shader Bindings、Vertex Streams、Index Buffer、Cached Pipeline ID、首索引、图元数、实例数和 Primitive ID 流等。`FMeshPassProcessor` 根据材质 Blend Mode、Shading Model、Vertex Factory、Pass 类型和绘制状态筛选 Batch，并把共享状态与元素状态填进 Draw Command。如果跳过 Processor，不同 Pass 就会各自重复实现材质兼容性和 PSO 组合，容易产生错误，也无法统一缓存。

## 题 2：相机移动后的复用和重建

相机只移动而 P 所属方块的几何、材质、Vertex Factory、索引缓冲和 Pass 资格都没有改变时，适用的静态 `FMeshDrawCommand`、最小管线状态 ID 和共享 Shader 绑定可以复用。每个 View 仍要组织适用的视锥/遮挡判断、可见命令列表、View 常量与实例参数，透明排序键也要随观察条件更新。不是所有对象每次都执行完整 HZB 路径，具体取决于配置和历史有效性；反向剔除等 View 覆盖也只在相关条件下需要调整。若 Batch 标记为 `bViewDependentArguments`，动态构建阶段先让 Proxy 根据当前 View 修改副本后再调用 `AddMeshBatch`。

因此“命令缓存”不等于“整帧结果缓存”。屏幕位置、可见性、矩阵和部分排序是 View 相关的；材质和几何状态才是适合跨帧缓存的部分。

## 题 3：透明 Q 与不透明 P 的差异

至少有以下差异：

1. Blend Mode 不同。P 的方块使用不透明输出；Q 的薄片需要透明混合，可直接参与 Scene Color 或先写透明中间目标再合成。方块的 Base Pass 是否写深度还取决于预通道和当前深度状态，不能固定为每次都写。
2. Pass 不同。P 可进入 DepthPass 和 Deferred BasePass，写入深度与 GBuffer；Q 通常进入 `TranslucencyStandard` 或其他透明 Pass，不按普通不透明流程写 GBuffer。
3. 顺序约束不同。P 的遮挡主要由深度测试决定；Q 的结果依赖透明排序、Sort Priority、排序轴或 OIT 策略。
4. 深度写入和读取策略可能不同。Q 常读取 P 的深度以避免把透明片画到遮挡物后，但不一定把自己的深度像不透明物体一样写回。
5. Shader permutation、Render Target 和后处理位置可能不同，例如透明可能在景深前后不同阶段处理。

所以 Q 可以复用同一套 Scene Proxy、Batch 和 Processor 框架，但必须选择透明专用的 Pass 规则和 PSO。

本版本标准透明等注册项没有 `CachedMeshCommands`，而不透明 BasePass 有该标志。因此薄片来自静态 Batch，也不等于标准透明 MDC 会沿不透明路径跨帧缓存，见 [BasePassRendering.cpp：2718 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/BasePassRendering.cpp:2718)。

## 题 4：SubmitMeshDrawCommandsRange 之后的边界

并不必定经过这个函数。当前并行 BasePass 的适用路径是 `AddDispatchPass → FParallelMeshDrawCommandPass::Dispatch → FDrawVisibleMeshCommandsAnyThreadTask → InstanceCullingContext.SubmitDrawCommands`。实例上下文启用时，它按实例参数调用 `SubmitDrawBegin/End`；不启用时才转普通 Range 回退。该版本 Range 函数本身禁止该路径动态实例化，不能把它当作当前实例压紧的统一入口。

某次 Range 调用完成后，CPU 已遍历指定可见命令并处理适用的状态、顶点流、Shader 绑定和 RHI Draw。某些命令还可能因明确的 PSO 预缓存策略跳过；所以返回也不证明每个候选都发出了设备 Draw。

这不能证明：

- RHI 命令已被翻译成 D3D12 原生列表；
- D3D12 队列已经调用并完成 `ExecuteCommandLists`；
- GPU 已完成深度、GBuffer 或透明像素；
- 后处理已完成或交换链已经显示。

这些边界由后续 RHI、D3D12 提交、GPU Fence 和 Present 流程决定。即使 `DrawIndexedPrimitive` 在调用栈中返回，通常也只是 CPU 记录阶段结束。

另外，GPU 可以在后续计算阶段把实际可见实例数写入间接参数缓冲。CPU 记录引用这个缓冲的 Draw，无需先读回最终计数；前后 GPU 工作由资源依赖连接。

## 题 5：不同 Vertex Factory 不能随意实例化

动态实例化要求两个 `FMeshDrawCommand` 的 Cached Pipeline ID、Stencil Ref、Shader Bindings、Vertex Streams、Primitive ID Stream、Index Buffer、首索引、图元数、实例数以及顶点参数等满足匹配条件，源码的 `MatchesForDynamicInstancing` 对这些字段逐项比较。不同 Vertex Factory 通常意味着不同顶点声明、Vertex Streams、Primitive ID 处理方式或 Shader permutation，因此即便 Material 相同，也可能不满足匹配。

即使匹配成立，还要满足实际合并路径的相邻状态桶、裁剪标记、实例顺序和批次容量等条件。实例化主要减少重复命令和状态设置，不能把合并前后的 Draw 数比值当成全帧加速比；后续实例裁剪才可能进一步去掉不可见实例的部分 GPU 工作。

[返回第 12 章](../../chapters/12-mesh-draw-commands.md)
