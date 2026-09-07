# 第 25 章参考答案

[返回正文](../../chapters/25-hardware-ray-tracing.md) · [全部答案](../exercise-answers.md)

这些答案解释本地 UE 5.7.4 源码条件与教学模型，不声称实际运行了 HWRT 或 MegaLights。

## 题 1：实例运动与几何变形

纯平移改变实例变换，几何在自身空间中的三角形关系保持不变，概念上可复用适用 BLAS，再更新顶层实例表示。WPO 改变相对顶点位置，相关光追几何也需要更新；是否能 Update／Refit，还是必须 Build，要看初始化标志、拓扑和具体支持。

同一资产可以有不同几何、LOD、驻留、变形或材质处理路线，不能保证所有运行实例必然共享一份 BLAS。本版 `FRayTracingScene::Build` 按 Layer 与 Active View 组织构建，也不能保证全帧只有一棵 TLAS。参照 [RayTracingScene.cpp:478](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/RayTracing/RayTracingScene.cpp:478)。

## 题 2：Compute 与硬件求交

不能。Inline HWRT 可以在 Compute Shader 中通过 RayQuery 遍历硬件加速结构；软件 Lumen 的距离场追踪也常在 Compute 中执行。

先定位当前 Shader 类型和编译变体，再检查是否绑定 TLAS／相关加速结构，还是主要读取 Mesh SDF／Global SDF。继续看调用的是 Inline 查询还是距离场步进，并核验项目、设备、View 与功能开关。Lumen 反射同一源文件注册 CS／RGS 的例子见 [LumenReflectionHardwareRayTracing.cpp:341](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/Lumen/LumenReflectionHardwareRayTracing.cpp:341)。单凭名字不能代替实际资源和分支证据。

## 题 3：求交与命中光照

不矛盾。HWRT 决定如何找到表面；LightingMode 0 仍从 Surface Cache 取得相关命中光照，因此卡片覆盖、分辨率和更新仍影响结果。本版模式 2 为适用反射启用 Hit Lighting，增加完整材质和相应光照／阴影工作，GI 与后续传播仍可使用 Surface Cache。

几何代理误差、缺失实例、驻留、错误法线、材质支持和不可靠历史不会因为命中光照开启自动消失。需要 RayGen 等支持且实际 View 进入对应模式；本版反射 Hit Lighting 分支显式不用 Inline。参照 [LumenReflectionHardwareRayTracing.cpp:721](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/Lumen/LumenReflectionHardwareRayTracing.cpp:721)。

## 题 4：采样估计与方差

均匀概率各 0.5 时，结果为 `8/0.5=16` 或 `2/0.5=4`。期望 `0.5*16+0.5*4=10`，方差 `0.5*6^2+0.5*(-6)^2=36`。

第一盏被挡住后，真实总贡献为 2。继续按 0.8、0.2 选择时，结果为 0 或 `2/0.2=10`，期望仍为 2，方差 `0.8*(0-2)^2+0.2*(10-2)^2=16`。若此时均匀选灯，结果为 0 或 4，方差只有 4。

所以按未遮挡亮度进行的重要性估计，可能在真实遮挡后反而浪费样本。历史可见性帮助改善概率，但历史变化与阈值、权重裁剪、滤波又引入额外取舍。这个理想标量估计的无偏性不能直接推广为完整实现的无偏保证。

## 题 5：VSM、方向光与成本

不一定错误。MegaLights 每灯可选择 VSM；此时采样可以参与专属页面请求，再渲染／复用所需页面并查询样本阴影。通用 `BeginMarkVirtualShadowMapPages` 在采样前已开始安排工作；只有样本专属请求必须等选出样本，不能说所有页面标记都在采样后。本章 `r.MegaLights.DirectionalLights=0`，方向光继续原来的路线，点光才进入 MegaLights。源码有默认关闭的方向光扩展和额外灯缓冲条件，不能用旧提示宣布所有条件下绝对不支持方向光。

4 个样本只约束一项采样预算。下采样、有效 Tile、屏幕命中、世界追踪、材质求值、结构更新、VSM 页面、体积、去噪和并行关键路径仍影响成本。1280×720×4=3,686,400 只是所声明简化前提下的名义样本数，既不是捕获射线数，也不是 GPU 毫秒。页面请求顺序见 [DeferredShadingRenderer.cpp:3115](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/DeferredShadingRenderer.cpp:3115)。
