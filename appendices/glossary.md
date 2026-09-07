# 术语表：先辨清概念，再记英文

[返回目录](../README.md) · [第一章](../chapters/01-from-scene-to-pixel.md) · [资源关系](resource-flow.md)

本表随已完成章节扩充。解释用于快速找回上下文，不替代正文算法与源码解读。英文大小写通常不影响概念含义，但 C++ 符号的大小写需要与源码一致。

## 图像与表面

| 中文 | 英文／缩写 | 含义与容易混淆的地方 |
|---|---|---|
| 渲染 | Rendering | 按观察条件将场景数据计算成图像，包括工作组织与实际计算 |
| 像素 | Pixel | 图像网格中的位置；同一位置不永久属于某个物体 |
| 采样 | Sampling | 按规则取得或估计某处的信息；一个最终像素可能结合多个样本和历史 |
| 片元 | Fragment | 图元对图像某处的潜在贡献，仍需满足相关条件才可能影响结果 |
| 分辨率 | Resolution | 图像的宽与高；输出、内部场景和阴影资源可有不同尺寸 |
| 红绿蓝颜色表示 | RGB | Red、Green、Blue 三个分量；数值含义取决于颜色空间与所在阶段 |
| 不透明度／混合分量 | Alpha | 在简单混合中控制前景权重；不能推断所有资源第四通道都是同一用途 |
| 线性颜色 | Linear Color | 可按线性光量关系进行相关加法与比例计算的颜色表示 |
| 标准 RGB | sRGB | 常见颜色空间及编码约定；编码后的数字不能随意当作线性光量相加 |
| 高动态范围 | HDR, High Dynamic Range | 表达较宽亮度范围；线性场景分量可以大于 1，不等同于 HDR 显示已经启用 |
| 标准动态范围 | SDR, Standard Dynamic Range | 常规显示动态范围；本书首批以普通 SDR 输出作解释 |
| 曝光 | Exposure | 控制场景光量向后续显示尺度的映射；Unlit 材质也不自动绕过它 |
| 预曝光 | Pre-Exposure | 对内部场景颜色使用的曝光相关数值缩放，需要了解约定才能读原始数值 |
| 色调映射 | Tone Mapping | 将场景高动态范围颜色映射到目标显示与外观所需的表示 |
| 混合 | Blending | 按指定规则组合已有目标与新的贡献；不同模式和顺序会产生不同结果 |
| 非预乘 Alpha | Straight / Non-Premultiplied Alpha | 源 RGB 尚未预先乘 Alpha；简单 source-over 可用 `αCs+(1-α)Cb` |
| 预乘 Alpha | Premultiplied Alpha | 源 RGB 已含 Alpha 的乘积；不能再不加区分地用源 Alpha 乘一次 |

## 几何与可见性

| 中文 | 英文／缩写 | 含义与容易混淆的地方 |
|---|---|---|
| 网格 | Mesh | 组织表面几何的数据，基础路径通常使用三角形 |
| 顶点／索引 | Vertex / Index | 顶点提供位置及其他属性；索引用编号引用顶点，减少重复 |
| 静态网格 | Static Mesh | 网格资产和相关渲染方式，不等于使用它的 Actor 永远不能移动 |
| 图元 | Primitive | GPU 语境中的三角形等基本几何单位；UE 场景 Primitive 是更高层的组件表示 |
| 变换 | Transform | 平移、旋转、缩放及其组合 |
| 局部空间／世界空间 | Local Space / World Space | 前者相对模型自身坐标系；后者使用场景共同坐标系 |
| 法线 | Normal | 描述表面朝向的向量；不同来源的法线不必相同 |
| 切线 | Tangent | 沿表面的方向，参与构造局部表面坐标与法线贴图等计算 |
| 纹理坐标 | UV / Texture Coordinates | 表面查找纹理数据的位置；不是屏幕列号与行号 |
| 透视投影 | Perspective Projection | 将三维观察关系映射到图像形成所需表示，表现近大远小 |
| 视场角 | FOV, Field of View | 可见方向张开的角度；变化不等于模型实际缩放 |
| 视锥 | View Frustum | 相机相关投影定义的可见空间范围 |
| 裁剪 | Clipping | 对跨越裁剪边界的图元进行截取等处理 |
| 剔除 | Culling | 排除不需要继续处理的对象或图元，如视锥外对象 |
| 遮挡剔除 | Occlusion Culling | 利用遮挡关系减少后续几何等工作，不等于单个片元的深度测试 |
| 光栅化 | Rasterization | 根据投影后的图元产生屏幕覆盖和插值相关工作 |
| 深度测试 | Depth Test | 按当前深度约定比较新旧深度，帮助决定是否保留贡献 |
| 反向 Z | Reversed Z | 一种深度映射与比较约定；不能将深度值一概解释为越小越近 |
| 模板测试 | Stencil Test | 利用缓冲中的整数标记与规则限制处理区域；不是 C++ 模板 |
| 层级深度缓冲 | HZB, Hierarchical Z-Buffer | 深度的多层分辨率表示，服务区域深度检索与相关算法 |

