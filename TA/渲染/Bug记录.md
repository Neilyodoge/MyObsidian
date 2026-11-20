## 2026-11-20｜红米 5：Nova 四边特效变洋红，报 `variant not found`

**现象**：同一套四边警示特效在红米 5（Adreno 506，OpenGL ES 3.2，驱动 V@269.0）上变成整片洋红。开启 Unity 严格变体匹配后，日志反复报 `Nova/Particles/UberUnlit` 的极坐标＋染色变体找不到。

**修复**：在 `ParticlesUber.hlsl` 的 `RotateTintMapUV()` 中，只删掉 `if (_TintMapRotation >= 2.5h)` 上方的 `UNITY_BRANCH`。四档旋转、切档阈值、材质和关键词都保留。TH 测试提交：`0263eb0b23b`。

![[TA/渲染/附件/2026-11-20-Nova-Redmi5-UNITY_BRANCH-AB.png|540]]

左：保留提示的 A 包；右：只去掉提示的 B 包。同一台红米、同一场景，均为冷启动第 19 秒。

### 为什么报“找不到变体”

A、B 两个最终 APK 的 Pass 0 都包含 `_POLAR_COORDS_ON _TINT_AREA_ALL _TINT_MAP_ENABLED`。A 包冷启动时，**第一条图形错误是链接失败**，随后 Unity 才报找不到变体：

```text
Shader Nova/Particles/UberUnlit
GLSL link failed, no info log provided.
Creation of internal variant of shader 'Nova/Particles/UberUnlit' failed.
… variant _POLAR_COORDS_ON _TINT_AREA_ALL _TINT_MAP_ENABLED not found.
```

也就是说，变体打进了 A 包，顶点和片元程序分别编译成功，但驱动没能把它们链接成可绘制的程序。B 包四边特效恢复，冷启动观察 26 秒，两类错误均为 0。

### 怎么定位到顶点 UV 分支

从两个 APK 取出实际编译的 GLES 程序，在同一台红米上交叉配对。下表各组均通过单阶段编译；结果一栏指**链接**。正序和倒序各跑一次，结果一致。

| 顶点程序 | 片元程序 | 链接 |
|---|---|---|
| A：原版 | A 或 B | 都失败 |
| B：去掉提示 | A 或 B | 都通过 |
| A：保留外层 `if`，四档运算简化为只取 UV 分量 | A | 失败 |
| A：保留四档旋转公式，只把外层 `if` 换成 `条件 ? 结果A : 结果B` | A | 通过 |

单独核过普通 UV、超出 0～1 的 UV 和三个切档点 `0.5 / 1.5 / 2.5`，修复前后旋转结果一致。证据指向**这套顶点程序里染色 UV 旋转的分支写法**。驱动没有给出内部原因；其他位置仍有 `UNITY_BRANCH` 的 B 包可以正常显示，无需全局删除，也无需按机型或 GLES 版本加特判。

> [!tip] 下次看到 `variant not found`
> 先往前找同一 Shader 的第一条错误，再核对**最终包里的 Pass 和完整关键词组合**。包里没有该组合，再查变体收集和剥离；包里有，且前面先报 `GLSL link failed`，优先在出问题的真机上查链接。只看最后一行，容易误判为要补 SVC。