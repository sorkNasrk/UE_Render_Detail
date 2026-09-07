# 参考资料与证据分级

核对日期：2026-09-07。本书的具体实现以本地 UE **5.7.4，CL 51494982** 源码为依据，入口见 [源码索引](source-index.md)。官方文档用于概念、设计意图、使用方法和功能边界的补充。

下面五篇 Epic 官方文档均在核对日请求了 `application_version=5.7`，网页标题均显示 **Unreal Engine 5.7 Documentation**，已读取下文引用的正文段落。网页的版本选择不保证所有历史示例都已经更新为 5.7；具体差异在各条目说明。没有把 HTTP 200 或页面能打开当作正文与实现已核验的证据。

## 怎样判断一个说明能信到什么程度

| 标签 | 在本教材中的含义 | 不代表什么 |
|---|---|---|
| 源码已确认 | 已阅读本地对应版本的有关实现、上下文或数据定义，给出文件和符号入口 | 不代表当前工程运行时一定进入该分支 |
| 教学简化 | 为分离概念而使用的模型、算例、图示或省略条件 | 不代表与 UE 的完整算法、精确数值或实际 GPU 时序相同 |
| 运行观察已验证 | 在记录的场景、设置、平台和工具下实际观察到的结果 | 不代表所有平台和配置都相同；当前教材尚无此类 UE 实验结果 |
| 尚未验证 | 尚未核对的推测、待做实验或无法访问的证据 | 不可写成已确认事实，也不可用预期截图代替实测 |

基础数学等一般原理不应伪装成 UE 源码结论。概念段落按以下来源说明依据；凡涉及 UE 具体函数、配置行为或版本差异，继续以对应源码或明确版本文档核对。

## 已读取的官方文档

### DOC-MESH

