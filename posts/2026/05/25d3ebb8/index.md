# 聊聊5种正则表达式引擎的实现原理

正则表达式看似简单，底层却藏着五花八门的实现思路。本文将带你了解递归引擎、BBVM 虚拟机、Pike VM、Thompson DFA 和导数引擎的工作原理，帮助你在实际项目中做出明智的技术选型。

<!--more-->

## 导语

你可能写过这样的正则：`^(a+)+b$`

它看起来无害，用来匹配以若干个 `a` 开头、以 `b` 结尾的字符串。但如果有人用它验证输入，并传入 `"aaaaaaaac"`...恭喜，你的服务可能直接卡死。

这不是 bug，而是**灾难性回溯**——某些正则引擎在处理特定模式时，会尝试所有可能的匹配路径，时间复杂度从线性变成指数级。这就是为什么理解正则引擎的实现原理，不只是面试题，更关乎生产系统的稳定性。

本文将带你了解五种主流正则引擎的设计思路：**它们如何"消化"你的正则表达式，又是如何决定匹配行为的？** 读完你会对技术选型有更清晰的方向。

## 从一个正则到匹配结果：引擎的使命

当我们写下一个正则 `a(b|c)*d`，引擎需要完成两件事：

1. **编译**：把正则表达式转化为内部可执行的形式
2. **匹配**：用这个"程序"去扫描文本，返回匹配结果

不同的引擎，走的是完全不同的两条路：

| 路线 | 代表 | 核心思路 | 匹配速度 | ReDoS 风险 |
|------|------|----------|----------|------------|
| 递归回溯 | 教科书实现 | 逐分支尝试，失败就"撤销" | 指数级（最坏） | **极高** |
| 回溯虚拟机 | PCRE、Python `re` | 编译成字节码，用 VM 解释执行 | 指数级（优化后较好） | **高** |
| Pike VM | Rust `regex` | 模拟 NFA，多状态并发 | 线性 O(n·m) | **无** |
| Thompson DFA | `grep -E` | 预构建 DFA 状态机 | 线性 O(n) | **无** |
| 导数引擎 | `derivre` | 数学求导 | O(n) 线性 | ❌ | **安全过滤、流式处理** |

下面逐一介绍。

## 递归回溯引擎

递归回溯是正则引擎最直观的实现方式。它的核心思想很朴素：**把正则解析成一棵树，然后对树做递归匹配**。

### 工作原理

假设正则 `(a|b)*c`，解析后的 AST 长这样：

```
Seq
├── Star
│   └── Alt
│       ├── Char('a')
│       └── Char('b')
└── Char('c')
```

匹配时，递归函数会这样工作：

```python
def match(node, text, pos):
    if isinstance(node, Char):
        # 字符匹配：检查当前位置字符是否相同
        return pos + 1 if pos < len(text) and text[pos] == node.c else None
    
    if isinstance(node, Alt):
        # 分支：尝试左分支，失败则回溯尝试右分支
        pos1 = match(node.left, text, pos)
        return pos1 if pos1 is not None else match(node.right, text, pos)
    
    if isinstance(node, Star):
        # 量词：先尝试匹配一次 child，继续贪婪匹配
        pos1 = match(node.child, text, pos)
        if pos1 is not None:
            pos2 = match(node, text, pos1)  # 递归继续匹配 Star
            return pos2 if pos2 is not None else pos
        return pos  # 匹配零次
```

### 回溯的双刃剑

实现确实简单，几十行代码就能跑起来。但**回溯是双刃剑**——对于 `(a|a)+b` 匹配 `"aaa...a"`（一长串 a 加个 b），回溯次数是指数级的。这就是 ReDoS 攻击的温床。

### 适用场景

- 教学演示
- 模式简单、输入可控的内部工具
- 需要支持反向引用等高级特性时（回溯天然支持）

## BBVM 引擎：字节码虚拟机的工业实践

BBVM（Backtracking Bytecode Virtual Machine）是工业界的主流选择——PCRE、Python `re`、JavaScript 的 RegExp 都是这个路数。

### 核心思路

它分两步走：