## 材质与光照

| 中文 | 英文／缩写 | 含义与容易混淆的地方 |
|---|---|---|
| 材质 | Material | 表面属性与相关求值规则；资产会参与多个用途的 Shader 编译 |
| 着色器 | Shader | GPU 可编程阶段的程序；可以计算位置、颜色或其他数据 |
| 顶点／像素着色器 | Vertex Shader / Pixel Shader | 分别参与顶点和片元等阶段的计算，不代表所有功能都只使用这两类程序 |
| 计算着色器 | Compute Shader | 用线程组组织通用 GPU 计算，不要求按普通三角形光栅化入口执行 |
| 基础颜色 | Base Color | 材质输入属性；不是曝光与显示转换后的最终颜色 |
| 粗糙度 | Roughness | 控制微表面方向分布等响应，影响高光与反射的集中程度 |
| 金属度 | Metallic | 控制传统工作流中的金属／非金属响应分类与相关计算 |
| 镜面反射参数 | Specular | 传统非金属工作流相关参数，不等于任意数值就是相同比例的总反射率 |
| 自发光 | Emissive | 表面自身输出的颜色贡献；是否照亮其他表面取决于相关系统与配置 |
| 默认受光／无光照 | Default Lit / Unlit | 着色模型选择；无光照不意味着无曝光或无后处理 |
| 不透明／半透明／遮罩 | Opaque / Translucent / Masked | 分别处理不透背景、混合前后贡献、依据遮罩保留或丢弃表面的情况 |
| 双面 | Two Sided | 允许按双面相关规则处理表面，不自动创造真实几何厚度 |
| 直接／间接光照 | Direct / Indirect Lighting | 光源直接到表面的贡献／经过其他场景交互后的贡献 |
| 双向反射分布函数 | BRDF | Bidirectional Reflectance Distribution Function，描述入射与出射方向间的表面反射分布 |
| 双向散射分布函数 | BSDF | Bidirectional Scattering Distribution Function，同时容纳反射与透射等表面散射关系 |
| 阴影贴图 | Shadow Map | 从光源相关投影获得的深度等数据，用于判断受光表面是否被遮挡 |
| 级联阴影贴图 | CSM, Cascaded Shadow Maps | 将相机相关范围分段使用不同阴影投影，常见于方向光 |
| 屏幕空间反射 | SSR, Screen Space Reflections | 利用屏幕可得信息近似反射，存在屏幕外、遮挡与材质等限制 |
| 贴花 | Decal | 叠加污迹、标记等表面变化；DBuffer、GBuffer 与发光贴花有不同阶段 |

## 引擎与执行

