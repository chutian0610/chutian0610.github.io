# EWMA 指数加权移动平均统计方法


如果你写过监控告警、自适应限流或者负载均衡，大概率遇到过这个问题：怎么用一个数值，实时地描述"当前系统有多慢"？EWMA（Exponentially Weighted Moving Average，指数加权移动平均）几乎是工程实践里的标准答案——它只需要一个 float 的内存，一行加法就能更新，却能给出比简单平均更贴近"当下"的估计。

<!--more-->

## 先说结论

EWMA 的更新公式只有一行：

$$S_t = \alpha \cdot x_t + (1 - \alpha) \cdot S_{t-1}$$

其中 $x_t$ 是本次观测值，$S_{t-1}$ 是上一次的平滑值，$\alpha \in (0, 1)$ 是平滑系数。

把递推式展开就能看出"指数加权"这个名字的来历：

$$S_t = \alpha x_t + \alpha(1-\alpha) x_{t-1} + \alpha(1-\alpha)^2 x_{t-2} + \cdots$$

**每个历史样本的权重按 $(1-\alpha)^k$ 指数衰减**。越老的数据影响越小，但从不真正归零。这一句话就概括了 EWMA 的全部性格：**它记得所有历史，但只在乎最近**。

## 为什么不用其他方法

假设我们要统计接口的响应耗时，对比几种常见做法：

### 累计平均（Cumulative Average）

$$S_t = \frac{1}{t}\sum_{i=1}^{t} x_i$$

内存 O(1)，但有个致命问题：**样本量越大，新数据的话语权越小**。一个跑了三天的服务累计了千万级请求，此时突然发生 GC 抖动，耗时从 5ms 涨到 500ms，累计平均可能只从 5.0 挪到 5.05。等它反应过来，故障早已过去。

累计平均适合描述"这个服务历史上有多快"，不适合回答"它现在有多慢"。

### 简单移动平均（SMA / 滑动窗口）

只保留最近 N 个样本求平均。这解决了时效性问题，但代价明显：

- **内存 O(N)**：QPS 上万时，一个 1 分钟窗口就是几十万个样本；
- **权重不合理**：窗口内所有样本等权，第 1 个和第 N 个一样重要，但窗口外的样本权重瞬间归零；
- **窗口效应**：一个极端值进入窗口时曲线跳一次，滑出窗口时再跳一次，出现虚假的"双峰"。

滑动窗口的边界是硬切的，而 EWMA 的边界是软的——这是它在监控场景更受欢迎的主要原因。

### 分位数统计（P99 / TDigest）

分位数能刻画长尾，信息量比均值大得多，但**内存和计算成本高一个量级**，且难以做增量的时间衰减。生产上常见的组合是：用 TDigest 出报表看长尾分布，用 EWMA 做实时决策。

### 横向对比

| 方法 | 内存 | 更新成本 | 时效性 | 突变响应 |
|------|------|----------|--------|----------|
| 累计平均 | O(1) | O(1) | 差 | 迟钝 |
| 简单移动平均 | O(N) | O(1) | 好 | 有窗口效应 |
| EWMA | O(1) | O(1) | 好 | 平滑可调 |
| TDigest 分位数 | O(压缩比) | O(log n) | 依赖实现 | 成本高 |

EWMA 的定位很清晰：**用 O(1) 的成本，换取接近滑动窗口的时效性**。

## α 该怎么选

α 是唯一的调参旋钮，它决定了这条曲线的"脾气"：

- **α 越大**（趋近 1）：越信任新数据，响应快，但对抖动敏感、噪声大；
- **α 越小**（趋近 0）：越平滑稳定，但对突变反应迟钝。

工程上有两种常用的换算方式，把抽象的 α 翻译成能理解的物理量。

**按样本数换算**——等效于"N 个样本的滑动窗口"：

$$\alpha = \frac{2}{N + 1}$$

这是股票技术分析里 EMA 的经典公式，N=20 对应 α≈0.095。

**按时间常数换算**——这在采样间隔不均匀时更实用：

$$\alpha = 1 - e^{-\Delta t / \tau}$$

其中 $\Delta t$ 是距上次更新的实际时间间隔，$\tau$ 是时间常数（衰减半衰期约 $0.69\tau$）。Linux 内核的 load average 和 gRPC 的负载均衡器用的都是这个形式。

**为什么时间衰减版本更重要？** 因为真实的请求不是均匀到达的。如果服务空闲了 10 分钟，按样本数衰减的 EWMA 会认为"上一个值就是当前值"；而按时间衰减的版本知道"这 10 分钟里数据已经很陈旧了"，会给新样本更高的权重。

## 代码实现

### 基础版本

```go
// EWMA 基础实现：按样本数衰减，适用于均匀采样场景
type EWMA struct {
    alpha float64
    value float64
    init  bool // 标记是否已有初值，避免从 0 缓慢爬升
}

func NewEWMA(alpha float64) *EWMA {
    return &EWMA{alpha: alpha}
}

func (e *EWMA) Update(x float64) {
    if !e.init {
        // 冷启动直接用首个样本作为初值，绕开预热偏差
        e.value, e.init = x, true
        return
    }
    e.value = e.alpha*x + (1-e.alpha)*e.value
}

func (e *EWMA) Value() float64 { return e.value }
```

注意 `init` 这个字段。如果初值取 0，前几十个样本都会被这个虚假的 0 拖住，形成明显的"预热爬坡"。直接用首个样本做初值是最简单的规避方式（另一种是 Adam 优化器里的偏差修正 $S_t / (1 - (1-\alpha)^t)$）。

### 时间衰减版本

