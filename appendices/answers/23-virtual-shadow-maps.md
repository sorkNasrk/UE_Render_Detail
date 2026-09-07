# 第 23 章答案：Virtual Shadow Maps

[返回本章](../../chapters/23-virtual-shadow-maps.md) · [返回目录](../../README.md)

## 题 1

源码常量是 128×128 页、16384×16384 最大虚拟平面、完整局部 mip 8 级。以下是只计 R32 深度元素的教学估算：

```text
每页 = 128*128*4 = 65536 B = 64 KiB
2048 页单层 = 128 MiB
2048 页双层 = 256 MiB
虚拟页 = (4,2)，页内 = (5,3)
物理 texel = (10*128+5,7*128+3) = (1285,899)
```

实际池按行宽向上取整，缓存和可视化影响数组层数；还存在页表、元数据、HZB、列表、对齐和 RHI 分配成本。虚拟地址空间也不要求全部驻留，故不能用这些值冒充实测总显存。

## 题 2

方向光建立多个不同范围的正交 clipmap，每级普通路径一个 mip；完整局部阴影建立投影面及最多八级 mip，点光通常需要六面。两个系统都可以减少远处采样密度，但 clipmap 索引和局部 mip 索引不是同一量。

本版 `RawRadius(6)=2^(6+1)=128` 世界单位，`HalfLevelDim=256`，投影全宽为 512；采用厘米时全宽 5.12 m。它包含 snapping 的范围扩展，不是 128 的简单直径。

请求来自接收者可能查询的灯光投影区域；相机外的几何也能挡住射向接收者的光。粗级页表指针可供采样回退，但不设置本级可渲染位，细级坐标直接写入会混淆覆盖密度和地址。[页表编码](G:/UnrealEngineInstalled/UE_5.7/Engine/Shaders/Private/VirtualShadowMaps/VirtualShadowMapPageAccessCommon.ush:265)

## 题 3

固定深度附件没有承担此处的存储工作。普通 VSM Pixel Shader 解码物理页后，用 UAV `InterlockedMax(asuint(DeviceZ))` 保留 reverse-Z 最近深度；`asuint` 是浮点位解释，不是把小数转换成整数 0。

教学 texel 最终是 `max(0,0.25,0.70,0.40)=0.70`，忽略 bias 时 `0.70>0.40`，接收者被挡。SMRT 因子为 `3/7≈0.428571`，是可见比例近似，不是最终 RGB。RayCount=0 走单次 VSM 查询；关闭 VSM 则退出该阴影方法。

不能把 7×8=56 说成精确纹理读取数，模板有终端样本、命中早退、层级和有效性处理。

## 题 4

Receiver Mask 按当前接收区域限制动态内容，页可能不完整，所以该投影动态部分标 uncached；静态层仍可复用。方向 Receiver Mask 在本版注册初值为 true，因此静止画面未必全绿。[页更新条件](G:/UnrealEngineInstalled/UE_5.7/Engine/Shaders/Private/VirtualShadowMaps/VirtualShadowMapPhysicalPageManagement.usf:333)

WPO 和骨骼动画可在 Transform 不变时改变遮挡形状；旋转方向光改变投影缓存键；改变物理池尺寸/标志会重建资源并丢缓存。这些问题不能仅靠停止相机移动解决。强制 Static 会抑制某些失效，在不符合静态承诺的对象上可能留下旧阴影，而不是无损地去掉成本。

## 题 5

可以。局部 One Pass 阴影投影独立于旧 clustered deferred 的运行选择，也不需要 HWRT。方向光仍走相应每灯投影。预算 16 时，超出打包预算的灯在适用消费者里有单次 VSM 查询回退，不保证享有同样 SMRT 过滤，也不是一律无阴影。可视化 ShowFlag 会关闭 One Pass，计时要回到普通视图。

```text
Cb2-Cb1 = (-0.1,-0.15,-0.2)
Q2-Q1 = (1-0.35)*(Cb2-Cb1)
      = (-0.065,-0.0975,-0.13)
```

这是背景经普通透明合成传到 Q 的线性差值，薄片自身发光仍可保持不变。不能据此宣称 Unlit 薄片获得 Lit VSM 表面响应，更不能当作色调映射后显示 RGB 的实测变化。