| 中文 | 英文／缩写 | 含义与容易混淆的地方 |
|---|---|---|
| 场景对象／组件 | Actor / Component | Actor 组织游戏实体，组件提供网格、灯光等具体能力 |
| UE 基础对象类型 | UObject | UE 对象系统的基础类型，不是 GPU 绘制数据格式 |
| 渲染代理 | Scene Proxy | 组件在渲染侧的重要表示，配合场景登记与更新管理 |
| 视图／视图族 | View / ViewFamily | 单次观察数据／相关视图与共同设置的组织 |
| 移动性 | Mobility | 组件的 Static、Stationary、Movable 等设置，影响更新与光照行为 |
| 渲染阶段 | Render Pass | 围绕输出组织的绘制、计算等工作；概念阶段可能包含多个 RDG Pass |
| 基础通道 | Base Pass | 基础路径中求值表面材质、写相关目标的重要阶段，可包含颜色贡献 |
| 几何缓冲集合 | GBuffer | 记录表面属性供后续使用，格式与数量受配置影响 |
| 深度预通道 | Depth Prepass | 在 Base Pass 前先处理适用几何深度；范围和是否执行有条件 |
| 延迟渲染 | Deferred Rendering | 先记录表面属性，再做一部分光照；不是指固定多延迟一帧 |
| 渲染依赖图 | RDG | Render Dependency Graph，用任务和资源读写关系组织工作 |
| 渲染硬件接口 | RHI | Render Hardware Interface，UE 对底层图形平台的接口抽象 |
| 命令列表／队列 | Command List / Queue | 前者记录工作，后者组织提交和执行；记录完成不等于 GPU 完成 |
| 屏障／同步 | Barrier / Synchronization | 确保资源访问、状态与执行满足条件，不等于每一步全设备停等 |
| 异步计算 | Async Compute | 适用计算可走异步队列路径；是否重叠以及是否加速需要实际分析 |
| 后处理 | Post Processing | 利用场景图像等资源继续处理与重建结果，具体组织依配置而变 |
| 时间抗锯齿 | TAA | Temporal Anti-Aliasing，结合当前与历史信息抑制时空锯齿 |
| 时间超分辨率 | TSR | Temporal Super Resolution，UE 的时间重建路径之一，不能默认在 TAA 后再完整跑一遍 |
| 后处理体积 | PPV, Post Process Volume | 为视图提供区域性或全局的后处理参数 |
| 景深／运动模糊 | DOF / Motion Blur | 对焦相关模糊／曝光时间内运动相关模糊；不是同一种来源 |
| 图形 API | Graphics API | 程序与图形平台之间的接口规范，本书具体后端为 Direct3D 12 |
| 着色器模型 | SM6, Shader Model 6 | Shader 的能力体系，不是某个抗锯齿选项或硬件光追开关 |
| 后备缓冲／交换链 | Back Buffer / Swap Chain | 用于呈现的图像资源／管理相关缓冲与呈现行为的机制 |
| 呈现 | Present | 请求显示系统使用准备的图像；函数返回不等于屏幕扫描结束 |
| UE 界面框架 | Slate | 组织窗口与 UI 等绘制，可能使用独立于主场景的 RDG |
| 编辑器内运行 | PIE, Play In Editor | 通过编辑器运行游戏的方式，不能默认拥有与 Standalone 相同的观察条件 |

## 基础篇补充：变换、采样与时间信息

| 中文 | 英文／缩写 | 含义与边界 |
|---|---|---|
| 点／向量 | Point / Vector | 点表示位置；向量可表示位移或方向。空间与单位必须一致才能组合 |
| 点积 | Dot Product | 分量乘积之和；与单位方向做点积可得到沿该方向的分量 |
| 单位向量 | Unit Vector | 长度为 1 的方向表示；零向量不能靠除以长度得到有效方向 |
| 齐次坐标 | Homogeneous Coordinates | 用额外分量统一表达变换与比例关系；点的 w 不等于颜色 Alpha |
| 透视除法 | Perspective Divide | 按裁剪位置的 w 恢复投影比例；不是在除世界空间 Z 高度 |
| 归一化设备坐标 | NDC, Normalized Device Coordinates | 投影后经除法得到的设备相关坐标范围，不是纹理 UV |
| 平移世界空间 | Translated World Space | 保留世界轴、将原点平移到视图附近；不同于旋转后的视图空间 |
| 大世界坐标 | LWC, Large World Coordinates | UE 的大范围精度策略，不意味着所有 GPU 运算都使用原生 double |
| 双部分浮点表示 | DoubleFloat | 用高低部分及配套运算表达较高精度值；与 HLSL 原生 double 不同 |
| 重心坐标 | Barycentric Coordinates | 三角形内部的三个权重；屏幕权重用于普通属性时还可能需透视校正 |
| 上／左边归属规则 | Top-left Rule | 为一致共享边上的样本分配唯一归属，条件依坐标和绕序约定 |
| 辅助调用 | Helper Invocation | 支持邻域计算等用途的执行，不一定提交有效像素颜色 |
| 四位置组 | Quad | 像素阶段中常用的 2×2 邻近位置关系，帮助理解导数与小三角成本 |
| Shader 变体 | Shader Permutation | 按平台、功能及参数组合得到的编译变体；不是所有选择都属于运行时 if |
| 管线状态对象 | PSO, Pipeline State Object | 将 Shader 和相关固定状态等组织为底层可用配置；不等于单个材质资产 |
| 资源视图 | Resource View | SRV、UAV、RTV、DSV 等访问形式，区别于摄像机视图 |
| 纹理元素 | Texel | 纹理网格中的元素；屏幕一个像素可能读取多个不同纹理的元素 |
| 采样器 | Sampler | 描述过滤、寻址等访问规则；不等于纹理资源本身 |
| 细节层级 | Mip Level | 同一纹理的不同分辨率表示，不是另一帧历史或数组切片 |
| 无符号归一化 | UNORM | 将有限整数编码按约定归一化，常见 8 位通道 0～255 对应 0～1 |
| 资源尺寸／视图矩形 | Extent / View Rect | 整张资源的范围／本视图在其中的有效区域，可能有偏移与大小差异 |
| 传递函数 | Transfer Function | 在线性量与编码量之间映射；完整颜色空间还涉及基色和白点等约定 |
| 重投影 | Reprojection | 寻找当前信息在先前视图中的对应，不是直接平均相同屏幕坐标 |
| 新显露区域 | Disocclusion | 当前看见但之前被挡住的表面，历史可能不存在有效对应 |
| 拖影 | Ghosting | 时间处理错误保留旧信息的典型现象之一，原因需结合具体算法检查 |
| 图外资源登记 | Register External Resource | 将由图外生命周期持有的资源纳入当前 RDG 使用，不表示网络资源 |
| 资源提取 | Resource Extraction | 将图内适用结果交给图外持有，便于后续使用；不等于 CPU 读回 |

