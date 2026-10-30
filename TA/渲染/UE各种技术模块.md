---
title: UE 各种技术模块
tags:
  - UnrealEngine
  - TA
  - 渲染
cssclasses:
  - action-note
updated: 2026-10-30
source: https://shimo.im/docs/5xkGo4p2vrTEQMkX/
---

先看模块选用和设置入口，再按需要展开原理、操作步骤与截图。原笔记的主要对照版本是 **UE 5.5 / Unity 2022.3**；更早的经验记录保留了时间与适用条件。

快速跳转：[[#一、模块选用与关键设置|结论与入口]] · [[#二、渲染基础与性能分析|性能工具]] · [[#三、几何复杂度与批处理|几何与合批]] · [[#四、可见性与空间加速|剔除]] · [[#五、光照、阴影与距离场|光照]] · [[#六、纹理流送与显存|纹理]] · [[#七、抗锯齿与时序超分辨率|抗锯齿]] · [[#八、角色材质与毛发|角色与毛发]]

## 一、模块选用与关键设置

### 1.1 先按要解决的问题选模块

- **选择渲染方案**：先定平台、最低硬件、Forward / Deferred 和必需特性，再确定 RHI。DX11 / DX12 是接口后端，不能只凭名称判断快慢。[[#2.1 先区分 RHI、渲染路径与画质特性|查看层次关系]]
- **复杂几何**：评估 Nanite 的支持条件和资产适配；它不代替材质、阴影与帧时间优化。[[#3.1 Nanite：几何复杂度优化|查看 Nanite]]
- **大量重复网格**：优先评估 ISM / HISM；现代 ISM 也支持实例剔除与 LOD，不必一律使用 HISM。[[#3.4 Instanced Static Mesh（ISM）|ISM]] · [[#3.5 Hierarchical Instanced Static Mesh（HISM）|HISM]]
- **动态间接光与反射**：评估 Lumen，并分别检查软件追踪、硬件追踪与目标 RHI 的条件。[[#5.1 Lumen|查看 Lumen]]
- **阴影成本**：同时检查投影灯、投影物体、VSM 页面更新与动态变形；Contact Shadows 用于补充局部接触细节。[[#5.4 阴影|查看阴影]]
- **纹理显存压力**：传统 Streaming 管 Mip；SVT 按瓦片流送；RVT 侧重运行时材质缓存与地形融合。[[#六、纹理流送与显存|查看三种路径]]
- **像素开销与锯齿**：评估 TAA / TAAU / TSR 与内部渲染分辨率，静止和运动画面都要比较。[[#七、抗锯齿与时序超分辨率|查看时序方案]]
- **毛发方案**：先比较发片与 Groom 在目标平台上的质量、LOD 和耗时。Groom 的旧经验记录不能推广为“只适合影视”。[[#八、角色材质与毛发|查看毛发]]

### 1.2 常用设置入口

- **Project Settings → Platforms / Engine → Rendering**：检查目标 RHI、渲染路径、GI / Reflection Method、光追、距离场、阴影和默认抗锯齿。字段名称及支持条件随版本、平台变化。
- **Post Process Volume**：调整 Lumen 等功能的效果与质量参数。带覆盖标记的参数可覆盖对应默认值；它不能替代项目层面的功能启用与平台支持。[[#5.1 Lumen|查看 Lumen 设置]]
- **Static Mesh 资产**：检查 Nanite、传统 LOD、回退网格和距离场等设置。[[#3.1 Nanite：几何复杂度优化|Nanite]] · [[#5.3 距离场（SDF）|距离场]]
- **Actor → Merge Actors**：区分 Merge / Simplify 的几何合并与代理生成，以及 Batch 的实例组织。[[#3.3 合并 Actor|查看使用方式]]
- **蓝图组件 / 实例编辑工具**：用 ISM / HISM 管理重复实例；不要把 Material Instance 当成自动合批开关。[[#3.2 UE 与 Unity：批处理机制的区别|查看批处理区别]]
- **纹理资产与 VT 配置**：检查 Streaming、Virtual Texture Streaming、瓦片和物理池；具体参数与调试命令在纹理章节。[[#六、纹理流送与显存|查看纹理配置]]
- **Screen Percentage 与抗锯齿质量**：固定输出分辨率，比较不同输入比例下的整体 GPU 时间和运动稳定性。[[#7.3 分辨率、画质与性能取舍|查看测试方式]]

### 1.3 性能排查按这个顺序走

1. 固定设备、场景、相机、输出分辨率、画质和帧率限制。
2. 用 **`stat unit` / `stat unitgraph`** 判断慢在 Game、Draw、GPU，还是等待与尖峰。
3. **CPU、线程与卡顿看 Unreal Insights；单帧 GPU 阶段看 `profilegpu`。**
4. 需要检查某个 Draw 的 Shader、网格、纹理与像素结果时，再用 RenderDoc。
5. 一次只改一个变量，在目标设备上比较帧时间和画面，CPU/GPU 时间不要简单相加。

[[#2.5 用 stat 快速判断瓶颈|stat 命令]] · [[#2.6 Unreal Insights：查多帧、线程和卡顿|Insights 操作]] · [[#2.7 ProfileGPU / GPU Visualizer：查单帧渲染阶段|ProfileGPU 操作]]

---

## 二、渲染基础与性能分析

### 2.1 先区分 RHI、渲染路径与画质特性

RHI、Forward / Deferred 和 Lumen / Nanite 等特性属于不同层次，先按平台与需求选型。

> [!note]- 展开：原理、操作、截图与参考
> **RHI 是 UE 对图形 API 的抽象层**，在 Windows 上可使用 D3D11 / D3D12 等后端；它负责把渲染命令落实到图形接口。**Forward / Deferred 是渲染路径**，决定材质和光照如何组织。Lumen、Nanite、VSM、TSR 则是具体功能。它们相互影响，但属于不同层次。
>
> 做选型时，先确定目标平台、最低硬件和必需特性，再核对所用 UE 版本的支持条件。原笔记偏向“高端特性考虑 DX12，低端适配评估 DX11”；实际应在同一场景、分辨率和画质下测试，不能只根据 API 名称判断快慢，也不能笼统认为所有时序抗锯齿都需要 DX12。

### 2.2 一帧是怎样画出来的

一帧可按场景更新、可见性、深度与阴影、Base Pass、光照、透明与后处理、UI 来定位；实际执行会重叠并随路径变化。

> [!note]- 展开：原理、操作、截图与参考
> 以常见的 **UE 桌面延迟渲染路径** 为例，可按以下阶段理解：
>
> 1. **游戏逻辑与场景更新**：Game Thread 更新角色、动画、对象变换等，把需要渲染的状态交给渲染侧。
> 2. **可见性与绘制准备**：Render Thread 及工作线程进行视图准备、可见性判断、LOD 选择、绘制命令整理等工作。
> 3. **深度与阴影相关阶段**：按配置生成场景深度、阴影深度或 VSM 页面；HZB 使用深度信息建立分层深度数据，供相关查询使用。
> 4. **Base Pass**：绘制不透明和 Masked 表面，计算材质并写入 GBuffer 等目标。典型属性包括法线、基础颜色、粗糙度、金属度等；具体写入布局由着色路径决定。
> 5. **光照与反射**：结合表面信息计算直接光、阴影、间接光及反射。启用 Lumen 时，还会出现对应的场景更新、追踪与合成阶段。
> 6. **透明、特效与后处理**：处理透明材质、部分粒子、景深、运动模糊、色调映射等。TAAU / TSR 位于后处理链中，后续部分 Pass 会按重建后的分辨率运行。
> 7. **UI 与最终输出**：完成界面和最终合成，提交到显示目标。
>
> 这是便于定位问题的功能划分，不能当作所有项目都相同的严格执行顺序。前向渲染、移动端、Nanite、异步计算及使用上一帧数据的路径会改变实际安排。CPU 与 GPU 也会流水重叠执行。

### 2.3 BasePass 与 BasePassParallel

**Base Pass 是包含很多 Draw 的阶段；Parallel 表示相关工作并行组织，不代表自动合成一次 Draw。**

> [!note]- 展开：原理、操作、截图与参考
> **Base Pass 是一个渲染阶段，一个阶段里通常包含很多 Draw Call。** 同一个模型还可能在深度、阴影、Base Pass 等不同阶段分别绘制，因此模型数量也不能直接等于 Draw Call 数量。
>
> **BasePassParallel 表示 Base Pass 的并行准备或提交相关工作/事件分组**。UE 会把绘制命令的准备、排序、合并及命令列表录制等工作分给任务线程，减轻 Render Thread 的串行负担。具体标签在 CPU 时间线或 GPU 事件树中的范围，以当前工具及 UE 版本为准。
>
> - 在 **Unreal Insights** 中看到它：优先看 CPU 任务耗时、任务分布和线程等待。
> - 在 **ProfileGPU / RenderDoc** 的事件树中看到它：展开内部事件，查看具体 Draw 和渲染目标。
> - `Parallel` 不保证 GPU 绘制免费，也不表示原本许多物体自动变成一次 Draw。
>
> Base Pass 较贵时，应同时检查材质像素开销、屏幕覆盖面积、Masked / WPO、过度绘制、几何量和提交成本。降低 Draw Call 只解决其中一部分问题。
>
> **BasePassParallel 原始参考截图 37**
>
> ![[TA/渲染/附件/UE各种技术模块/37.png|900]]
>
> 参考：[Epic｜Mesh Drawing Pipeline（含 Mesh Drawing Parallelism）](https://dev.epicgames.com/documentation/en-us/unreal-engine/mesh-drawing-pipeline-in-unreal-engine?application_version=5.5)。

### 2.4 性能工具怎么选：与 Unity Profiler 的对应关系

**Insights 看多帧与线程，ProfileGPU 看单帧 GPU 阶段，RenderDoc 看具体绘制。**

> [!note]- 展开：原理、操作、截图与参考
> - **`stat` 系列**：快速显示当前帧和各系统统计，适合先判断问题方向。
> - **Unreal Insights**：更接近 Unity Profiler 的多帧 Timeline 分析用途。用于查看线程、任务、等待、尖峰、加载等；其 Memory、Asset Loading 等模块需要采集对应的 Trace 数据。
> - **ProfileGPU / GPU Visualizer**：更接近 Unity Profiler GPU 模块中的单帧阶段耗时分析，用来定位哪一个渲染 Pass 花时间。
> - **RenderDoc**：查看某次绘制的管线状态、Shader、纹理、网格及像素结果。它与 Unity Frame Debugger 有部分用途重叠，但不是 Unreal Insights 的替代品。

### 2.5 用 stat 快速判断瓶颈

先用 `stat unit` 判断瓶颈方向，再按需要打开其他统计；60 FPS 的帧预算约为 16.67 ms。

> [!note]- 展开：原理、操作、截图与参考
> 在控制台输入需要的命令，不必一次把全部统计都打开：
>
> ```text
> stat unit
> stat unitgraph
> stat gpu
> stat scenerendering
> stat initviews
> stat rhi
> stat streaming
> ```
>
> - **`stat unit`**：查看 Frame、Game、Draw、GPU、RHIT 等时间。Game 对应游戏线程，Draw 对应渲染线程，RHIT 对应 RHI 线程（是否独立运行取决于配置）。
> - **`stat unitgraph`**：观察帧时间波动，区分持续低帧与偶发卡顿。
> - **`stat gpu`**：观察 GPU 各阶段耗时。
> - **`stat scenerendering`**：查看总体渲染统计。
> - **`stat initviews`**：检查可见性判断、剔除耗时及可见元素数量。
> - **`stat rhi`**：检查 RHI 相关资源和性能统计。
> - **`stat streaming`**：检查传统纹理流送与池压力；虚拟纹理另看第 6 章的 VT 统计。
>
> 帧预算示例：60 FPS 约为 **16.67 ms/帧**，30 FPS 约为 **33.33 ms/帧**。CPU 与 GPU 有重叠，Game、Draw、GPU 的时间不能简单相加；VSync、帧率上限、线程同步和空闲等待也会影响读数。
>
> 参考：[Epic｜Stat Commands](https://dev.epicgames.com/documentation/en-us/unreal-engine/stat-commands-in-unreal-engine?application_version=5.5)。

### 2.6 Unreal Insights：查多帧、线程和卡顿

录制问题片段，在 Timing Insights 中检查慢帧、线程工作、任务和等待；内存与加载分析需要对应 Trace 数据。

> [!note]- 展开：原理、操作、截图与参考
> **用途**：解释“某一帧为什么慢”“主线程在做什么”“渲染线程是否在等任务”“加载是否造成尖峰”等问题。
>
> 基本操作：
>
> 1. 从编辑器底部 **Trace / Insights** 入口打开 Unreal Insights；也可运行引擎目录下的 `Engine/Binaries/Win64/UnrealInsights.exe`。
> 2. 通过 Trace 控件选择采集目标与所需通道。分析 CPU/GPU 时间时，确保相应数据通道已启用；分析内存或资源加载时，另启用相关通道。
> 3. 开始录制，复现问题，然后停止录制。
> 4. 在 Session Browser 中打开会话，或直接打开 `.utrace` 文件。
> 5. 在 **Timing Insights** 中选择慢帧或一段时间，展开 Game Thread、Render Thread、RHI Thread、Task 和 GPU 轨道，按耗时追踪。
>
> 读图时重点关注：
>
> - **Timing / Timeline**：工作在哪条线程、什么时间执行，是否存在明显等待。
> - **Timers、Callers、Callees**：高耗时事件及其上下层关系。对比单次耗时、调用次数与累计耗时。
> - **Frames**：先选有问题的帧，再检查细节，避免只看平均值。
> - **Asset Loading / Memory Insights**：分别针对加载与内存问题，不应仅凭 CPU 时间线推断全部内存情况。
>
> 如果某条轨道为空，先确认平台、构建方式及 Trace 通道支持情况。没有采集到数据，不等于这一系统没有开销。
>
> 参考：
>
> - [Epic｜Unreal Insights](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-insights-in-unreal-engine?application_version=5.5)
> - [Epic｜Timing Insights](https://dev.epicgames.com/documentation/unreal-engine/timing-insights-in-unreal-engine?application_version=5.5)
> - [Epic｜Trace Quick Start Guide](https://dev.epicgames.com/documentation/unreal-engine/trace-quick-start-guide-in-unreal-engine?application_version=5.5)

### 2.7 ProfileGPU / GPU Visualizer：查单帧渲染阶段

固定条件后输入 `profilegpu`，展开最贵的渲染阶段；父子事件和异步重叠时间不能重复相加。

> [!note]- 展开：原理、操作、截图与参考
> 在目标视图中输入：
>
> ```text
> profilegpu
> ```
>
> 结果通常显示在 GPU Visualizer 窗口或日志中；菜单、快捷键及输出形式可能随编辑器版本和配置变化。
>
> 使用方法：
>
> 1. 固定相机、场景、分辨率和画质，尽量预热 Shader、纹理及相关缓存。
> 2. 捕获代表性的一帧，查看总 GPU 时间和主要 Pass。
> 3. 展开 Base Pass、Shadow / VSM、Lumen、Translucency、Post Processing、TSR 等实际存在的分组。
> 4. 根据最贵阶段提出假设，每次只改一个变量，再对比相同条件下的结果。
>
> 例如：Base Pass 贵就查材质、覆盖率与绘制；阴影贵就查投影光源、投影物体和阴影更新；TSR 贵就结合输入/输出分辨率及抗锯齿质量观察。
>
> 单帧结果可能受首次加载、缓存失效或编辑器开销影响，最好采几次代表性样本。父事件通常包含子事件；异步计算又可能与图形队列重叠，不能把事件树里的所有数字简单相加。
>
> 参考：[Epic｜Graphics Programming Overview（包含 profilegpu 用法）](https://dev.epicgames.com/documentation/unreal-engine/graphics-programming-overview-for-unreal-engine)。

### 2.8 建议的排查顺序

先固定复现条件、区分 CPU/GPU，再选择工具和验证假设；最终以目标设备的帧时间为准。

> [!note]- 展开：原理、操作、截图与参考
> 1. **复现与固定条件**：同一场景、相机、分辨率、画质与帧率限制，记录目标设备。
> 2. **先看 `stat unit`**：分清游戏线程、渲染提交、GPU 或同步等待方向。
> 3. **CPU / 卡顿问题用 Insights**；**GPU 阶段问题用 ProfileGPU**。
> 4. **需要具体 Draw、材质输入或像素结果时再用 RenderDoc**。
> 5. 修改后用同一段操作对比帧时间与画面质量，最终以目标硬件上的实际运行结果为准。
>
> 尽量在独立运行或适合分析的 Development 构建中复测，避免把编辑器窗口与工具开销全归到游戏本身。

---

## 三、几何复杂度与批处理

### 3.1 Nanite：几何复杂度优化

Nanite 主要解决复杂几何的细节管理与细粒度剔除，不能直接等同于实例化或 SRP Batcher。

> [!note]- 展开：原理、操作、截图与参考
> Nanite 是 UE 的虚拟几何系统，重点在高复杂度网格的细粒度剔除和细节管理。它与 ISM / HISM 的重复实例组织、Unity SRP Batcher 的 CPU 提交优化属于不同层次，可以在符合条件的资产上配合使用。
>
> #### 技术选用参考
>
> - 适合需要表现大量几何细节的资产与场景，可减少传统手工 LOD 工作；是否仍需回退网格与其他 LOD 资产，取决于目标平台和路径。
> - 支持条件、显存和流送成本应按 UE 版本与硬件评估。原笔记强调复杂场景中的 GPU、VRAM 和 SSD 预算。
> - 不能因为启用了 Nanite 就忽略材质、阴影、透明及整体帧时间。
>
> #### 使用方法
>
> 选中支持的静态网格，在对应菜单或资产设置中启用 Nanite，再在网格查看界面观察结果。
>
> **启用 Nanite 截图 07**
>
> ![[TA/渲染/附件/UE各种技术模块/07.png|900]]
>
> **Nanite 模型结果截图 08**
>
> ![[TA/渲染/附件/UE各种技术模块/08.png|900]]

### 3.2 UE 与 Unity：批处理机制的区别

**合并几何、共享实例和减少 CPU 状态准备是不同优化。** UE Material Instance 也不等于自动实例化绘制。

> [!note]- 展开：原理、操作、截图与参考
> “合批”包含几种不同的优化：**合并几何、复用实例、减少渲染状态设置，以及减少场景对象管理开销**。比较两款引擎时，要先确认自己想减少的是哪一种成本。以下 Unity 对照以 **2022.3 / 常见 MeshRenderer 路径** 为例，UE 以 **5.5 官方 ISM 文档** 为主要依据。
>
> - **Unity Static Batching**：把满足条件的静态物体顶点变换到世界空间，建立合并后的顶点/索引缓冲。原始 Mesh 可以不同，但需要满足材质、顶点布局等合批条件。它会增加几何内存；Unity 内建静态合批仍可对原物体分别剔除。
> - **UE Merge Actors 的 Merge / Simplify**：生成新的合并网格或简化代理，更接近 Unity 中手动合并 Mesh、制作远景代理的工作流。它改变资产与对象组织，不能直接等同于 Unity 勾选 `Batching Static` 后的行为。
> - **Unity GPU Instancing ↔ UE ISM / HISM**：都适合大量重复网格，通过共享 Mesh / 材质和提供实例数据来减少重复提交。UE 的 ISM / HISM 还把许多实例归到同一个组件中，可减少大量独立 Actor / Component 的管理开销。
> - **Unity SRP Batcher**：重点优化相同 Shader Variant 的一串绘制所需的 CPU 材质/状态设置，并不要求所有物体使用同一个 Mesh，也不意味着把它们变成一次实例化 Draw。其收益应看 CPU 渲染耗时，不能只看 Draw Call 数是否下降。
> - **UE Dynamic Instancing**：符合条件的静态网格绘制可由引擎自动合并为实例化绘制，需要匹配的 Mesh、材质和实际 Shader 绑定等条件。它不等同于 Unity 的 Dynamic Batching——后者会在 CPU 侧处理符合条件的小网格顶点。
>
> UE 的 **Material Instance** 也不等于“自动实例化渲染”。两个物体使用同一父材质，不代表纹理、参数和 Shader 绑定相同，更不能据此保证合并绘制。实例间只需变颜色、随机值等数据时，可评估 Per Instance Custom Data，减少为每个实例单独创建材质的需要。
>
> 实际选择可按场景区分：重复的树木、石头、建筑模块优先评估实例化；不同形状但基本不再单独编辑的远景结构可考虑合并/代理。剔除粒度、LOD、材质成本和几何内存都应一起看，最终比较帧时间。
>
> 官方参考：
>
> - [Unity｜Draw Call Batching](https://docs.unity3d.com/2022.3/Documentation/Manual/DrawCallBatching.html)
> - [Unity｜Static Batching](https://docs.unity3d.com/2022.3/Documentation/Manual/static-batching.html)
> - [Unity｜GPU Instancing](https://docs.unity3d.com/2022.3/Documentation/Manual/GPUInstancing.html)
> - [Unity｜SRP Batcher](https://docs.unity3d.com/2022.3/Documentation/Manual/SRPBatcher.html)
> - [Epic｜Mesh Drawing Pipeline](https://dev.epicgames.com/documentation/en-us/unreal-engine/mesh-drawing-pipeline-in-unreal-engine?application_version=5.5)
>
> 原文补充资料：[合批性能对比数据：rdBPtools Benchmarks](https://recourse.nz/index.php/rdbptools-benchmarks/)。
>
> **合批参考截图 17**
>
> ![[TA/渲染/附件/UE各种技术模块/17.png|900]]

### 3.3 合并 Actor

Merge / Simplify 生成合并网格或代理；Batch 将重复网格组织为实例组件，选项不能混用。

> [!note]- 展开：原理、操作、截图与参考
> 菜单路径：**Actor → Merge Actors → Merge Actors Settings**。
>
> - **Merge**：把多个对象的几何合成新的静态网格，可用于固定组合或远景整理。
> - **Simplify**：生成简化代理，材质合并与烘焙结果取决于所选设置。
> - **Batch**：将重复网格分组为 ISM 组件，属于实例化工作流；它与直接把几何拼成一个新 Mesh 不同。

### 3.4 Instanced Static Mesh（ISM）

**同一 Mesh 的多个实例共享一个 ISM 组件。** UE 5.4 / 5.5 的 ISM 已支持逐实例 LOD 与剔除，但不保证整个组件只有一个 Draw。

> [!note]- 展开：原理、操作、截图与参考
> **ISM 是一个管理同一静态网格多个实例的组件。** 比如同一块石头摆放 500 次，实例共享网格和组件级材质等属性，每个实例保留各自的位置、旋转、缩放，并可用 Per Instance Custom Data 表达颜色或随机变化。
>
> 它更接近 **Unity GPU Instancing**。实例化可减少重复绘制提交，组件化组织还可减少独立 UObject 的数量与相关内存开销。
>
> - **不是整个组件永远只有一次 Draw Call**：材质槽/Section、LOD 和深度、阴影、主绘制等不同 Pass，都可能形成不同批次。
> - **支持实例剔除**：官方文档描述 ISM 在 GPU 上对实例做剔除和 LOD 处理，不能概括为“被遮挡的实例一定全部绘制”。
> - **UE 5.4 起支持逐实例 LOD**：远近不同的实例可选择不同细节级别。早期版本的限制不应直接套到 UE 5.4 / 5.5。
> - **共享属性有约束**：材质、碰撞、阴影等许多属性在组件级配置，不能把每个实例都当成完全独立的 Static Mesh Component。
>
> 适用于重复的树木、石头、围栏、建筑模块等。创建方式包括给蓝图添加 ISM 组件、Merge Actors 的 Batch、建模模式下的 Harvest Instances 等。
>
> **官方说明：[Epic｜Instanced Static Mesh Component（UE 5.5，含 ISM / HISM、LOD、剔除和编辑方式）](https://dev.epicgames.com/documentation/en-us/unreal-engine/instanced-static-mesh-component-in-unreal-engine?application_version=5.5)。**

### 3.5 Hierarchical Instanced Static Mesh（HISM）

HISM 用空间层次加速实例处理，适合大量基本不移动的实例；层次维护和 LOD 分组也有成本，完全使用 Nanite 时优先评估 ISM。

> [!note]- 展开：原理、操作、截图与参考
> HISM 在实例上建立静态空间层次，用于加速剔除和 LOD 处理。**大量、基本不移动的实例**可以评估 HISM；频繁改变实例布局时，需要考虑层次维护成本。
>
> 选择时注意：
>
> - 现代 ISM 也有 LOD 和剔除能力，不能再简单按“ISM 没有、HISM 才有”区分。
> - HISM 的分组处理与独立 Static Mesh 的细粒度 LOD 行为可能不同，需观察切换效果。
> - 官方对完全使用 Nanite 的项目建议优先 ISM，因为 Nanite 有自己的剔除和 LOD 系统；存在非 Nanite 回退网格时再结合情况评估 HISM。
> - 是否更快，仍要在目标平台测试，不能只凭实例数量下结论。
>
> 编辑步骤：切换到建模模式，使用 XForm 分类中的 **ISM Editor** 选择、移动或编辑实例。
>
> **建模模式截图 18**
>
> ![[TA/渲染/附件/UE各种技术模块/18.png|900]]
>
> **ISM 编辑器截图 19**
>
> ![[TA/渲染/附件/UE各种技术模块/19.png|900]]

---

## 四、可见性与空间加速

> [!info]- 展开：本章参考与原始资料
> 参考：
>
> - [Visibility and Occlusion Culling 官方文档](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/visibility-and-occlusion-culling-in-unreal-engine)：包含调试参数。
> - [原文推荐文章](https://zhuanlan.zhihu.com/p/526622741)

### 4.1 PrePass 与遮挡剔除

PrePass 生成深度；遮挡剔除判断对象是否需要绘制。HZB 使用深度建立层次数据，但不应把它理解成与 PrePass 同一种剔除。

> [!note]- 展开：原理、操作、截图与参考
> 原文区分：
>
> - 遮挡剔除：对象级别。
> - PrePass：从像素深度角度理解。
> - HZB 依赖深度图，因此原文把 HZB 放在 PrePass 之后讨论。
> - “OC 和 HZB 只能二选一”为原文转述，应结合具体渲染路径确认。
>
> **PrePass 相关截图 20**
>
> ![[TA/渲染/附件/UE各种技术模块/20.png|900]]
>
> **深度与剔除截图 21**
>
> ![[TA/渲染/附件/UE各种技术模块/21.png|900]]
>
> **剔除流程截图 22**
>
> ![[TA/渲染/附件/UE各种技术模块/22.png|900]]

### 4.2 Occlusion Culling 与 HZB

Occlusion Query 与 HZB 都可辅助对象遮挡判断，实际路径、开关和延迟要结合版本验证。

> [!note]- 展开：原理、操作、截图与参考
> 原文理解：HZB 利用像素深度信息进行对象剔除；原文标题中对是否基于显卡保留了疑问。
>
> 参考：[UE4/5 遮挡剔除（Occlusion Culling）浅析](https://zhuanlan.zhihu.com/p/565197985)。
>
> 调试参数：
>
> ```text
> r.AllowOcclusionQueries
> r.HZBOcclusion
> r.VisualizeOccludedPrimitives
> stat scenerendering
> ```
>
> - `r.AllowOcclusionQueries`：控制是否开启 Occlusion Queries。
> - `r.HZBOcclusion`：选择 HZB 遮挡剔除相关行为。
> - `r.VisualizeOccludedPrimitives`：用于可视化被剔除物体对应的 Bounds。原文认为物体较少时容易观察，物体多时不易分辨；其括注“应该只是视锥剔除”仍属待确认内容。
> - `stat scenerendering`：观察 Draw Calls 和剔除相关耗时。
>
> **遮挡剔除调试截图 23**
>
> ![[TA/渲染/附件/UE各种技术模块/23.png|900]]

### 4.3 BVH 与四叉树

**BVH 是包围体层次；四叉树是按空间划分的结构，两者不能直接混称。** 动态物体的维护成本取决于具体构建与更新方式。

> [!note]- 展开：原理、操作、截图与参考
> 原文记录：“BVH 构建四叉树的时候最费”，并据此关注动态物体的处理成本。
>
> > 待确认：原文将 BVH 与四叉树混在同一句中，未展开具体结构和构建方式。
>
> **BVH 与四叉树参考截图 24**
>
> ![[TA/渲染/附件/UE各种技术模块/24.png|900]]

### 4.4 PVS

PVS 预计算可能可见的对象集合，再按观察位置查询；适用性由场景变化、内存和预计算条件决定，不能按“3A 是否使用”概括。

> [!note]- 展开：原理、操作、截图与参考
> 原文理解：按空间划分进行剔除。
>
> **PVS 截图 25**
>
> ![[TA/渲染/附件/UE各种技术模块/25.png|900]]
>
> 原文经验判断：在其关注的 3A 场景中较少采用这种做法，但空间划分的思想仍可用于资源加载等系统。
>
> **空间划分参考截图 26**
>
> ![[TA/渲染/附件/UE各种技术模块/26.png|900]]

---

## 五、光照、阴影与距离场

### 5.1 Lumen

Lumen 提供动态间接光与反射；先检查追踪路径和平台支持，再调整 Lumen Scene 与后处理质量。

> [!note]- 展开：原理、操作、截图与参考
> **Lumen 原文截图 01**
>
> ![[TA/渲染/附件/UE各种技术模块/01.png|900]]
>
> #### 4.1.1 Lumen 场景细节
>
> Lumen Scene Detail 影响 Lumen 场景表示中的小物体与细节，可通过表面缓存视图观察；它不是主相机中模型是否被绘制的通用开关。
>
> **Lumen 场景细节截图 02**
>
> ![[TA/渲染/附件/UE各种技术模块/02.png|900]]
>
> #### 4.1.2 距离相关
>
> 原文记录：`1000 = 1m`。
>
> > 待确认：原文没有注明对应参数及其单位，此处保留记录，不作为通用单位换算结论。
>
> #### 4.1.3 最终采集质量
>
> Final Gather Quality 控制最终采集的质量取舍；提高质量可改善噪声，也可能增加 GPU 开销。具体数值结合当前版本与场景测试。
>
> #### 4.1.4 设置方法
>
> Lumen 有软件与硬件追踪路径，支持条件应按 UE 版本、目标 RHI 与平台核对。Project Settings 启用功能，后处理体积（PPV）可覆盖对应的质量参数；不能把原文的 DX12 经验概括到所有路径。
>
> 1. 打开项目设置，搜索 `lumen`。
> 2. 对截图中带箭头的选项酌情调整。
>
> **Lumen 项目设置截图 03**
>
> ![[TA/渲染/附件/UE各种技术模块/03.png|900]]
>
> 3. 在项目设置中搜索 `ray`。
>
> **光追项目设置截图 04**
>
> ![[TA/渲染/附件/UE各种技术模块/04.png|900]]
>
> 4. 选中平行光，搜索 `ray`。
>
> **平行光设置截图 05**
>
> ![[TA/渲染/附件/UE各种技术模块/05.png|900]]
>
> **光照相关设置截图 06**
>
> ![[TA/渲染/附件/UE各种技术模块/06.png|900]]
>
> 5. 在后处理体积中调整全局光照。

### 5.2 MegaLights

MegaLights 是需要单独核对版本、平台和配置的多光源方案；原笔记只记录了硬件光追的准备步骤。

> [!note]- 展开：原理、操作、截图与参考
> 原文配置记录：先开启硬件光追。
>
> ```text
> Engine → Rendering → Hardware Ray Tracing
> ```

### 5.3 距离场（SDF）

Mesh SDF 描述单个网格的距离场，Global SDF 组织场景距离信息；是否生成要看软件追踪、距离场阴影、DFAO 等实际需求。

> [!note]- 展开：原理、操作、截图与参考
> 参考：[Mesh Distance Fields 官方文档](https://dev.epicgames.com/documentation/en-us/unreal-engine/mesh-distance-fields-in-unreal-engine)。
>
> 原文区分：
>
> - **Mesh SDF**：单个网格体的距离场。
> - **Global SDF**：原文理解为由 Mesh SDF 组成的全局距离场。
>
> 可通过网格体详细信息界面查看 Mesh SDF 的消耗。
>
> **网格体距离场截图 27**
>
> ![[TA/渲染/附件/UE各种技术模块/27.png|900]]
>
> 原文列出的用途：
>
> - Lumen
> - SDF 软阴影
> - DFAO
> - SDFGI
>
> 原文建议：若不依赖相关功能，可评估关闭 SDF 生成。

### 5.4 阴影

**VSM 管阴影贴图的虚拟页面与缓存；Contact Shadows 补充局部接触细节。** 两者不是同一个开关。

> [!note]- 展开：原理、操作、截图与参考
> #### 4.4.1 Virtual Shadow Maps（VSM）
>
> VSM 将阴影贴图组织为虚拟页面，按需要分配、渲染并复用缓存。动态光源、物体移动和 WPO 等可能使页面失效，需要重新绘制；评估时同时看页面更新与投影物体成本。原笔记未记录具体参数。
>
> #### 4.4.2 Contact Shadow（接触阴影）
>
> 原文记录：用于提供更细致的照明表现，通常考虑在点光源上开启。
>
> 参考：[Contact Shadows 官方文档（UE 5.0）](https://dev.epicgames.com/documentation/en-us/unreal-engine/contact-shadows-in-unreal-engine?application_version=5.0)。

---

## 六、纹理流送与显存

> [!info]- 展开：本章参考与原始资料
> 参考：[2020 Virtual Texture 的理解和应用｜Epic 李文磊](https://www.bilibili.com/video/BV1KK411L7Rg/)。
>
> 原文记录：对贴图集合尤其有用；植被（Foliage）采用贴图合并方式合批后，也适合关注虚拟纹理方案。

### 6.1 Texture Streaming

传统 Texture Streaming 根据需求选择并驻留 Mip；只看到纹理一小部分，也可能需要该 Mip 的整张纹理。

> [!note]- 展开：原理、操作、截图与参考
> 主要决定贴图使用哪一级 Mip。
>
> 原文对传统纹理流送的理解：CPU 根据对象可见性和剔除信息进行 Mip 选择，决策较保守；即使只看到纹理的一部分，也可能需要加载对应 Mip 的整张纹理。
>
> **传统纹理流送截图 09**
>
> ![[TA/渲染/附件/UE各种技术模块/09.png|900]]

### 6.2 Streaming Virtual Texturing（SVT）

SVT 将纹理分成瓦片，按需求流送并通过页表定位；同时关注物理池、反馈、页面更新与采样开销。

> [!note]- 展开：原理、操作、截图与参考
> 应用目标：在使用大尺寸、高分辨率纹理时控制显存占用。
>
> 参考：[Streaming Virtual Texturing 官方文档（UE 5.3）](https://dev.epicgames.com/documentation/en-us/unreal-engine/streaming-virtual-texturing-in-unreal-engine?application_version=5.3)。
>
> 原文要点：
>
> - 加载高分辨率纹理的高精度 Mip，可能产生显著的性能和内存开销。
> - 虚拟纹理把各级 Mip 拆分为固定大小的瓦片（Tile），按需流送可见部分。
> - 原文将其理解为通过 Page Table 查找需要的页面，主要关注 Physical Texture 部分的占用。
>
> 调试命令：
>
> ```text
> r.VT.Borders 1
> r.VT.Flush
> stat virtualtexturing
> stat virtualtexturememory
> ```
>
> **SVT 原理截图 10**
>
> ![[TA/渲染/附件/UE各种技术模块/10.png|900]]
>
> **Page Table 与 Physical Texture 截图 11**
>
> ![[TA/渲染/附件/UE各种技术模块/11.png|900]]
>
> 双击纹理，可在对应界面观察采样的 Mip 部分。
>
> **纹理 Mip 查看截图 12**
>
> ![[TA/渲染/附件/UE各种技术模块/12.png|900]]

### 6.3 Runtime Virtual Texturing（RVT）

RVT 侧重运行时材质缓存、地形混合和表面融合；与普通 Render Texture 的类比只用于理解。

> [!note]- 展开：原理、操作、截图与参考
> 原文列出的应用场景：
>
> - 复杂地形材质造成性能压力。
> - 让模型边缘自然融入地面。
> - 雪地痕迹、地形混合等效果。
>
> 原文以 Unity 的运行时 Render Texture（RT）作类比，强调其材质缓存用途；这是帮助理解的类比，不是两种机制完全相同的定义。

### 6.4 VT 参数

Tile Size、Border 和 Feedback 参数在颗粒度、过滤质量、缓存占用与流送延迟之间取舍；128 / 4 是原笔记的经验起点。

> [!note]- 展开：原理、操作、截图与参考
> 参考：[虚拟纹理设置和属性](https://dev.epicgames.com/documentation/en-us/unreal-engine/virtual-texturing-settings-and-properties-in-unreal-engine)。
>
> - **Tile Size（瓦片大小）**：原文建议从 `128` 开始评估，认为可适配较多情况。
> - **Tile Border Size（瓦片边界大小）**：增大边界可支持更高程度的各向异性过滤，同时增加磁盘与缓存内存占用。原文提到默认值为 `4`。
> - **Feedback Resolution Factor（反馈分辨率系数）**：较小的系数提高反馈分辨率，增加 CPU/GPU 开销；在材质使用较多虚拟纹理时，可能降低流送延迟。
>
> **VT 瓦片设置截图 13**
>
> ![[TA/渲染/附件/UE各种技术模块/13.png|900]]
>
> 网格体绘制参考：[Mesh Texture Color Painting 入门](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/getting-started-with-mesh-texture-color-painting-in-unreal-engine)。
>
> **网格体绘制截图 14**
>
> ![[TA/渲染/附件/UE各种技术模块/14.png|900]]
>
> 原文标注：以下为 SVT 设置。
>
> **SVT 设置截图 15**
>
> ![[TA/渲染/附件/UE各种技术模块/15.png|900]]

### 6.5 虚拟纹理池

物理纹理池的配置容量与实际驻留压力要分开看；容量有限时可能出现细节退化或流送压力。

> [!note]- 展开：原理、操作、截图与参考
> 原文转述的经验：纹理池消耗相对固定，会根据资源压力决定采样的 Mip/LOD 级别。具体池容量、反馈和驻留行为仍需结合实际配置观察。
>
> **虚拟纹理池截图 16**
>
> ![[TA/渲染/附件/UE各种技术模块/16.png|900]]

---

## 七、抗锯齿与时序超分辨率

### 7.1 TAA、TAAU、TSR 的关系

**TAA 侧重时序抗锯齿，TAAU 加入时序上采样，TSR 是 UE5 的时序重建方案。**

> [!note]- 展开：原理、操作、截图与参考
> **UE 的时序抗锯齿要区分 TAA、TAAU 和 TSR。TAAU 与 TSR 都利用历史帧，属于时序方案。** 从技术路线理解，可以把 TSR 看作 UE5 内建方案中比传统 TAA / TAAU 更进一步的时序重建方案。
>
> - **TAA（Temporal Anti-Aliasing）**：通过子像素采样抖动和多帧历史积累抑制锯齿、闪烁，主要解决抗锯齿与稳定性。
> - **TAAU（Temporal Anti-Aliasing Upsampling）**：把时序抗锯齿与上采样结合，在较低内部渲染分辨率下生成更高分辨率输出。UE 官方将这一方法列为 TAAU；Temporal Upsampling 开启时，TAA 路线会与 Screen Percentage 配合工作。
> - **TSR（Temporal Super Resolution）**：UE5 的时序超分辨率方案，更重视低分辨率输入下的细节重建、历史复用、反遮挡处理及稳定性。它会在重建步骤投入更多工作，以换取前面阶段降低渲染分辨率后的整体收益。
>
> **便于记忆：TAA 侧重多帧抗锯齿，TAAU 把它扩展到时序上采样，TSR 则是 UE5 内建的更高阶时序重建方案。** “TAA 最终级”可作为笔记中的通俗类比，官方名称仍是 TSR；选择时要结合质量与性能测试。UE 也允许关闭 Temporal Upsampling，使用普通 TAA 再配合空间上采样。

### 7.2 时序方案为什么能从较低分辨率重建画面

利用采样抖动、运动矢量、深度与历史可信度重建画面；不是把单帧图片简单放大。

> [!note]- 展开：原理、操作、截图与参考
> 可以按以下过程理解：
>
> 1. 每帧对采样位置施加轻微抖动，获得不同的子像素信息。
> 2. 使用运动矢量和深度等输入，将上一帧的历史信息重投影到当前帧。
> 3. 判断哪些历史信息仍可信；遇到新显露区域、遮挡变化、材质变化等情况时，减少或拒绝历史贡献。
> 4. 结合当前帧与可用历史，形成更稳定、更高分辨率的结果。
>
> 它不是单张图片简单放大。较好的重建依赖正确的运动矢量、深度、历史管理及足够的当前帧信息；历史失效、快速运动、透明和细碎几何都会增加难度。

### 7.3 分辨率、画质与性能取舍

降低 Screen Percentage 可减少部分像素成本，但 TSR、阴影、几何和提交仍有开销；同时比较运动画质与整帧耗时。

> [!note]- 展开：原理、操作、截图与参考
> **Screen Percentage 控制的主要是内部渲染分辨率，输出分辨率可以保持不变。** 例如输出 1920×1080，内部按 50% 的宽高渲染时约为 960×540，输入像素数量约为四分之一；TAAU / TSR 再进行时序重建。
>
> 但这不表示整帧开销变为四分之一：几何、提交、阴影以及不随该分辨率缩放的工作仍存在，TSR 自身和部分后续后处理也有成本。
>
> 评估时建议：
>
> - 在项目设置 **Engine → Rendering → Default Settings** 中选择抗锯齿方法；实际名称以 UE 版本为准。
> - 用同一场景、相机运动和输出分辨率，对比不同 Screen Percentage 与 Anti-Aliasing 质量档位。
> - 静止和运动都要看，重点观察头发、植被、细线、远景纹理、透明物体、WPO 动画的拖影、闪烁与锐度。
> - 用 ProfileGPU 观察 TSR 自身耗时和整帧收益，而不是只凭一张静态截图判断质量。
>
> 可用的观察命令：
>
> ```text
> r.ScreenPercentage
> stat gpu
> profilegpu
> ```
>
> 无参数查询当前值；需要修改时，再按项目测试计划传入目标值。编辑器视口覆盖、动态分辨率等也可能影响实际比例。

### 7.4 头发中的 TAA / TSR + Dither

Masked 发片可用 Dither 与时序积累表现部分覆盖；快速运动、细发丝与历史失效时仍需检查闪烁和拖影。

> [!note]- 展开：原理、操作、截图与参考
> 发片常用 **Masked / Alpha Test** 表达覆盖，不走普通半透明的逐对象混合排序。Dither 用随采样变化的遮罩表现部分覆盖，再由时序抗锯齿/重建在多帧中积累出较平滑的结果。
>
> 原笔记参考《虚幻争霸》发片方案。这种方法与项目采用的 TAA / TAAU / TSR 配置有关：启用 TSR 不能直接保证旧发片材质得到相同结果，尤其要检查快速运动、远景发丝、运动矢量及遮罩抖动的稳定性。
>
> **TAA 与 Dither 原文截图 31**
>
> ![[TA/渲染/附件/UE各种技术模块/31.png|900]]
>
> **头发透明表现原文截图 32**
>
> ![[TA/渲染/附件/UE各种技术模块/32.png|900]]

### 7.5 与其他抗锯齿方案的区别

FXAA / MSAA 和 TAAU / TSR 的输入、支持路径与目标不同；第三方时序超分也应分别评估。

> [!note]- 展开：原理、操作、截图与参考
> - **FXAA**：基于当前帧图像做空间抗锯齿，不依赖历史帧；成本较低，但细节和稳定性与时序方案的取舍不同。
> - **MSAA**：重点处理几何边缘的多重采样，在 UE 中受渲染路径支持条件限制；它不等同于 TAAU / TSR 的时序超分。
> - **第三方时序超分**：DLSS、FSR 的时序版本、XeSS 等也利用多帧信息，但算法、硬件要求和插件集成不同，不应与 TSR 混称。
>
> 官方参考：
>
> - [Epic｜Anti-Aliasing and Upscaling（包含 TAAU 与关闭 Temporal Upsampling 的说明）](https://dev.epicgames.com/documentation/en-us/unreal-engine/anti-aliasing-and-upscaling-in-unreal-engine?application_version=5.5)
> - [Epic｜Temporal Super Resolution（原理、性能与调试）](https://dev.epicgames.com/documentation/en-us/unreal-engine/temporal-super-resolution-in-unreal-engine?application_version=5.5)

---

## 八、角色材质与毛发

### 8.1 皮肤

本节保留皮肤着色参考，具体材质参数尚未展开；不要把来源链接当成已验证的项目配置。

> [!note]- 展开：原理、操作、截图与参考
> 原文参考：[皮肤着色相关文章](https://zhuanlan.zhihu.com/p/1930921012570100742)。

### 8.2 头发：标准制作方法

区分发片覆盖、颜色、染色遮罩和切线等数据用途；移动端应另外检查材质与平台方案。

> [!note]- 展开：原理、操作、截图与参考
> 原文提到以下资产和材质信息：
>
> - 头发通常使用多种贴图，原文未完整列出各类贴图。
> - **DyeMask**：染色遮罩，原文标注“有可能会用到 2u”，含义未展开。
> - **MetaHuman**：其材质中有面向移动端的方案。

### 8.3 虚幻争霸头发

原笔记以《虚幻争霸》发片为主要参考，重点记录 ID 贴图和发丝切线扰动。

> [!note]- 展开：原理、操作、截图与参考
> 原文主要参考《虚幻争霸》角色的头发实现。
>
> **虚幻争霸头发截图 28**
>
> ![[TA/渲染/附件/UE各种技术模块/28.png|900]]
>
> 原文重点讨论的贴图：
>
> **头发 ID 贴图截图 29**
>
> ![[TA/渲染/附件/UE各种技术模块/29.png|900]]
>
> 原文记录：虚幻争霸使用 ID 贴图；交流中有人建议通过灰度，在 Houdini 中利用 `step` 生成类似贴图。笔记将其目的归纳为对单根头发的切线进行扰动，也可参考以下方式。
>
> **头发切线扰动截图 30**
>
> ![[TA/渲染/附件/UE各种技术模块/30.png|900]]
>
> 时序抗锯齿与发片 Dither 的原理、注意事项和对应两张原图，统一移至[[#七、抗锯齿与时序超分辨率|第 7 章]]。

### 8.4 Groom

Groom 与发片按目标画质、LOD 和设备成本选择；“偏影视”是原笔记在 2025-04-10 的经验记录。

> [!note]- 展开：原理、操作、截图与参考
> 原文观点的时间点为 **2025-04-10**：作者当时认为 Groom 更偏影视使用，关注其性能成本。具体消耗说明见原图。
>
> **Groom 消耗相关截图 33**
>
> ![[TA/渲染/附件/UE各种技术模块/33.png|900]]
>
> 原文提出两条可参考的思路：
>
> 1. 参考武装猫工作室的 Groom 游戏应用分享。
> 2. 使用 Groom 生成的 LOD Mesh，再重新编写材质；原文认为其面数和 UV 较可靠。
>
> **Groom LOD Mesh 截图 34**
>
> ![[TA/渲染/附件/UE各种技术模块/34.png|900]]
>
> 原文对发片与 Groom 的效果判断：发片也可取得不错的效果，但在其观察的影视级细节目标下，仍与 Groom 有差距。
>
> **发片与 Groom 参考截图 35**
>
> ![[TA/渲染/附件/UE各种技术模块/35.png|900]]
>
> 原文展示的渲染流程：
>
> **Groom 渲染流程截图 36**
>
> ![[TA/渲染/附件/UE各种技术模块/36.png|900]]
>
> 参考资料：
>
> - [UnrealCircle 武汉：Groom 毛发的制作及游戏应用分享｜聂超·武装猫工作室](https://www.bilibili.com/video/BV1Vs421u7Qc/)
> - [Unreal Engine 毛发与皮毛白皮书（中文 PDF）](https://cdn2.unrealengine.com/unreal-engine-hair-and-fur-whitepaper-zhcn-73d68a66a2a5.pdf)

---

## 九、其他原始资料

> [!info]- 展开：本章参考与原始资料
> 原文仅提供以下插图，未附说明。
>
> **其他技术笔记截图 38**
>
> ![[TA/渲染/附件/UE各种技术模块/38.png|900]]

---

## 十、关联笔记与资料

- [[TA/渲染/小嗯的unity渲染设置#三、合批原理与常见断批|Unity 合批设置与条件]]。
- [[TA/动作/小嗯的动作文档#五、项目测试与管理成本|骨骼动画与 BAT 的项目测试]]。
- [石墨原始资料：ue各种技术模块](https://shimo.im/docs/5xkGo4p2vrTEQMkX/)。

整理基于原笔记的 2026-09-22 版本，并于 2026-10-30 从石墨原文补回 38 张插图。官方与其他参考链接保留在各节折叠块内。未注明参数的单位、旧项目经验及跨版本结论保留了上下文，不能直接当作当前项目配置。