1. **编译**：把正则编译成字节码指令
2. **解释执行**：虚拟机逐条执行字节码，模拟回溯过程

常见的字节码指令：

- `Char`：匹配单个字符
- `Split`：无条件分叉，记录两个分支待探索
- `Jmp`：跳转
- `Save`：记录捕获组的起止位置
- `Match`：匹配成功

```c
// Split 指令：创建两条执行路径
case Split:
    addthread(inst->x, nlist);  // 路径 A
    addthread(inst->y, nlist);  // 路径 B
    break;
```

### 特点

- **功能强大**：天然支持捕获组、环视、反向引用
- **性能不稳定**：回溯在最坏情况下仍是指数级
- **优化空间大**：现代实现（RE2、PCRE2）会加各种剪枝缓存

### 实际案例

Python 的 `re` 模块就是典型的 BBVM 实现。当你写出这样的模式时：

```python
import re
re.search(r'(a+)+b', 'aaaaaaaab')
```

Python 会在内部把它编译成类似的字节码序列，然后由虚拟机解释执行。这套机制足够通用，但也意味着**无法从根本上避免 ReDoS**——即使加了时间限制，恶意构造的模式仍能让服务器 CPU 打满。

## Pike VM：NFA 模拟的优雅解法

Pike VM（以 Rob Pike 命名）是另一种思路：**不一条路走到底，而是同时"跟踪"所有可能的路径**。

### 核心思想：并发模拟

Pike VM 维护两个**活跃状态集合**：

- `clist`：当前字符位置所有可能的执行状态
- `nlist`：处理完当前字符后的新状态集

对每个输入字符，它会：

1. 遍历 `clist` 中的每个状态
2. 根据指令产生新状态，加入 `nlist`
3. 交换 `clist` 和 `nlist`，继续下一个字符

关键在于 **`addthread` 会去重**——同一个程序计数器只保留一个线程，不会指数膨胀。

### 代码示意

下面是完整的实现：

```c
// 添加线程到待执行列表，自动去重
void addthread(int pc, int *list, int *count) {
    // 检查是否已存在（去重）
    for (int i = 0; i < *count; i++) {
        if (list[i] == pc) return;  // 已存在，直接跳过
    }
    // 不存在则加入
    list[(*count)++] = pc;
}

bool pike_vm(Inst *prog, char *input) {
    int curr[100], next[100], curr_cnt = 0, next_cnt = 0;

    addthread(0, curr, &curr_cnt);  // 初始化：从起点开始

    for (char *p = input; *p; p++) {
        for (int i = 0; i < curr_cnt; i++) {
            Inst *ip = &prog[curr[i]];
            switch (ip->opcode) {
                case Char:
                    if (*p == ip->c)
                        addthread(curr[i] + 1, next, &next_cnt);
                    break;
                case Split:
                    // 分叉：两个分支都加入待执行列表
                    addthread(ip->x, next, &next_cnt);
                    addthread(ip->y, next, &next_cnt);
                    break;
                case Match:
                    return true;
            }
        }
        // 交换状态集，准备处理下一个字符
        memcpy(curr, next, next_cnt * sizeof(int));
        curr_cnt = next_cnt;
        next_cnt = 0;
    }
    return false;
}
```

#### 去重的威力

以正则 `(a|aa)*` 匹配 `"aaaa"` 为例：无回溯引擎会尝试所有分支，路径数指数增长；但 Pike VM 的 `addthread` 会发现：无论走 `"a"` 分支还是 `"aa"` 分支，最终都落在同一个程序计数器（因为 `aa` 匹配后恰好等价于继续匹配），所以只保留一条线程。

这就是为什么即使正则有大量分支，每个字符位置同时处理的线程数也**不超过 NFA 状态数**——不是减少尝试，而是**合并等价状态**，从根本上消灭指数爆炸的土壤。

### 为什么更安全

无论正则多复杂，每个字符位置处理的活跃状态数受限于**正则本身的大小**。因此时间复杂度是 **O(n·m)**（n = 文本长度，m ≈ 正则大小），**永远不会指数爆炸**。

Rust 官方的 `regex` 库就是基于 Pike VM，所以你在 Rust 项目中使用正则时可以安心——**不会因为某个极端模式把程序卡死**。