## 场景与执行架构

下列术语沿第 06～09 章展开；这里提供查阅入口，详细条件仍看对应正文。

| 中文／标识 | 英文 | 用途与区别 |
|---|---|---|
| 场景内部记录 | Scene Info | 保存 primitive 或灯光在渲染场景中的索引、管理关系和缓存，区别于游戏组件 |
| 注册／注销 | Registration / Unregistration | 接入或退出组件相关世界系统；不等于 UObject 创建或最终销毁 |
| 脏标记 | Dirty Flag | 表示某类更新待处理，不是消费完成通知 |
| 包围体 | Bounds | 用简单范围支持空间筛选；不替代精确表面覆盖或深度 |
| 轴对齐包围盒 | AABB | 用沿坐标轴的范围包住物体，旋转物体可使世界 AABB 变宽 |
| 动态材质实例 | MID, Material Instance Dynamic | 运行时覆盖材质参数，普通参数更新与替换组件材质不同 |
| 持久视图状态 | View State | 保留跨次观察所需数据；当前 FSceneView 可以重新构造 |
| 视图扩展 | View Extension | 在约定阶段参与观察设置或渲染相关工作 |
| 场景捕获 | Scene Capture | 为特定目标请求额外观察，部分条件下可并入主 Renderer |
| 命令管道 | Command Pipe | 组织 CPU 渲染命令的调度和重放，不等于 GPU 队列 |
| 线程局部存储 | TLS, Thread-Local Storage | 保存每条 CPU 线程自己的上下文，不会自动复制所有场景数据 |
| 记录／翻译／提交 | Recording / Translation / Submission | 保存引擎操作、形成后端工作、交给提交机制；均不等于 GPU 已完成 |
| 旁路 | Bypass | 跳过适用 RHI 命令记录层，不跳过底层设备命令和异步执行 |
| 提交载荷 | Submission Payload | 后端用于组织命令列表、等待与信号等的一批工作 |
| 栅栏 | Fence | 跟踪特定执行边界；必须说明 RenderThread、RHI、GPU 或 Swapchain 等范围 |
| 读回 | Readback | 将 GPU 结果复制到 CPU 可读取资源，并在完成条件满足后访问 |
| 吞吐率／延迟 | Throughput / Latency | 稳定单位时间完成量／一份输入到指定结果的时间，不是同一个量 |
| 关键路径 | Critical Path | 目标完成前必须依次满足的工作与约束链 |
| 任务裁剪 | Pass Culling | 删除不贡献必要输出的图节点，区别于几何可见性裁剪 |
| 瞬态资源 | Transient Resource | 按有限使用区间管理的资源；使用范围还须考虑跨队列依赖 |
| 内存别名复用 | Memory Aliasing | 不冲突的资源使用区间复用底层存储，需要兼容分配与同步 |
| 子资源 | Subresource | 纹理 Mip、数组层等访问范围，读写冲突需按实际声明判断 |
| 写后读／读后写／写后写 | RAW / WAR / WAW | 三类访问冲突，分别保护生产结果、未完读取与写入顺序 |
| 图编译 | Graph Compilation | 分析渲染任务与资源关系，不是 HLSL 的 Shader 编译 |
| 线程组 | Thread Group | 计算 Shader 的一组设备调用；组数与每组线程数要分开 |

