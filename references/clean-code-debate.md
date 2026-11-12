# Clean Code 骂战档案——数据、观点与出处

> 本文档是 SKILL.md "performance counterweight" 部分的完整背景：Muratori 的实验与数据、Uncle Bob 的回应、Hovinen 的反驳，以及综合判断。所有数据均引用自原文测量（2023），本仓库未复测。

## 实验设计

Muratori 用 Uncle Bob《Clean Code》教学用的经典例子做对照实验：一堆形状，求总面积。三个版本：

```cpp
// 版本 1 "整洁"：多态 + 虚函数（践行"用多态代替 switch"）
struct shape_base { virtual f32 Area() = 0; };
struct square    : shape_base { f32 W, H; f32 Area() override { return W * H; } };
struct triangle  : shape_base { f32 W, H; f32 Area() override { return 0.5f * W * H; } };
struct circle    : shape_base { f32 W, H; f32 Area() override { return 0.5f * Pi * W * H; } };

// 版本 2：扁平结构体 + switch（教科书不齿的写法）
switch (Shapes[i].Type) {
    case Shape_square:    Accum += Shapes[i].W * Shapes[i].H; break;
    case Shape_triangle:  Accum += 0.5f * Shapes[i].W * Shapes[i].H; break;
    // ...
}

// 版本 3：发现四个公式全是 系数×W×H → 查表
f32 const CTable[] = {1.f, 1.f, 0.5f, 0.5f * Pi};
Accum += CTable[Shapes[i].Type] * Shapes[i].W * Shapes[i].H;
```

## 测量数据

| 版本 | cycles/shape | 相对速度 |
|---|---|---|
| 多态 + 虚函数（"整洁"） | ~35 | 基准 |
| switch | ~24 | 快 1.5 倍 |
| 查表 | ~3 | 快 **10 倍** |
| AVX 向量化版 | — | 快 **20–25 倍** |

量化语境：1.5 倍 ≈ 把 iPhone 14 Pro Max 降级成 iPhone 11 Pro Max（三年硬件进化蒸发）；10–15 倍 ≈ CPU 倒退约十二年。

对 Clean Code 结构性规则的逐条判决（他的结论）："一条值得想想，四条最好别碰"：

- **多态优于 switch**：代价 1.5x；指针间接 + 不透明虚调用让编译器看不见代码路径
- **不应了解对象内部**：打破它（数据模型与操作融合）恰恰带来 10–15x 收益，且代码**更少**、语义复杂度**更低**
- **小函数/单一职责**：问题越复杂 clean code 缩放越差；拆散到多个编译单元+虚分发限制了编译器**和程序员**能看见的范围
- **DRY**：他基本认同（"别写两遍一模一样的代码是合理建议"），仅偶尔为性能复制表
- 另一论点：switch 让**跨类型的模式**可见（四个公式只差一个系数一眼可见），类层次把它撕碎进四个文件

## 后续骂战

1. 文章发出后 [HN 七百多分](https://news.ycombinator.com/item?id=34966137)大讨论
2. Uncle Bob 下场，两人在 GitHub 留下[公开问答记录](https://github.com/unclebob/cmuratori-discussion/blob/main/cleancodeqa.md)：Martin 为 Clean Code 作为策略辩护，同时承认在许多性能关键场景"代码了解具体类型好过虚分发"
3. 旁观者共识：**鸡同鸭讲**——Muratori 主场是游戏引擎/系统编程（热点路径 cycle 就是钱），Martin 写的是企业软件方法论（瓶颈在需求变更和理解成本）
4. [Hovinen 的反驳](https://hovinen.me/blog/2024/04/08/clean-code-horrible-performance-so-what)：
   - 把 CPU 速度置于可理解性之上是"悲剧性的优先级倒置"
   - 更好的定义："**clean code 最小化阅读者的额外认知负荷**"
   - ROI 算账：一小时约 $100 的优化省 250 个 cycle，代码需跑 36 万亿次才回本——1000 QPS 下要一千多年
   - 指出 Muratori 打的是稻草人：多态优于 switch 本来就是启发式不是铁律，shape 教学例子不代表真实继承用法
   - 软件变慢的真正原因是功能期望膨胀（跨平台、安全、无障碍、4K、emoji），不是代码"太整洁"

## 综合判断（本 skill 的立场）

**他们共同反对的**：不匹配问题结构的抽象——为"整洁"拆出的虚函数、为"复用"提前织出的间接层、教条驱动的接口隔离。这种代码人和编译器都得绕着走。

**他们真正分歧的**：性能敏感度。业务冷路径里可读性复利几乎永远赚；引擎热点里一次指针追逐几乎永远亏。这个判断无法外包给任何一方。

**关键观察**：用 Logan Smith 的尺子量 Muratori 的查表版本——`TotalArea(shape*, u32)` 输入走签名、输出走返回值、无全局状态 = **诚实函数**；函数体一行贴着问题本身的层级 = **单一抽象层级**。Muratori 的优化过程本身就是一次 Smith 式重构：他发现 switch 四个分支是同一块砖，把它抽成了一张表（抽出的砖是**数据**而非函数调用）。最快的版本同时也是最"干净"的版本，这不是巧合——**当抽象匹配问题结构时，人、编译器、性能三者利益一致**。

Muratori 暴论的真正价值：把"读者"的范围扩到了编译器——抽象不仅对人隐藏细节，也对优化器隐藏细节；当被隐藏的不是问题的本质结构时，就在同时给人和机器交"理解税"。

## 来源

- [《"Clean" Code, Horrible Performance》— Casey Muratori (2023)](https://www.computerenhance.com/p/clean-code-horrible-performance)
- [SE Radio #577: Casey Muratori 访谈](https://se-radio.net/2023/08/se-radio-577-casey-muratori-on-clean-code-horrible-performance)
- [Uncle Bob × Muratori 公开问答（GitHub）](https://github.com/unclebob/cmuratori-discussion/blob/main/cleancodeqa.md)
- [Clean Code, Horrible Performance. So what? — Bradford Hovinen (2024)](https://hovinen.me/blog/2024/04/08/clean-code-horrible-performance-so-what)
- [HN 讨论串](https://news.ycombinator.com/item?id=34966137)
- [PVS-Studio：Clean code: blessing or curse?](https://pvs-studio.com/en/blog/posts/1157)
- [How to write the perfect function — Logan Smith](https://www.youtube.com/watch?v=2OMRWPOSw9s)（[B站搬运](https://www.bilibili.com/video/BV1sv8B6ZEkB/)）
