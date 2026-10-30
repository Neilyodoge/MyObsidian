---
title: 小嗯的 Unity 渲染设置
tags:
  - Unity
  - TA
  - 渲染/项目设置
cssclasses:
  - action-note
updated: 2026-10-30
---

适用 **Unity 2022.3**，URP 相关入口以对应包版本为准。先看可操作的关键设置，再展开设置细节、合批条件和 Shader 变体管理。最终以目标设备上的 Frame Debugger / Profiler 结果为准。

快速跳转：[[#一、关键设置与验证|关键设置]] · [[#二、设置位置与平台细节|设置位置与平台]] · [[#三、合批原理与常见断批|合批]] · [[#四、Shader 变体收集、预热与裁剪|变体管理]]

## 一、关键设置与验证

### 1.1 项目设置速查

入口：`Edit > Project Settings`。具体选项会随 Build Target 和已安装包变化。

- **先确认实际生效的渲染管线**：当前 `Quality` 档的 Render Pipeline Asset → `Graphics` 的 Default Render Pipeline → 两处都为空时使用 Built-in。先确认目标平台的默认质量档，再改对应 Asset。[[#2.1 渲染管线与设置位置|查看设置位置与覆盖关系]]
- **Color Space**：`Player > Other Settings > Rendering`，通常采用 **Linear**；切换后重新检查材质、光照与 UI 颜色。[[#2.4 平台设置细节|查看平台设置细节]]
- **Graphics APIs**：在 `Player` 中按目标平台配置；关闭 Auto 后，列表中第一个受支持的 API 优先使用。多线程渲染和 Graphics Jobs 要在目标设备上比较后决定。[[#2.4 平台设置细节|查看平台设置细节]]
- **纹理质量**：在 `Quality` 中设置 Mipmap Limit、各向异性与 Texture Streaming；需要 Streaming 的纹理还要在导入设置中开启 Mip Streaming。[[#2.3 纹理层级与流送|查看纹理层级与流送]]
- **LOD与帧率**：在 `Quality` 中设置 LOD Bias、Maximum LOD Level 和 VSync。Bias 越高越晚降级；VSync 会影响帧率上限与输入延迟。[[#2.2 质量设置细节|查看质量设置细节]]
- **阴影与抗锯齿**：Built-in 看 `Quality`；URP 主要看当前质量档绑定的 URP Asset。URP 的 Render Scale、HDR、MSAA、灯光和阴影也在 Asset 中调整。[[#2.1 渲染管线与设置位置|查看URP设置位置]]
- **Shader包含与预热**：`Graphics` 中的 Always Included Shaders 仅添加确实需要强制包含的 Shader；需要预热时使用 ShaderVariantCollection，并放入 Preloaded Shaders 或在加载流程中调用预热。[[#4.2 变体收集与预热|查看变体收集与预热]]
- **着色器变体裁剪**：根据项目实际使用的功能配置 Graphics / URP Global Settings；发布包通常可剔除调试变体，需要 Player 中的 Rendering Debugger 时保留。动态加载和运行时切换的变体要单独检查。[[#4.3 变体裁剪|查看变体裁剪]]

### 1.2 合批设置速查

- **固定不动的物体**：开启 Static Batching，并为对象勾选 `Batching Static`；适用于 Mesh 可以不同、材质与渲染状态兼容的静态物体。[[#3.1 合批方式与兼容关系|查看Static Batching条件]]
- **大量重复的同Mesh、同材质物体**：Shader 与平台支持时，在材质上开启 `Enable GPU Instancing`。GPU Instancing 没有 Player 全局开关。[[#3.1 合批方式与兼容关系|查看GPU Instancing条件]]
- **URP/HDRP的普通Renderer**：确认 SRP Batcher 已启用且 Shader 兼容；普通 Renderer 同时兼容时，通常优先使用 SRP Batcher。[[#3.1 合批方式与兼容关系|查看SRP Batcher与变体]]
- **同一Renderer的路径选择**：Static Batching 可与 SRP Batcher 共用；Static Batching 与 GPU Instancing 互斥，同时满足时 Unity 优先 Static Batching。[[#3.1 合批方式与兼容关系|查看组合原理]]
- **Dynamic Batching的位置**：Built-in 看 Player；URP 看 URP Asset。URP 中兼容对象优先走 SRP Batcher，Dynamic Batching 只补剩余对象。[[#2.1 渲染管线与设置位置|查看设置位置]]
- **遇到断批先检查**：Shader 变体、材质、MPB、`Renderer.material` 与 Lightmap。[[#3.2 合批常见坑|查看常见断批原因]]

### 1.3 配置后的检查

1. 确认设备实际使用的质量档位、渲染管线、URP Asset 和图形 API。
2. 用 Frame Debugger 检查绘制路径和断批原因。
3. 用 Profiler 比较 CPU、GPU 和内存；固定场景，一次只改一项。
4. 用接近正式发布的构建复测，尤其检查动态加载资源与运行时关键词切换。

---

## 二、设置位置与平台细节

### 2.1 渲染管线与设置位置

先确认当前 Quality 档使用的 Pipeline Asset；URP Asset、Renderer Data、Lighting 和 Volume 各管不同设置。

> [!note]- 展开：设置项、条件与操作细节
> **Render Pipeline Asset 优先级：**当前 Quality 档绑定的 Asset → Graphics 的 Default Render Pipeline → 两处都为空时使用 Built-in。
>
> #### Graphics
>
> 官方说明：[Graphics Settings](https://docs.unity3d.com/2022.3/Documentation/Manual/class-GraphicsSettings.html)。
>
> - **Default Render Pipeline**：指定项目默认的 URP/HDRP Asset；**注意**：`Quality` 某档绑定了 Asset 时，会覆盖这里的默认值
> - **Camera Settings**：设置透明物体的排序模式和排序轴；**注意**：2D、等距视角常用；普通 3D 通常保持默认
> - **Tier Settings**：设置 Built-in 各硬件等级的渲染能力；**注意**：主要用于 Built-in，URP/HDRP 不要只看这里
> - **Shader Settings / Stripping**：强制包含 Shader、预热 ShaderVariantCollection，或裁剪 Built-in 变体；**注意**：[[#4.2 变体收集与预热|变体收集与预热]]、[[#4.3 变体裁剪|变体裁剪]]
> - **[URP Global Settings](https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@14.0/manual/urp-global-settings.html)**：Rendering Layers 名称、Shader Variant 日志与剔除；**注意**：URP 安装后显示在 `Graphics` 下；开启 Strip Debug 后，Player 中不能使用 Rendering Debugger
>
> #### 容易找错的位置
>
> - **[URP Asset](https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@14.0/manual/universalrp-asset.html)**：Project 资源的 Inspector；**主要内容**：Render Scale、HDR、MSAA、灯光、阴影、后处理；SRP Batcher 与 Dynamic Batching 同时开启时，兼容对象优先走 SRP Batcher，Dynamic Batching 只补剩余对象
> - **[Universal Renderer Data](https://docs.unity3d.com/Packages/com.unity.render-pipelines.universal@14.0/manual/urp-universal-renderer.html)**：Project 资源的 Inspector；**主要内容**：Rendering Path、Depth Priming、Intermediate Texture、Renderer Features
> - **Lighting**：`Window > Rendering > Lighting`；**主要内容**：环境光、GI、Lightmap、反射探针
> - **Camera / Volume**：场景组件的 Inspector；**主要内容**：相机覆盖项和后处理参数
>
> [[#一、关键设置与验证|返回关键设置]]

### 2.2 质量设置细节

Quality 档决定平台的质量配置与 Pipeline Asset；纹理、LOD、帧率和上传预算需要一起检查。

> [!note]- 展开：设置项、条件与操作细节
> 官方说明：[Quality Settings](https://docs.unity3d.com/2022.3/Documentation/Manual/class-QualitySettings.html)。
>
> - **Quality Levels**：设置各平台默认质量档；**注意**：先确认设备实际进入了哪一档
> - **Render Pipeline Asset**：为每个质量档绑定不同的 URP/HDRP Asset；**注意**：**优先级高于 Graphics 中的默认 Asset**
> - **Textures**：Mipmap Limit、分组、各向异性、[Texture Streaming](https://docs.unity3d.com/2022.3/Documentation/Manual/TextureStreaming.html)；**注意**：Limit 管最高细节；Streaming 管当前实际驻留层级，见[[#2.3 纹理层级与流送|详细说明]]
> - **LOD**：LOD Bias、Maximum LOD Level；**注意**：Bias 越高越晚降级；Maximum LOD 还可能影响构建时的 LOD 裁剪
> - **VSync**：与屏幕刷新率同步；**注意**：会影响帧率上限和输入延迟
> - **Shadows / Anti Aliasing**：Built-in 的阴影和抗锯齿；**注意**：URP 下主要看该档绑定的 URP Asset
> - **Async Upload**：控制纹理、Mesh 上传时间片和缓冲区；**注意**：缓冲过小易卡顿，过大会增加内存
>
> [[#一、关键设置与验证|返回关键设置]]

### 2.3 纹理层级与流送

**Mipmap Limit 限制允许的最高细节，Texture Streaming 决定当前驻留层级。** 还受导入 Max Size 与可用 Mip 限制。

> [!note]- 展开：设置项、条件与操作细节
> Mipmap Limit 限制允许使用的最高细节；Texture Streaming 决定当前实际驻留的 Mip 层级，两者需要一起检查。
>
> - **Limit 0 / 1 / 2 / 3**：宽、高依次为原图的 1、1/2、1/4、1/8
> - **Use Global**：跟随全局限制
> - **Offset -1 / +1**：比全局清晰一级 / 模糊一级
> - **Override**：直接覆盖该组的限制
>
> > 仅影响有 Mipmap 且支持该功能的纹理；不能突破导入 Max Size 或已有 Mip。Texture Streaming 还需在纹理导入设置中开启 Mip Streaming。Unity 2022.3 对 Texture2DArray 的说明有冲突，按目标平台实测。
>
> [[#一、关键设置与验证|返回关键设置]]

### 2.4 平台设置细节

Player 的图形 API、Color Space、线程与数据编码按目标平台配置；开关有支持条件，最终在设备上验证。

> [!note]- 展开：设置项、条件与操作细节
> 官方说明：[Player Settings](https://docs.unity3d.com/2022.3/Documentation/Manual/class-PlayerSettings.html)。
>
> - **Color Space**：选择 Gamma 或 Linear 工作流；**注意**：Linear 通常更符合物理光照，切换后需重查材质和 UI 颜色
> - **Graphics APIs**：设置图形 API 及优先顺序；**注意**：关闭 Auto 后，第一个受支持的 API 会优先使用
> - **Multithreaded Rendering / Graphics Jobs**：将部分渲染提交移出主线程；**注意**：支持情况因平台和 API 而异，必须实机验证
> - **Static / Dynamic Batching**：控制内置合批开关；**注意**：URP 的 Dynamic Batching 在 URP Asset；GPU Instancing 没有 Player 全局开关
> - **GPU Skinning**：将蒙皮计算交给 GPU；**注意**：仅在对应平台支持时出现，需比较 CPU、GPU 和内存
> - **Lightmap / HDR Cubemap Encoding**：控制烘焙数据编码质量；**注意**：按平台生效，修改后重新构建验证
> - **Mipmap Stripping / Virtual Texturing**：剔除不用的 Mip，或启用虚拟纹理；**注意**：仅部分平台显示；错误配置可能造成远近景模糊或纹理缺级
> - **Resolution and Presentation**：分辨率、方向、全屏和显示缓冲；**注意**：移动端与桌面端字段不同
>
> [[#一、关键设置与验证|返回关键设置]]

---

## 三、合批原理与常见断批

### 3.1 合批方式与兼容关系

**Static Batching 合并静态几何，GPU Instancing 共享重复实例，SRP Batcher 降低 CPU 状态准备。** 三者的条件与收益不同。

> [!note]- 展开：设置项、条件与操作细节
> - **Static Batching**：固定不动的物体；Mesh 可不同；**关键条件**：材质、Pass、渲染状态和顶点布局兼容；**主要限制**：子物体不能单独改 Transform；增加 Mesh 内存；单个生成批次最多 64k 顶点
> - **GPU Instancing**：大量重复的同 Mesh、同材质物体；**关键条件**：同 Mesh、同材质、同 Pass；Shader、材质和平台支持实例化；**主要限制**：普通 `SkinnedMeshRenderer` 不走自动实例化；没有固定的“划算数量”
> - **SRP Batcher**：URP/HDRP；材质或 Mesh 不同，但 Shader 变体相同；**关键条件**：连续使用同一 Shader 变体；Shader 常量布局兼容；**主要限制**：主要降低 CPU 提交成本，不等于减少 Draw Call；Built-in 不支持
>
> - **Static Batching + SRP Batcher**：**可共用**；**原因**：Static Batching 将静态几何预转换并放入共享批次；SRP Batcher 缓存 Shader/材质常量，降低 CPU 提交成本。两者优化环节不同
> - **Static Batching + GPU Instancing**：**同一 Renderer 互斥**；**原因**：GPU Instancing 需要保留多个“同 Mesh、同材质”的独立实例，并提交各自的 Transform/参数；静态合批后会改走静态批次路径，不再作为原 Mesh 的独立实例绘制
>
> 项目中可以同时开启 Static Batching 和 GPU Instancing，让不同 Renderer 各走合适的路径；同一 Renderer 同时满足时，Unity 优先使用 Static Batching。
>
> SRP Batcher 所说的“同一 Shader 变体”，见[[#4.1 着色器变体|着色器变体]]。
>
> [[#1.2 合批设置速查|返回合批设置速查]]

### 3.2 合批常见坑

断批先看 Shader 变体、材质实例、MPB、Lightmap 与实际 API；不要只凭材质名称判断。

> [!note]- 展开：设置项、条件与操作细节
> - **Static Batching**：构建时要开启该功能并勾选 `Batching Static`；运行时 `StaticBatchingUtility.Combine` 要求 Mesh 开启 Read/Write
> - **Transform**：静态批次根节点可整体移动，子物体不能单独移动
> - **MPB**：会破坏 SRP Batcher；若只设置 Shader 已声明的实例化属性，GPU Instancing 仍可能生效
> - **`Renderer.material`**：会生成独立材质，可能破坏 GPU Instancing；修改 `sharedMaterial` 会影响所有共享对象
> - **烘焙光照**：同一实例批次需使用同一张 Lightmap，可使用图集中的不同区域
> - **URP/HDRP**：普通 Renderer 同时兼容时，通常优先 SRP Batcher，而非自动 GPU Instancing
> - **`RenderMeshInstanced`**：仅该 API 单次最多 1023 个实例；默认双矩阵时通常最多 511 个，实际还受平台和实例数据影响；按整批 Bounds 剔除
>
> [[#1.2 合批设置速查|返回合批设置速查]]

---

## 四、Shader 变体收集、预热与裁剪

### 4.1 着色器变体

变体是同一 Shader 在不同关键词组合下生成的程序版本；同名 Shader 不代表同一变体。

> [!note]- 展开：设置项、条件与操作细节
> Shader 变体是同一个 Shader 在不同关键词组合下生成的程序版本。法线贴图、阴影、雾、实例化等功能开关都可能引入不同变体；同一个 Shader 名称并不代表使用了同一变体。
>
> - **关键词组合会扩大数量**：例如三个独立的二选一关键词组，理论上形成 8 种组合；实际还会受 Pass、图形 API、平台与裁剪规则影响。
> - **`shader_feature`**：通常用于由材质决定的功能，未被构建内容引用的组合可能被裁剪。运行时才开启的组合需要明确保留。
> - **`multi_compile`**：通常用于需要保留多种组合的功能；仍可能受引擎、渲染管线和自定义裁剪规则影响。
> - **与SRP Batcher的关系**：连续绘制使用同一变体有利于降低 CPU 提交成本；关键词切换可能导致切换批次。具体用 Frame Debugger 检查。
>
> 管理变体时，要分别处理[[#4.2 变体收集与预热|收集与预热]]和[[#4.3 变体裁剪|裁剪]]。
>
> [[#一、关键设置与验证|返回关键设置]]

### 4.2 变体收集与预热

**收集记录需要哪些组合，预热提前准备已保留的组合。** 一次游玩收集不保证覆盖动态加载与运行时开关。

> [!note]- 展开：设置项、条件与操作细节
> **收集**用于记录需要的 Shader、Pass 类型与关键词组合；**预热**用于提前准备已保留的变体，减少首次使用时的卡顿。ShaderVariantCollection（SVC）用于保存这些组合。
>
> #### 收集流程
>
> 1. 在编辑器的 `Graphics` 页面找到 Shader 变体追踪区域，清空已追踪记录。
> 2. 运行代表性场景，覆盖材质、灯光、阴影、后处理、实例化、质量档切换与动态加载内容。
> 3. 将已追踪的组合保存为 ShaderVariantCollection；根据内容模块拆分集合，便于加载管理。
> 4. 检查运行时才会开启的关键词与动态加载资源，补充未被这次运行覆盖的组合。一次游玩收集不能保证覆盖全部需求。
>
> #### 预热方式
>
> - **启动时预热**：把集合加入 `Graphics > Preloaded Shaders`。适合启动阶段需要的变体；集合过大会增加启动时间与内存占用。
> - **按内容预热**：在场景或模块的加载阶段加载集合，再调用 `ShaderVariantCollection.WarmUp()`。避免在正常游玩时集中预热大量变体。
> - **实机验证**：预热效果受图形 API 与驱动影响，仍需用目标设备检查首次渲染卡顿。
>
> > **Always Included Shaders 会带入该 Shader 的全部变体**，过多会增加包体、启动时间和内存。SVC 能声明所需组合并参与变体保留，但仍要核对管线与自定义裁剪规则；预热不能恢复构建时已经被裁掉的变体。
>
> [[#一、关键设置与验证|返回关键设置]] · [[#4.3 变体裁剪|查看变体裁剪]]

### 4.3 变体裁剪

裁剪减少构建中不需要的组合；运行时需求必须保留，**预热无法恢复已经被裁掉的变体。**

> [!note]- 展开：设置项、条件与操作细节
> 变体裁剪是在构建时去掉不需要的组合，用于减少编译工作、包体与加载负担。关键是保留项目实际会使用的组合，尤其是编辑器当前场景没有体现的运行时路径。
>
> - **Built-in**：检查 `Graphics` 中的 Shader Stripping，例如雾与 Lightmap 模式。依赖场景自动判断时，要覆盖动态加载内容的需求。
> - **URP**：检查 URP Global Settings 与构建中使用的 URP Assets；关闭未使用的功能有助于管线判断可裁剪的组合。结合 Shader Variant 日志检查结果。
> - **调试变体**：开启 Strip Debug 后，Player 中不能使用 Rendering Debugger；需要在构建中调试时保留。
> - **自定义裁剪**：项目使用 `IPreprocessShaders` 等裁剪逻辑时，检查它是否误删 SVC 或运行时需求中的组合。
> - **验证遗漏**：用接近正式发布的构建覆盖场景、质量档、动态加载与关键词切换；出现粉色材质、效果缺失或关键词切换异常时，检查缺失的 Shader / 变体及构建日志。
>
> [[#一、关键设置与验证|返回关键设置]] · [[#4.2 变体收集与预热|查看变体收集与预热]]

---

## 五、关联笔记与资料

- [[TA/动作/小嗯的动作文档#二、Unity 组件与资源怎样配合|Unity 动画组件与 GPU 蒙皮]]。
- [[TA/动作/小嗯的动作文档#3.4 BAT：烘焙骨骼矩阵，播放时 GPU 蒙皮|BAT、Instancing 与 SRP Batcher 的选择]]。
- [[TA/渲染/UE各种技术模块#三、几何复杂度与批处理|UE 与 Unity 的批处理对照]]。

由《小嗯的unity渲染设置》整理迁入。原有设置项、限制和官方参考链接已保留在对应章节，整理日期为 2026-10-30。
