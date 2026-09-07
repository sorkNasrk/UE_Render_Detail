# 第 21 章答案：Substrate

[返回本章](../../chapters/21-substrate.md) · [返回目录](../../README.md)

## 题 1

作者图中的节点数、编译后独立散射响应数和每像素实际有效响应数不是同一量。需要检查目标平台和 GBuffer 格式、Closure 与字节预算、编译器是否做参数混合或特征简化，以及当前材质／视图实际使用的排列。

B 的 Blendable 路线在 `GetClosurePerPixel` 直接返回 1，因此两个 Slab 的图可能合成单一近似响应。修改项目 Closure 值不能绕过这个格式分支。[源码入口](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/RenderCore/Private/RenderUtils.cpp:2124)

## 题 2

```text
D(r) = 1 / (pi*r^4)
D(0.25) ≈ 81.48733
D(0.75) ≈ 1.00602
先求响应再平均 ≈ 41.24667

平均粗糙度 = 0.5
D(0.5) ≈ 5.09296
```

它证明非线性响应通常不满足“参数平均后求值等于求值后平均”。它不是完整 BRDF、最终颜色或 Substrate 实际参数混合算法的逐行模拟；不能凭这个例子断言引擎只采用线性平均 Roughness。

## 题 3

CPU 的材质编译阶段先分析 Front Material 拓扑、合并平台预算并生成 Shader。GPU 当帧才按当前表面和纹理求得 Front 数据。B 经 `SubstrateMaterialExportOut` 得到适配单 Closure 的表面属性，写入生成编码对应的 GBuffer。

延迟光照从 GBuffer 与深度取得当前表面，由 `SubstrateReadGBufferBSDF` 恢复 BSDF，结合灯光、可见性与阴影调用 Substrate 散射求值，最后累加到 Scene Color。这个颜色还不是显示器 RGB。B 的 Nanite 几何求值组织不能被一句“普通网格 Base Pass”覆盖，需继续第 22 章。

## 题 4

分类归约把像素所需类别做位或，主类别按复杂度优先；有 Complex 时，这一块不能只进入 Simple 主类。它不改变其余像素的原始材质属性。

1280/8=160，720/8=90，共 14,400 块。B 仍可分配分类列表和间接参数，但 `NeedsClosureOffsets` 排除 Blendable；分类存在不等于多 Closure Offset 纹理存在。[分类列表写入](G:/UnrealEngineInstalled/UE_5.7/Engine/Shaders/Private/Substrate/SubstrateMaterialClassification.usf:359)

## 题 5

```text
背景系数 = (1-a) + a*T = (0.6,0.8,0.9)
前景贡献 = a*R = (0.05,0.10,0.15)
背景贡献 = (0.6,0.8,0.9)*(0.6,0.4,0.2)
         = (0.36,0.32,0.18)
结果     = (0.41,0.42,0.33)
```

这是题目指定的同一线性表示下、覆盖／透过分离的简化算例。原蓝片的材质模式、发光强度、兼容转换和合成路径另有定义；实际输出还涉及预曝光、时间处理、后处理与颜色编码。未运行和抓帧，不能把这个结果标为实测屏幕颜色。