[Mesh Drawing Pipeline in Unreal Engine](https://dev.epicgames.com/documentation/en-us/unreal-engine/mesh-drawing-pipeline-in-unreal-engine?application_version=5.7)

- 发布者：Epic Games。适用页面版本：5.7；本教材使用范围：普通网格绘制组织的基本架构。
- 已读取部分：开头的绘制过程、Cached and Dynamic Mesh Batches、FMeshPassProcessor、Shader Bindings，以及绘制命令缓存和合并的说明。
- 可支持的解释：`FPrimitiveSceneProxy` 向渲染器提供 `FMeshBatch`；Mesh Pass Processor 将其整理为某个 Pass 的 `FMeshDrawCommand`；绘制描述再转换为 RHI 命令；静态缓存和动态提交具有不同生命周期。
- 阅读提示：把这篇当作“一个网格的绘制描述如何走到 RHI”的路线图。它不是 Nanite 软件光栅化的完整流程，也不能用旧示例中的函数或“一次缓存”的表述替代当前代理失效、重建和特殊 Pass 的实际条件。

对应源码入口：[SRC-PRIMITIVE](source-index.md#src-primitive)。已完成的[第 12 章](../chapters/12-mesh-draw-commands.md) 核对普通网格的绘制命令链，并区分缓存、实例参数生成与提交。

### DOC-THREADS

[Threaded Rendering in Unreal Engine](https://dev.epicgames.com/documentation/en-us/unreal-engine/threaded-rendering-in-unreal-engine?application_version=5.7)

- 发布者：Epic Games。适用页面版本：页面标题为 5.7，但正文保留旧版 API 示例。
- 已读取部分：Rendering Thread、Thread Specific Data Structures、Inter-thread Communication、Rendering Resources、UObjects and Garbage Collection 及动态资源更新示例。
- 可支持的解释：游戏侧与渲染侧数据应有明确所有权；跨线程传递稳定的数据副本或命令；资源生命周期必须延续到最后一个使用者不再访问；仅凭代码书写顺序无法保证另一线程已完成工作。
- 版本限制：正文仍出现 `DrawDynamicElements`、`ENQUEUE_UNIQUE_RENDER_COMMAND_XXXPARAMETER` 等历史写法，并使用“落后一两帧”的概括。本书不将这些作为 UE 5.7 当前 API 或所有配置固定帧差的依据。当前入口使用本地的 `CreateRenderState_Concurrent`、组件接口、`ENQUEUE_RENDER_COMMAND` 与 `FSceneRenderBuilder` 核对。

对应本批源码入口：[SRC-PRIMITIVE](source-index.md#src-primitive)、[SRC-SCENE-BUILDER](source-index.md#src-scene-builder)。

### DOC-RDG

[Render Dependency Graph in Unreal Engine](https://dev.epicgames.com/documentation/en-us/unreal-engine/render-dependency-graph-in-unreal-engine?application_version=5.7)

- 发布者：Epic Games。适用页面版本：5.7。
- 已读取部分：概述、特性列表、RDG Programming Guide 开头与 Shader Parameter Structs。
- 可支持的解释：RDG 用 Pass 和资源关系描述渲染工作；Shader 参数结构中的资源访问帮助形成依赖；图处理可以管理资源生命周期、资源状态转换、未使用 Pass 裁剪、CPU 并行命令记录与异步计算同步。
- 阅读提示：开头的“immediate-mode API”描述的是开发者逐项声明工作的 API 风格，不意味着 GPU 每见一次 `AddPass` 就立刻执行一次。本地实现另有调试用 Immediate Mode 分支；它与一般的构图、编译、执行流程必须区分。允许异步计算也不保证任意两项工作都能重叠。

对应本批源码入口：[SRC-RDG](source-index.md#src-rdg)、[SRC-SLATE](source-index.md#src-slate)。

### DOC-SUBSTRATE

[Overview of Substrate Materials in Unreal Engine](https://dev.epicgames.com/documentation/en-us/unreal-engine/overview-of-substrate-materials-in-unreal-engine?application_version=5.7)

- 发布者：Epic Games。适用页面版本：5.7。
- 已读取部分：概述、New Projects、Material Editor Conversion、Substrate Material Conversion、GBuffer Format for Visual Fidelity、Substrate Blend Modes 与透明光照模式。
- 可支持的解释：Substrate 以可组合的 BSDF 材质表示扩展旧材质系统；Blendable GBuffer 和 Adaptive GBuffer 有不同复杂度与性能取向；UE 5.7 新建项目默认启用 Substrate，一般新项目使用 Blendable GBuffer，汽车和建筑等模板可能默认 Adaptive；升级的已有项目通常保留非 Substrate 路径，除非显式启用。
- 配置限制：文档明确指出打开旧材质不再自动改变资产，旧输入可以在编译时适配 Substrate；手动转换为 Substrate 节点则会改变资产兼容性。因此教材先保持可对照的传统材质输入，不能把“开启 Substrate”和“把所有材质永久转换为节点图”当成同一步。
- 阅读提示：所谓“类似传统 GBuffer”是使用和性能目标的描述，不意味着二者的精确打包、Shader 分支和支持的材质行为完全一致。项目实际默认设置还应结合模板和工程配置确认。

对应本批源码入口：[SRC-BASEPASS](source-index.md#src-basepass)、[SRC-SCENE-TEXTURES](source-index.md#src-scene-textures)。

### DOC-PBR

[Physically Based Materials in Unreal Engine](https://dev.epicgames.com/documentation/en-us/unreal-engine/physically-based-materials-in-unreal-engine?application_version=5.7)

- 发布者：Epic Games。适用页面版本：5.7；讲解以传统 PBR 材质参数工作流为主。
- 已读取部分：What Does Physically Based Mean、PBR Material Attributes、Base Color、Roughness、Metallic 和 Specular。
- 可支持的解释：Base Color、Roughness、Metallic、Specular 是不同含义的材质输入；粗糙度影响镜面反射的分布；纯金属和纯非金属通常使用不同的 Metallic 端点；Base Color 不是照明、曝光和显示变换后的最终颜色。
- 阅读限制：本文是材质制作指南，不是完整着色方程规范。例如其中关于 Specular 的“fully reflective”概括不能用来推导“Specular=1 等于 100% 入射光都反射”；精确映射、菲涅耳和能量分配应继续核对 Shader。第一章的颜色乘法和 Alpha 混合只用于建立输入、处理、输出的认识，不宣称重现完整 PBR 光照。

对应本批源码入口：[SRC-BASEPASS](source-index.md#src-basepass)。

## 未采用为事实依据的入口

核对过程中尝试了 `rendering-overview-for-unreal-engine` 与 `unreal-engine-rendering-overview` 两个概览候选路径。它们虽然返回 HTTP 200，但当前响应只有导航壳，缺少可确认的标题和正文。因此不将这两个候选地址作为已核验参考资料，也不从空白响应推断“渲染概览支持某个说法”。本书的一帧总览依据上列具体文档和本地源码组合建立。

本书尚未执行编辑器观察练习，也未读取全部关联官方文档。第 18～25 章主要依据下面列出的本地实现完成，没有把未打开的网络页面列为已核实资料，也没有以静态代码阅读替代运行证据。

## 后续章节的源码依据

版本统一为本机 UE 5.7.4、CL 51494982，核对日期为 2026-09-07。以下目录相对 `Engine` 根目录，实际文件、符号与行号链接保留在对应章节中。表格说明已阅读的实现范围，并不声称审计过整份引擎。

| 章节 | 主要源码区域 | 核验重点 |
|---|---|---|
| [18 透明与环境](../chapters/18-translucency-sky-fog-volume.md) | `Renderer/Private/TranslucentRendering.cpp`、天空／雾相关 C++ 与 `Shaders/Private` 对应 Shader | 普通与 Separate 透明表示、天空条件、体积散射历史和最终积分 |
| [19 时间重建](../chapters/19-velocity-taa-tsr.md) | `Renderer/Private/PostProcess/TemporalAA.cpp`、`TemporalSuperResolution.cpp`、速度写入与对应 Shader | 相机速度恢复、TAA 限制历史、TSR 阶段／条件／提取 |
| [20 后处理与显示](../chapters/20-postprocess-present.md) | `Renderer/Private/PostProcess`、`SlateRHIRenderer`、`D3D12RHI/Private` | 手动／自动预曝光区别、内部 LUT、场景／窗口输出和 DXGI Present |
| [21 Substrate](../chapters/21-substrate.md) | 材质表达／HLSL 翻译器、`Renderer/Private/Substrate` 与对应 Shader | 表达编译、预算简化、Blendable 与 Adaptive、Tile 分类 |
| [22 Nanite](../chapters/22-nanite.md) | `Developer/NaniteBuilder`、Engine 流送、`Renderer/Private/Nanite` 与对应 Shader | 层级／页面、两阶段遮挡、VisBuffer、深度导出及本版材质 CS |
| [23 VSM](../chapters/23-virtual-shadow-maps.md) | `Renderer/Private/VirtualShadowMaps`、ShadowSceneRenderer 与对应 Shader | 页寻址／分配、静动态缓存、Receiver Mask、SMRT 与 One Pass 回退 |
| [24 Lumen 软件](../chapters/24-lumen-software-tracing.md) | `Renderer/Private/Lumen`、`Shaders/Private/Lumen` | 卡片、距离场、场景光照、Radiosity、探针／缓存、反射与合成 |
| [25 HWRT 与 MegaLights](../chapters/25-hardware-ray-tracing.md) | `Renderer/Private/RayTracing`、Lumen HWRT、MegaLights、D3D12RayTracing | 实例／加速结构、Inline／RayGen、命中光照模式与随机直接光照 |

上表的模块路径在 `Source/Runtime` 或 `Source/Developer` 下展开；Shader 路径单独位于 `Engine/Shaders`。全文真实源码只作定位和必要机制解释，不随仓库分发引擎源码。源码声明与帮助文字发生差异时，章节明确记录已看到的分支及未验证范围，例如本版 MegaLights 的可选方向光／软件路径。

## 建议的阅读顺序

先读教材第一章，再按 `DOC-PBR` 理解材质参数；进入 UE 架构篇后读 `DOC-MESH` 与 `DOC-THREADS`；理解“Pass 读什么、写什么”之后再读 `DOC-RDG`。`DOC-SUBSTRATE` 用于明确配置 A 与 B 的差别，可先读 New Projects 和 GBuffer Format，再在第 21 章学习 BSDF 与材质组合。

所有外链都可能随官网更新而改变。后续引用新增结论时，应重新检查相关正文、版本和本地实现，保留“阅读日期”与“实现版本”两个独立信息。
