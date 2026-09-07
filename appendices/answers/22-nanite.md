# 第 22 章答案：Nanite 虚拟化几何与可见表面

[返回本章](../../chapters/22-nanite.md) · [返回目录](../../README.md)

## 题 1

Cluster 是几何处理单位，包含适用三角形、属性与元数据；页面是存储、传输和驻留单位。运行时常驻粗层级属于 Nanite 几何表示，帮助在细节页尚不可用时保留可绘制内容。Fallback Mesh 是 Builder 另外生成的回退网格表示，不能把它和常驻父层级混为一谈。

```text
3 * 32 KiB + 5 * 128 KiB = 96 KiB + 640 KiB = 736 KiB
```

这是按题设页面 GPU 容量计算的局部账目，没有计入层级、依赖、实例、队列、可见性目标、上传空间和其他资源。也不能据此推算磁盘压缩文件大小。Cluster 的 128 三角形与 256 顶点是当前容量上限，不是每个 Cluster 的固定实际数量。

## 题 2

```text
1200 cm：2 * 900 / 1200 = 1.5 像素
2400 cm：2 * 900 / 2400 = 0.75 像素
```

在题设 1 像素阈值下，近处需要更细表示，远处可能接受当前表示。这个判断还假设对应细节已可用，且没有其他条件改变选择。

实际代码额外考虑层级误差范围、投影包围范围、实例非均匀缩放、变形尺度、视图 LODScale 等。正交视图和跨近裁剪面情况不能直接套用该透视近似。屏幕几何误差也不等于颜色误差或帧时间。

## 题 3

本题共有 `4+2=6` 个不同 Cluster 进入两轮光栅，另一个延后者仍被遮挡。这里按题设把实际实例与节点队列折叠成 Cluster 集合。

历史遮挡可能因相机或遮挡物移动而过时，Post 使用当前可用 Scene Depth 与主阶段 Nanite 深度建立的新 HZB 复核延后候选，补画当前重新可见的部分。它既不能随意省略，也不意味着把全部场景无条件再画一次。进入光栅还要经历逐像素深度竞争，不保证每个 Cluster 都留下最终像素。

当渲染器没有输入 HZB，或 TwoPass 开关关闭，会取消这套两阶段遮挡，采用 NoOcclusion 路线；适用视锥与 LOD 筛选仍然存在。调用者在开启 PrimeHZB 且满足条件时可先构造 HZB，所以不能把所有首次视图一律描述为绝对无 HZB。本版默认 PrimeHZB 为 0。

## 题 4

```text
Payload = ((12 + 1) << 7) | 9 = 1664 + 9 = 1673
Cluster = (1673 >> 7) - 1 = 12
Triangle = 1673 & 127 = 9
```

在题设非负反向 Z 约定中，0.8 比 0.4 更近。适用原子 Max 将胜出的深度和几何载荷作为一条打包记录保留。

VisBuffer 没有保存完整 Base Color。它提供当前可见列表引用和三角形索引，后续还需找到页面、Cluster、实例、顶点与材质，重建 UV、法线、位置及导数，再求值材料。可见 Cluster 索引不是跨帧稳定的资产 ID，内部 Triangle 索引也不是材质槽编号。

## 题 5

当前主线先完成适用 Main/Post 几何可见性光栅，建立 VisBuffer 与可见 Cluster 列表，再通过 `EmitDepthTargets` 合并 Scene Depth 并生成适用 Shading Mask、速度与模板信息。`DispatchBasePass` 调用 `ShadeBinning` 整理材质工作，由 `ShadeGBufferCS` 调度计算着色。Nanite Vertex Factory 按可见记录重建材质输入，`BasePassPixelShader.usf` 的 `MainCS` 包装调用共用材质主函数，再通过 UAV 导出。

B 是 Substrate 格式 0 的 Blendable GBuffer，应使用这一表示的绑定与消费者；不能无条件插入 Adaptive 的主材质数组输出。

深度的计算导出需要 DepthUAV、ExplicitHTile 和开关等条件，否则使用对应像素路径；这与后续材质计算着色是两次不同的工作。材质处理还存在 helper lane、Quad 组织和可编程光栅所需的求值，执行 lane 数不保证等于最终像素数。

VSM 从灯光的适用视图处理几何，以 DepthOnly 输出阴影页池。主相机不可见表面也可能投影阴影，不能直接把主相机 VisBuffer 当成所有灯光的答案。Nanite 几何页与 VSM 阴影页也是不同资源。

Q 的蓝片仍是 Translucent，不满足这条 Nanite 材质混合模式资格。它在适用普通透明阶段继续深度测试和合成，既不自动写入不透明 Nanite VisBuffer，也不因 Nanite 开启而绕过曝光和后处理。