## Thompson DFA：子集构造的极致性能

如果 Pike VM 是"运行时多路并发"，那 Thompson DFA 就是**预处理阶段就把所有路径合并好**。

### 核心思路

1. 把正则编译成 ε-NFA（Thompson 构造法）
2. 用**子集构造算法**把所有 NFA 状态集合"压缩"成 DFA 状态
3. 匹配时：每读一个字符，只需查一次转移表

```python
# 简化版子集构造
def nfa_to_dfa(nfa):
    start_closure = epsilon_closure({nfa.start})  # 计算起始状态的 ε 闭包
    dfa_states = {frozenset(start_closure)}
    worklist = [dfa_states[0]]
    
    while worklist:
        state = worklist.pop()
        for char in alphabet:
            # 计算字符触发后的 ε 闭包
            next_state = epsilon_closure(move(state, char))
            if next_state not in dfa_states:
                dfa_states.add(next_state)
                worklist.append(next_state)
            dfa.transitions[state][char] = next_state
```

### 优缺点

| 优点 | 缺点 |
|------|------|
| **极快**：每个字符一步查表 | **状态可能爆炸**：最坏 2^m 个 DFA 状态 |
| 适合流式处理 | 不支持捕获组 |
| 内存固定（预处理后） | 不支持反向引用 |

`grep -E`、`lex`、许多高性能日志分析工具都用这套方案。

### 深入：为什么 DFA 不支持捕获组？

你可能好奇：Thompson DFA 速度那么快，为什么连捕获组都不支持？这里触及了正则引擎理论的一个核心权衡：**确定性 vs. 信息保留**。

**根本原因**：子集构造法合并了路径，从而丢失了"哪条分支被选择"的历史信息。而捕获组恰恰需要记住这些分支中的特定位置。

#### 捕获组的本质

捕获组（如 `(a|b)`）要求引擎在匹配过程中记录**该组对应的子串开始和结束位置**。这本质上是"带记忆的匹配"——同一个输入可能对应多种解析树，捕获组要选出其中一种并记录其边界。

#### 子集构造丢失了什么

Thompson 构造法将正则转为 ε-NFA，子集构造再把 NFA 状态集合"压缩"成 DFA 状态。**关键在于：DFA 中没有"选择"**——给定当前状态和下一个字符，下一个状态是唯一确定的，所有可能的分支都被合并了。

以正则 `(a|b)c` 为例：读 `a` 后 NFA 活跃集 = {3}（匹配了 `a`），读 `b` 后活跃集 = {5}（匹配了 `b`）。但 DFA 会把它们合并成同一个状态，因为两者"都准备接受 `c`"。此时你无法区分：**捕获组内容究竟是 `a` 还是 `b`？** 历史上走过哪条分支已经丢失。

#### 强行支持的后果：状态爆炸

如果非要保留这些历史，需要一种"带标记的 DFA"（Tagged DFA）：每个 DFA 状态携带捕获组的位置信息。但路径合并时标记会冲突，必须把冲突路径拆成不同状态，导致**状态数指数爆炸**。

正则 `(a|b)*(c|d)` 有 k 个捕获组，理论上需要 O(2^k) 个 DFA 状态来区分所有捕获历史——在实际中不可接受。

#### 工业界的折中方案

- **纯 DFA**（`grep`、`lex`）：不支持捕获组，或仅支持整个匹配
- **混合方法**：先用 DFA 快速扫描，发现匹配区域后用 NFA 模拟提取捕获组（如 RE2）
- **直接模拟 NFA**（Pike VM、Rust `regex`）：在支持捕获组的同时保持线性时间

> **一句话总结**：Thompson DFA 不支持捕获组，因为子集构造把 NFA 的多路径合并成一个确定状态，从而丢弃了用于确定捕获组边界的"行走历史"；若强行保留会导致状态数指数爆炸，失去 DFA 的紧凑优势。

## 导数引擎：数学家的浪漫

最后登场的是一个"学院派"——**导数引擎**（以 `derivre` 为代表）。它用数学上的"求导"替代了回溯。

### 核心思想