## 设备执行、可见性与深度

以下词汇对应第 10～13 章，具体源码条件以正文为准。

| 中文／标识 | 英文 | 用途与区别 |
|---|---|---|
| 根签名 | Root Signature | D3D12 约定 Shader 参数如何从根参数与描述符访问，不等于资源内容 |
| 描述符 | Descriptor | 描述一种资源访问视图；描述符、资源对象和底层内存分属不同层次 |
| 命令分配器 | Command Allocator | 管理原生命令记录所用存储，设备仍使用时不能任意重置 |
| 增强屏障 | Enhanced Barriers | D3D12 的同步／访问／布局描述方式；是否启用须看当前 RHI |
| GPU 场景数据 | GPU Scene | 供 Shader 索引的 primitive／instance 等记录，不是一张已渲染场景图 |
| 散布上传 | Scatter Upload | 按目的索引把准备好的记录写到目标缓冲的相应位置 |
| 相关性 | Relevance | 对象与某个 View 和 Pass 的关系，候选可见不保证所有 Pass 都参与 |
| 保守筛选 | Conservative Culling | 证据不足以排除时保留；基于历史的预测仍有有效性边界 |
| 绘制命令缓存 | Cached Mesh Draw Commands | 复用适用 Pass 的 Shader、状态与绑定描述，不缓存最终像素 |
| 可见命令包装 | Visible Mesh Draw Command | 给通用 MDC 附加本视图的实例、排序等信息 |
| 动态实例化 | Dynamic Instancing | 将适合共享状态的绘制工作压紧；不是改变组件 Mobility |
| 间接绘制参数 | Indirect Draw Arguments | 由缓冲提供绘制计数等参数，适用时可由 GPU 筛选产生 |
| 深度预通道 | Depth Prepass | 先绘制适用几何产生深度，有绘制和带宽成本 |
| 提前深度测试 | Early-Z | 在语义允许时提前拒绝样本；不等于对象遮挡剔除 |
| 遮挡者／被测对象 | Occluder / Occludee | 提供遮挡的几何／当前正在判断是否被挡住的候选 |
| 最远／最近 HZB | Furthest / Closest HZB | 本书反向 Z 下分别以 min／max 归约，不是颜色平均 Mip |
| 像素深度偏移 | Pixel Depth Offset, PDO | 材质对输出深度的适用修改，影响深度 Shader 与一致性条件 |
| 硬件遮挡查询 | Hardware Occlusion Query | 汇报适用绘制通过测试的样本相关结果；回读可能有等待 |

## 表面属性、阴影与光照

以下术语对应第 14～17 章；相同名称的 Alpha、深度或颜色必须结合生产者和消费者解释。

| 中文／标识 | 英文 | 用途与区别 |
|---|---|---|
| 多渲染目标 | MRT, Multiple Render Targets | 一次适用绘制绑定多个颜色附件；附件索引由当前布局决定 |
| 贴花缓冲 | DBuffer | 在适用 Base Pass 前保存贴花属性，供接收方合入材质；不是最终颜色贴纸 |
| 着色模型标识 | Shading Model ID | 让后续 Shader 选择表面响应分支，不是 Actor ID 或透明度 |
| 阴影投射者／接收者 | Shadow Caster / Receiver | 遮住光的几何／判断光源可见性的表面；主相机不可见仍可能投影 |
| 级联阴影贴图 | CSM, Cascaded Shadow Maps | 按主视图距离组织多个方向光投影范围；分割公式须核对版本 |
| 百分比渐近过滤 | PCF, Percentage-Closer Filtering | 对多次深度比较结果加权，不能替换成先平均深度再比较 |
| 阴影偏置 | Shadow Bias | 缓解深度与离散采样自遮挡误差，过大可能造成阴影脱离 |
| 光源体积 | Light Volume | 用球／锥等覆盖局部光可能影响的屏幕范围，不是可见灯具模型 |
| 光照网格 | Light Grid | 按屏幕区块及适用深度层索引灯光，延迟路径也可能准备或消费 |
| 直接／间接光照 | Direct / Indirect Lighting | 光源直接到达表面／经过其他表面传播的贡献，与漫反射／镜面分类独立 |
| 环境遮蔽 | AO, Ambient Occlusion | 近似局部可见性或遮挡信息；实际影响哪部分颜色取决于消费者 |
| 屏幕空间环境遮蔽 | SSAO, Screen-Space Ambient Occlusion | 从当前深度及适用法线邻域估计 AO，看不到完整离屏几何 |
| 屏幕空间反射 | SSR, Screen-Space Reflections | 在当前屏幕深度中搜索反射命中，再采样适用颜色来源；颜色可来自历史 |
| 反射回退 | Reflection Fallback | 对未由当前反射输入覆盖的部分使用适用 Capture／Sky 等来源；配置 A 没有这些环境源 |

