# 第 19 章理解检查题参考答案

[返回正文](../../chapters/19-velocity-taa-tsr.md) · [返回答案目录](../exercise-answers.md)

## 1. 相机运动产生速度

即使立方体 LocalToWorld 不变，相机矩阵改变也会使同一世界点投影到不同屏幕位置。速度 Shader 使用当前 `TranslatedWorldToClip` 和上一帧 `PrevTranslatedWorldToClip`，并在 TAA 侧用 `ClipToPrevClip` 推导相机反投影。速度是屏幕位移，不是物体 LocalToWorld 的唯一差值。[速度顶点矩阵](G:/UnrealEngineInstalled/UE_5.7/Engine/Shaders/Private/VelocityShader.usf:80)

这不保证静态方块在原始 Velocity 中显式写出相机速度：未写入时，TAA/TSR 可以用主深度与相机矩阵恢复它。不同深度的表面有不同平移视差，共用相机矩阵不等于二维速度相同。

## 2. 速度写入位置

值 1 在普通 Base Pass 增加速度目标；值 2 让 Base Pass 结束后再执行独立 `RenderVelocities`，需要再次组织速度 Mesh Pass，改变资源声明、绘制成本和与后续阶段的依赖。值 0 则把速度放入深度 Pass，并拆分有速度与无速度阶段。该变量是只读、会影响 Shader 编译组合，不能当普通后处理开关。[注册说明](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/VelocityRendering.cpp:29)

## 3. TAA 融合算例

忽略其他修正时，若当前帧权重 `w=0.04`，可用教学式 `Result=0.8×(1-w)+0.2×w`：

```text
Result = 0.8×0.96 + 0.2×0.04 = 0.768 + 0.008 = 0.776
```

实际 TAA 还要经过 HDR 权重、速度相关历史模糊、3×3 邻域裁剪、CameraCut、曝光校正和输出格式量化；源码参数中的 0.04 只是 CVar 注册初值，项目和运行时可覆盖，因此 0.776 不是 UE 实际像素值。[当前帧权重](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/PostProcess/TemporalAA.cpp:41)

## 4. 速度正确仍可能拖影

第一，历史颜色可能来自屏幕外映射或在深度／速度边界处不再对应当前表面；TAA 会用 OffScreen、CameraCut 和邻域 min/max 限制减轻，但保守范围仍可能保留旧颜色。第二，当前帧权重较低时，稳定性优先会让旧颜色保持较久；提高权重又可能带来抖动。第三，透明 Q 可能没有独立速度，历史只得到背景或相机近似。

源码中 `ClampHistory`、`IgnoreHistory` 与速度采样分别处理这些情况，却不能创造不存在的前帧表面。本版 `AA_CLIP` 默认 0，常用限制是分量 `clamp`，不要因为同文件有 `HistoryClip` 函数就把它当成本章必经步骤。[历史与邻域限制调用](G:/UnrealEngineInstalled/UE_5.7/Engine/Shaders/Private/TemporalAA.usf:2134)

## 5. TSR 像素数量与细节边界

输入像素数为 `960×540=518400`，输出为 `1920×1080=2073600`，比值是 `4`。TSR 可以结合多帧抖动、邻域和历史重建输出更多像素，但新像素的信息受输入 Scene Color、Depth、Velocity 和历史有效性限制。屏幕外、遮挡后、透明速度缺失或快速变化区域会触发拒绝、空间抗锯或较少历史融合，不能把四倍输出像素数等同于三倍真实几何细节。[TSR 更新与解析](G:/UnrealEngineInstalled/UE_5.7/Engine/Source/Runtime/Renderer/Private/PostProcess/TemporalSuperResolution.cpp:3122)
