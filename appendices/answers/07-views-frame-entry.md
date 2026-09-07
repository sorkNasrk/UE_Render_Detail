# 第 07 章理解检查参考答案

[返回第 07 章](../../chapters/07-views-frame-entry.md) · [返回目录](../../README.md)

> 数值题采用正文声明的简化条件。实现相关答案依据本地 UE 5.7.4 静态源码，不表示已观察到运行调用或具体时间。

## 题 1：相机对象数不是请求数

Camera Actor 提供可供使用的相机信息。主游戏视图需要由玩家、视口或其他系统实际请求。普通单人、单目且仅选择 A 的前提下，另外两台相机存在，不会自动各形成一个主 View。

双目条件下，同一个本地玩家可能需要多个眼睛 View，因此玩家数与 View 数不再一对一。独立 SceneCapture 有自己的观察与目标，可能创建额外 ViewFamily 和 Renderer；UE 5.7 又有将特定捕获并入主 Renderer 的条件分支，所以应先明确独立捕获前提，再统计 Renderer 数。

统计应该沿请求来源、族内 Views 和所选渲染分支进行，而不是只遍历 Camera Actor 数量。

## 题 2：矩形、有效尺寸和资源尺寸

`Origin` 与 `Size` 是无单位比例，视口分量单位为像素。逐分量计算：

```text
起点 = (0.5,0.5)×(1920,1080) = (960,540)
尺寸 = (0.5,0.5)×(1920,1080) = (960,540)

初始矩形 = Min(960,540), Max(1920,1080)
```

矩形 Max 不包含在区域中，宽高用 Max 减 Min 得到。再施加主比例 0.5，假定其他比例均为 1：

```text
内部有效尺寸 = (960,540)×0.5 = (480,270) 像素
```

这还没有确定整个承载纹理的尺寸。资源 Extent 可能受其他 View 布局、对齐、尺寸量化、分配策略与功能条件影响；内部矩形也可能向左上平移。即使知道有效宽高，也要继续查所属资源描述及实际 ViewRect，不能直接用旧窗口坐标访问它。

## 题 3：遗漏了 Renderer 的观察数据副本

`FSceneRenderer` 在游戏线程构造时复制 ViewFamily，再为每个原始 `FSceneView` 创建自己的 `FViewInfo`，并把 Renderer 内部 ViewFamily 的指针重新指向新对象。对应入口是 [SceneRendering.cpp：2645 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/SceneRendering.cpp:2645)，逐个 View 构造在 2679 行后。

所以局部 Context 删除原始 Views，并不意味着 Renderer 也失去了自己的本次观察描述。这个结论依赖真实交接步骤，不能用“异步系统会自动保活所有裸指针”解释。

复制 View 描述不等于深拷贝整个场景、全部纹理和 ViewState。持久状态由相应拥有者管理，资源引用还遵守独立生命周期与同步规则。Renderer 内部对象的存在也不代表 GPU 已完成使用所有资源。

## 题 4：登记、排队、组织图与执行图

| 调用 | 本章中的责任 |
|---|---|
| `AddRenderer` | 登记 Renderer 与渲染函数等信息 |
| `FSceneRenderBuilder::Execute` | 处理场景渲染操作，安排包括渲染命令在内的后续工作 |
| 回调中的 `Renderer->Render` | 使用收到的 RDG Builder 组织该 Renderer 的适用渲染工作 |
| `FRDGBuilder::Execute` | 执行 RDG 所负责的构图结果及相关工作组织 |

它们都不能仅凭名称证明显示器已经显示该帧。CPU 命令组织、底层提交、GPU 完成、窗口呈现请求以及实际扫描显示有不同边界。还要看调用上下文，才能判断是否处于某个额外捕获、选择标识或正常主视图分支。

## 题 5：相机位置一致不足以对齐两个请求

还应检查旋转、FOV、投影类型与宽高比、ViewRect 和资源尺寸、Capture Source、显示标记、隐藏对象集合、最终后处理与曝光、材质与输出格式解释、抗锯齿方法，以及历史是否存在和有效。普通捕获与并入主 Renderer 的捕获也可能采用不同工作组织。

位置相同并不保证构图相同，例如主窗口 16:9、捕获目标 1:1 时就需要继续检查投影。捕获的 HDR 场景颜色与主窗口最终 LDR 颜色也不能直接比较数值。

开启 Capture Every Frame 后，再在 Tick 中显式调用 CaptureScene，可能重复请求捕获。源码对这种用法给出低效警告，见 [SceneCaptureComponent.cpp：833 行](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Engine/Private/Components/SceneCaptureComponent.cpp:833)。

关闭持续捕获之后，`bAlwaysPersistRenderingState` 会影响是否继续保留捕获视图状态。没有状态和有状态但历史过期是两种问题；即使保留状态，长时间未更新后的历史仍需满足具体算法的有效性条件。保留状态不会自动产生缺失期间的中间帧。

[返回第 07 章](../../chapters/07-views-frame-entry.md)