## 透明、时间重建与输出

以下术语在第 18～21 章展开；“颜色”“Alpha”“历史”的具体意义均取决于当前资源约定。

| 中文／标识 | 英文 | 用途与区别 |
|---|---|---|
| 非预乘／预乘 Alpha | Straight / Premultiplied Alpha | RGB 是否已包含覆盖权重；混合因子必须与输入约定配套 |
| 分离透明 | Separate Translucency | 先在独立目标积累透明贡献，再在适用阶段合成；绘制时刻与合成时刻可以不同 |
| 剩余透射率 | Remaining Transmittance | 本章 Separate 目标 Alpha 的含义；清为 1 后逐层衰减，不等于普通覆盖 Alpha |
| 视锥体素 | Froxel, Frustum Voxel | 体积雾在相机视锥中划分的三维采样单元，区别于屏幕二维像素 |
| 散射／消光 | Scattering / Extinction | 介质向观察方向加入光／减少沿路径传输的光，积分共同形成雾 |
| 透明光照体积 | TLV, Translucency Lighting Volume | 为适用 Lit 透明提供光照近似；不是 Volumetric Fog 的同名资源 |
| 抖动 | Jitter | 在不同帧偏移采样位置，为时间重建提供不同样本；不是物体运动 |
| 去遮挡 | Disocclusion | 原被挡住的表面在本帧显露，旧像素可能没有可用历史 |
| 历史拒绝／限制 | History Rejection / Clamping | 识别或限制不可靠历史，降低拖影；不能创造当前与历史都缺失的真实细节 |
| 历史引导数据 | History Guide | TSR 比较历史与当前信号的辅助数据，区别于高频颜色历史 |
| 预曝光 | Pre-Exposure | 内部场景颜色的尺度变换；手动模式的取值与自动模式历史路线不同 |
| 曝光值 | EV100, Exposure Value at ISO 100 | 用对数表达相机曝光条件；用于曝光计算，不直接等于最终屏幕亮度 |
| 色调映射 | Tone Mapping | 将场景亮度范围映射到目标显示表示，可能伴随颜色分级和编码 |
| 查找表 | LUT, Look-Up Table | 预计算映射；没有用户外部 LUT 也可能生成内部颜色分级 LUT |
| 双向散射分布函数 | BSDF, Bidirectional Scattering Distribution Function | 描述表面反射和透射响应；使用 BSDF 不意味着材质一定透明 |
| 材质薄层 | Slab | Substrate 的基本散射层表达，携带粗糙度、漫反射、镜面等参数 |
| 散射闭包 | Closure | 编译后保留的一份散射响应及关联数据；不是材质编辑器的任意一个节点 |
| 参数混合 | Parameter Blending | 先合并适用参数再求值，成本与多响应求值不同；非线性使二者通常不等价 |
| 混合式／自适应 GBuffer | Blendable / Adaptive GBuffer | Substrate 的两种运行表示；本书 B 固定为 Blendable，不能套用所有 Adaptive 容器描述 |
| 材质区块分类 | Material Tile Classification | 根据区域内所需材质复杂度组织后续工作，B 也可以存在这一阶段 |

## UE5 功能名称

以下是功能定位，具体算法和启用条件在第 21～25 章核对展开。

