# clean-code-skill

一个给 agent 用的代码判断力 skill：把 Logan Smith 的《How to write the perfect function》演讲和 Casey Muratori 的《"Clean" Code, Horrible Performance》骂战，浓缩成一套写函数、重构、做 code review 时可直接套用的判断框架。

它不教 agent "什么是规则"，而是教它**怎么判断**——三个测试 + 一个性能对冲：

1. **Honesty test**：函数是否只通过签名与外界打交道？不诚实会传染，所以不纯的东西（IO/时间/随机）要推到调用树最顶层注入
2. **Signature empathy test**：不索要超出所需的保证（只要循环就别要 `vector`），真有强保证需求就让类型说话（回执类型、不变量类型），而不是写在注释里
3. **Abstraction level test**：函数体每行同一个抽象层级；分段注释是混层级的雷达；发现自己在手工维护 ad-hoc 数据结构时，把它封装掉
4. **性能对冲**：热点路径上，间接层对人和编译器都在收"理解税"（Muratori 的数据：虚函数 → switch → 查表 = 35 → 24 → 3 cycles）；反过来，冷路径上为 250 个 cycle 牺牲可读性要跑一千多年才回本（Hovinen 的账）

## 来源

| 材料 | 链接 |
|---|---|
| How to write the perfect function — Logan Smith | [YouTube](https://www.youtube.com/watch?v=2OMRWPOSw9s) ／ [B站搬运](https://www.bilibili.com/video/BV1sv8B6ZEkB/) |
| 《"Clean" Code, Horrible Performance》— Casey Muratori | [computerenhance.com](https://www.computerenhance.com/p/clean-code-horrible-performance) |
| Uncle Bob × Muratori 公开问答 | [GitHub](https://github.com/unclebob/cmuratori-discussion/blob/main/cleancodeqa.md) |
| Clean Code, Horrible Performance. So what? — Bradford Hovinen | [hovinen.me](https://hovinen.me/blog/2024/04/08/clean-code-horrible-performance-so-what) |

原始材料消化过程的产物之一：[函数该怎么写——从一段演讲和一场骂战说起](https://l1rics06.github.io/posts/how-to-write-the-perfect-function/)

## 安装

适用于任何支持 SKILL.md 约定的 agent 环境（ZCode、Claude Code 等），零依赖、无脚本：

```bash
# ZCode / 通用标准位置
git clone https://github.com/L1rics06/clean-code-skill ~/.agents/skills/clean-code

# 或 ZCode 用户级目录（优先级更高）
git clone https://github.com/L1rics06/clean-code-skill ~/.zcode/skills/clean-code

# 或 Claude Code
git clone https://github.com/L1rics06/clean-code-skill ~/.claude/skills/clean-code
```

装好后新开会话即可。触发不需要魔法咒语——正常说"帮我重构这个函数"、"review 一下这段代码"、"这里该用多态还是 switch"都可能命中；也可以显式调用 `/skill clean-code`。

## 用法示例

```
用 clean code 的标准看看这个函数有什么问题
```

```
把这段代码重构得更诚实一点，全局状态都拿到签名里去
```

```
我们在写一个粒子系统，这个 populate 函数的设计你觉得怎么样？
```

## 结构

```
├── SKILL.md                      # agent 入口：三个测试 + 性能对冲（常驻上下文的只有这个）
└── references/                   # 按需加载，平时不占 token
    ├── perfect-function.md       # 演讲完整提炼：全部原则 + 代码示例 + 函数分类学
    └── clean-code-debate.md      # 骂战档案：测量数据、逐条判决、三方观点、出处
```

## 边界声明（诚实比整洁重要）

- 这是**判断启发式，不是规则引擎**。三条测试之间、测试与性能对冲之间出现冲突是常态，最终裁量权在 agent 和用户——skill 教的是怎么权衡，不是替你权衡
- 文中所有性能数字引用自 Muratori 2023 年原文的测量，**本仓库未复测**；不同硬件/编译器/负载下结论可能不同
- 示例是 C++，原则跨语言；但"热点路径"的判断依赖 profiler，不依赖本 skill 的任何先验
- 本 skill 不会让 agent 自动重构你的代码——它只在被问到时提供判断框架

## License

[MIT](LICENSE)