对于正则 `R` 和字符 `c`，"导数" `∂c(R)` 表示：**在接受了 `c` 之后，还需要匹配什么**。

举几个推导规则：

- `∂c(char(d)) = ε` 当 `c == d`（匹配成功）
- `∂c(char(d)) = ∅` 当 `c != d`（匹配失败）
- `∂c(R | S) = ∂c(R) | ∂c(S)`（选择分支的导数）
- `∂c(RS) = ∂c(R)S | nullable(R)∂c(S)`（序列的导数）

匹配过程就是一个**逐字符求导的循环**：

```python
def is_match(rx, text):
    state = rx  # 初始状态是正则本身
    for ch in text:
        state = derivative(state, ch)  # 求导
        if state == Empty:  # 空集 = 永远匹配不了
            return False
    return nullable(state)  # 最终检查是否能接受空串
```

### 它的独特优势

- **线性时间复杂度**：与正则复杂度无关
- **原生支持集合运算**：理论上支持交集 (`&`) 和补集 (`~`)
- **免疫 ReDoS**：没有回溯机制

代价是**功能受限**（通常不支持反向引用），且学习曲线较陡。

## 总结与选型指南

| 引擎 | 时间复杂度 | 捕获组 | 反向引用 | 适用场景 |
|------|-----------|--------|----------|----------|
| 递归回溯 | 最坏指数级 | ✅ | ✅ | 教学、简单工具 |
| BBVM | 最坏指数级 | ✅ | ✅ | 通用文本处理（需注意 ReDoS） |
| Pike VM | O(n·m) 线性 | ✅ | ❌ | Rust regex、RE2（推荐） |
| Thompson DFA | O(n) 线性 | ❌ | ❌ | `grep`、流式处理、词法分析 |
| derivre | O(n) 线性 | 部分 | ❌ | 安全敏感、流式处理 |

### 选型建议

- **处理用户输入的正则**：优先选 Pike VM 或导数引擎，避免 ReDoS 风险
- **高性能搜索工具**：Thompson DFA，速度最快
- **需要捕获组**：Pike VM（Rust `regex`）或 BBVM（Python `re`）
- **教学演示**：从递归回溯入手，理解回溯的本质

正则表达式看似简单，背后却是编译器原理、有限自动机理论、性能优化的精彩交汇。读完本文，你应该能：**

- 区分五类主流引擎的核心差异
- 理解为什么 Thompson DFA 不支持捕获组
- 了解 Pike VM 如何用"去重"消灭指数爆炸
- 在实际项目中做出明智的正则选型

## 参考资料

### 核心概念与理论

- [正则表达式 - Wikipedia](https://en.wikipedia.org/wiki/Regular_expression)
- [NFA 到 DFA 的子集构造算法](https://en.wikipedia.org/wiki/Powerset_construction)
- [Thompson NFA 构造法 - 原始论文](https://dl.acm.org/doi/10.1145/363347.363387)

### ReDoS 与安全

- [正则表达式拒绝服务 (ReDoS) - OWASP](https://owasp.org/www-community/attacks/Regular_expression_Denial_of_service_-_ReDoS)
- [Python `re` 模块文档](https://docs.python.org/3/library/re.html)
- [PCRE - Perl Compatible Regular Expressions](https://www.pcre.org/)

### 各引擎实现

- [Rust `regex` 库](https://docs.rs/regex/latest/regex/)（Pike VM 实现）
- [RE2 库](https://github.com/google/regexp)（Google 的线性时间正则引擎）
- [Go `regexp` 包](https://pkg.go.dev/regexp)（基于 Pike 的实现）
- [`derivre` - 导数引擎的 Rust 实现](https://github.com/jameysharp/derivre)
- [pikevm - Pike VM 的参考实现](https://github.com/jameysharp/pikevm)

### 学术论文

- J. A. Brzozowski, "Derivatives of Regular Expressions" (1964) - 导数理论奠基之作
- S. Owens et al., "Regular-expression Derivatives Re-examined" (2009) - 导数引擎的现代复兴
- A. V. Aho et al., *Compilers: Principles, Techniques, and Tools* (Dragon Book) - 编译器经典教材，涵盖 NFA/DFA