| 名称 | 初步定位 | 本书的对照位置 |
|---|---|---|
| Substrate | 材质表达及相关编译与运行时处理系统 | 从传统材质切换到明确的 Blendable GBuffer 配置 |
| Nanite | 为适用几何体组织细节、可见性、光栅化与材质求值的系统 | 不透明网格的几何处理分支 |
| Lumen | 动态全局光照与反射系统 | 先软件追踪，再讨论硬件追踪 |
| VSM, Virtual Shadow Maps | 虚拟阴影贴图 | 常规阴影的分页与按需更新对照 |
| HWRT, Hardware Ray Tracing | 硬件光线追踪 | 适用功能的追踪路径，不自动替换所有光栅化 |
| MegaLights | 适用直接光照的采样和阴影组织方案 | A／B 主线关闭，单独解释分支 |

## 虚拟几何与光照表示

| 中文／标识 | 英文 | 用途与区别 |
|---|---|---|
| 几何簇 | Cluster | Nanite 的局部几何工作单位，不等于传输页面或整个 Actor |
| 可见性缓冲 | Visibility Buffer, VisBuffer | 记录胜出的可见表面定位与深度信息，后续还要恢复材质输入 |
| 光栅／着色分桶 | Raster / Shading Bins | 分别组织几何光栅状态与可见表面的材质工作，不是同一套分组 |
| 辅助执行通道 | Helper Lane | 为导数等计算保持必要的邻域执行，不保证每个执行通道都导出一个可见像素 |
| 虚拟页／物理页 | Virtual / Physical Page | 逻辑地址区域与实际存储槽位；已映射也不代表内容仍有效 |
| 页表 | Page Table | 从虚拟地址找到物理槽位和有效层级；细页回退指针不等于可按细级写入 |
| 裁剪层级映射 | Clipmap | 以逐层扩大范围覆盖观察者附近的方向阴影，不等于局部光同投影的 mip |
| 阴影贴图射线追踪 | SMRT, Shadow Map Ray Tracing | 沿候选方向查询阴影深度近似软阴影，不使用 HWRT 三角形 BVH |
| 接收者掩码 | Receiver Mask | 描述 VSM 页内当前需求区域，影响剔除与动态缓存完整性 |
| 有符号距离场 | SDF, Signed Distance Field | 几何距离近似，供软件追踪查询；不直接保存材质颜色 |
| 网格卡片 | Mesh Card | 以适用捕获方向组织表面参数化，不是主画面的平面替身 |
| 表面缓存 | Surface Cache | 卡片材质和光照图集；与世界空间辐射探针不同 |
| 屏幕探针 | Screen Probe | 从当前可见表面布置的方向采样位置，结果可供邻近像素共享 |
| 辐射缓存 | Radiance Cache | 世界空间可复用的方向光信息，减少重复远距离采样 |
| 辐射度传播 | Radiosity | 本书 Lumen 中的卡片低频间接传播与反馈，不是每像素完整路径积分 |
| 八面体方向映射 | Octahedral Mapping | 将单位方向编码到二维图，不表示固定只采八条光线 |
| 射线压紧／压缩 | Ray / Trace Compaction | 收集尚需处理的工作索引并生成计数，区别于图像压缩 |
| 包围体层次结构 | BVH, Bounding Volume Hierarchy | 先排除成组几何再求交；真实硬件布局由实现决定 |
| 底层／顶层加速结构 | BLAS / TLAS | 分别组织几何内部与场景实例；不保证每帧只有一份结构 |
| 内联光线追踪 | Inline Ray Tracing | Shader 控制流中推进硬件查询，Compute 标签不足以区分软／硬追踪 |
| 光线生成着色器 | Ray Generation Shader, RayGen / RGS | 发起射线并组织结果处理，可配合命中与未命中 Shader |
| Shader 绑定表 | Shader Binding Table, SBT | 把适用记录与 Shader／参数连接，区别于空间加速结构 |
| 命中光照 | Hit Lighting | 在命中处计算适用材质与光照，区别于找到表面后取缓存结果 |
| 重要性采样 | Importance Sampling | 按估计贡献分配样本概率并配套权重；历史不准确会影响效率 |
| 光线锥 | Ray Cone | 估计传播后采样足迹、辅助纹理过滤，不是实际发出无穷多射线 |

Radiance 在不同中文资料中也写作“辐亮度”或“辐射亮度”，本书二者均指同一物理量；不能与整合入射方向得到的 Irradiance（辐照度）直接互换。

相关实现定位见 [源码索引](source-index.md)，概念补充阅读见 [参考资料](references.md)。
