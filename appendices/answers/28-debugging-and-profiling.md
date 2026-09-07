# 第 28 章答案：缓冲观察与性能实践

[返回本章](../../chapters/28-debugging-and-profiling.md) · [返回目录](../../README.md)

## 题 1

先确认普通 Unlit Translucent 蓝片、主不透明 GBuffer/SceneDepth、透明颜色目标及其合成时点。普通透明通常不写这份不透明深度和材质数据，Q 的背景仍可以来自方块；再检查透明 Draw 的深度/混合状态、Separate 的 D/T 和合成回 SceneColor 的消费者。

Buffer Visualization 用材质解码和映射数据，部分颜色通道应用曝光；还要区分预曝光、纹理格式、ViewRect、后处理阶段和输出编码。截图不是原始资源字节，也不自动是可直接带入光照公式的线性数值。

## 题 2

```text
父 exclusive = 5-2-1 = 2 ms
队列重叠 = [2,4] + [6,7] = 3 ms
忙碌并集 = 8+5-3 = 10 ms
```

父 inclusive 已包含子项，不能重复加。两队列工作重叠，也不能直接加成 13 ms；最长单队列 8 ms 又漏掉 Compute 的独立忙碌。CPU/GPU 流水、等待、Present 与显示扫描有各自时间边界，busy 并集不是输入到显示延迟。本版还优先接受平台提供的 GPU frame time，缺少时才走并集回退。

## 题 3

先用 `Trace.Status` 确认没有要保留的已有会话，然后执行：

```text
Trace.File cpu,gpu,frame,bookmark,task
Trace.Bookmark Chapter28_Begin
```

完成规定操作后：

```text
Trace.Bookmark Chapter28_End
Trace.Stop
Trace.Status
```

按日志实际路径打开文件。CPU Submit 只是提交侧工作，不是 GPU 执行耗时。检查 Gpu 通道、Trace/GPU profiler 编译条件、RHI 时间戳支持、正确进程与记录区间、分析器版本和轨道显示，不能用 CPU scope 冒充 GPU 时间戳。

## 题 4

RenderDoc 适合 API 事件、绑定、纹理和状态；DumpGPU 适合 RDG Pass 的资源/参数；ProfileGPU 适合一帧 GPU 事件和队列统计。连续卡顿和等待需要 Trace。

VSM 颜色只是特定缓存状态，动态 Receiver Mask 和显示模式会改变解释，而且可视化禁用 One Pass。DumpGPU 加入读回/等待/文件处理；关闭 Shader 优化也改变程序工作。因此要恢复普通 B、等待缓存稳定，在相同配置/列/队列口径下重新测量。单帧捕获未必含缓存最初生成史。

## 题 5

先搜完整事件字符串定位 `FComputeShaderUtils::AddPass`，再跟 `AddTemporalAAPass`、参数结构和 `FTemporalAACS` 注册，到 `TemporalAA.usf:MainCS`，并反查当前颜色/深度/速度/历史的生产者与输出消费者。核验运行的是 TAA 还是 TSR、质量/分辨率、历史有效性与平台排列。

`0.75²=0.5625` 只是线性尺寸比例实际采用时的像素量算例。动态分辨率和编辑器策略可覆盖设置，部分资源不随其缩放，几何/固定开销不按像素变化。记录实际输入/输出 ViewRect、资源尺寸、AA、动态分辨率、场景、缓存、工具模式、唯一变量、稳定区间和多帧统计，才能讨论收益。