```go
// TimeDecayEWMA：按真实时间间隔衰减，适用于非均匀采样
type TimeDecayEWMA struct {
    tau      time.Duration // 时间常数，越大越平滑
    value    float64
    lastTime time.Time
    mu       sync.Mutex
}

func (e *TimeDecayEWMA) Update(x float64, now time.Time) float64 {
    e.mu.Lock()
    defer e.mu.Unlock()

    if e.lastTime.IsZero() {
        e.value, e.lastTime = x, now
        return e.value
    }
    // 间隔越长，alpha 越接近 1，即历史值几乎被完全遗忘
    delta := now.Sub(e.lastTime).Seconds()
    alpha := 1 - math.Exp(-delta/e.tau.Seconds())

    e.value = alpha*x + (1-alpha)*e.value
    e.lastTime = now
    return e.value
}
```

高并发下如果不想加锁，可以用 `atomic.Uint64` 存 `math.Float64bits(value)`，配合 CAS 循环更新——代价是极端竞争时可能丢失个别样本，对统计量而言通常可以接受。

### Python 版：一行搞定

```python
import pandas as pd

s = pd.Series([10, 12, 11, 50, 13, 12])  # 第 4 个是突刺
# span=5 等价于 alpha = 2/(5+1) ≈ 0.333
print(s.ewm(span=5, adjust=False).mean().round(2).tolist())
# [10.0, 10.67, 10.78, 23.85, 20.23, 17.49]
```

对照一下：简单平均会把突刺 50 均摊到全局，5 窗口 SMA 会让 50 在窗口内持续 5 拍再突然消失，而 EWMA 是**跳起来然后指数式回落**——这条曲线的形状，正是告警系统想要的。

## 编程领域的四个实战场景

### 场景一：自适应限流的耗时基线

Sentinel、Netflix 的 concurrency-limits 这类自适应限流器，核心思路是拿"当前 RT"和"历史基线 RT"比较，超过阈值就降低并发上限。这里的基线必须满足两个条件：跟得上业务的自然波动，又不被瞬时毛刺带偏。

```go
// 简化的自适应限流判断逻辑
func (l *Limiter) Allow(rt time.Duration) bool {
    cur := l.rtEWMA.Update(float64(rt.Milliseconds()), time.Now())
    // 当前请求耗时超过基线 2 倍，且并发已到上限时拒绝
    return !(float64(rt.Milliseconds()) > cur*2 && l.inflight.Load() >= l.limit.Load())
}
```

### 场景二：P2C 负载均衡的节点评分

gRPC-go 和 Kratos 的 EWMA 负载均衡器，为每个后端节点维护一个"耗时 EWMA + 成功率 EWMA"的加权分数，每次请求随机挑两个节点，选分数更好的那个（Power of Two Choices）。

用 EWMA 而非累计平均的原因很直接：**节点可能从故障中恢复**。累计平均会让一个曾经宕机的节点背负历史包袱很久，而 EWMA 让它在几十次成功请求后就重新回到候选池。

### 场景三：熔断器的错误率统计

Hystrix 用的是滑动窗口计数，但滑动窗口在 QPS 波动大时统计不稳定——低流量窗口里 2 个错误就是 50% 错误率。改用 EWMA 统计错误率可以缓解这个问题：

```go
// 每次请求后更新错误率 EWMA
func (b *Breaker) record(success bool) {
    v := 0.0
    if !success {
        v = 1.0
    }
    if b.errRate.Update(v, time.Now()) > 0.5 {
        b.trip() // 触发熔断
    }
}
```

**注意这里的隐含语义**：把 0/1 序列喂给 EWMA，得到的就是"指数加权的错误率"，这个技巧在统计任何比率型指标时都适用。

### 场景四：TCP 的 RTT 估计

这是 EWMA 在计算机领域最经典、也最古老的应用。TCP 需要根据 RTT 决定超时重传时间（RTO），RFC 6298 定义的算法就是两个 EWMA：

```text
SRTT   = (1 - 1/8) * SRTT   + (1/8) * R        # 平滑 RTT，alpha = 1/8
RTTVAR = (1 - 1/4) * RTTVAR + (1/4) * |SRTT-R| # RTT 波动，alpha = 1/4
RTO    = SRTT + 4 * RTTVAR
```

有意思的是第二行：**对偏差本身再做一次 EWMA**，就得到了平滑的方差估计。RTO 同时考虑均值和波动——网络越不稳定，超时时间给得越宽松。这个"均值 + 方差"双 EWMA 的模式，同样适用于动态告警阈值（比如阈值设为 $\mu + 3\sigma$，两者都由 EWMA 实时维护）。

## 几个容易踩的坑

**冷启动偏差**：初值取 0 会导致前期系统性偏低。用首样本初始化，或者做偏差修正。

**长时间静默后的失真**：按样本数衰减的 EWMA 在服务空闲后会保留陈旧值。如果你的流量有明显波谷，务必用时间衰减版本。

**α 与采样频率耦合**：同一个 α，在 100 QPS 和 10000 QPS 下的"记忆时长"差 100 倍。跨环境部署时，按时间常数配置比按 α 配置稳健得多。

**别指望它检测长尾**：EWMA 是均值估计，对 P99 恶化不敏感。1% 的请求从 10ms 涨到 1s，均值只从 10ms 涨到 20ms。需要长尾就老实上分位数统计。

## 小结

EWMA 的价值不在数学上有多精妙——它就是一行加权加法。它的价值在于**用最低的成本（O(1) 内存 + 一次乘加），提供了一个"随时间遗忘"的语义**，而这恰好是绝大多数在线系统对"当前状态"的真实需求。

选型时记住三条：需要"最近的均值"就用 EWMA；采样不均匀就用时间衰减版本；需要长尾就换分位数。

