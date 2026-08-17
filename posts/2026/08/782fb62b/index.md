# BBR算法


## 一、从一个常见但反直觉的现象说起

你可能听过这样的话："网线明明是千兆的，为什么下载速度总是上不去？"

或者更具体一点：你用 speedtest 测速，自己买的 200M 宽带，下行只能跑到 80M；公司升级到 10G 专线，看 YouTube 4K 仍然时不时卡顿。

很多人第一反应是"运营商限速"或"网站服务器不行"。但如果你是做后端的、或者管过一段广域网链路，会知道**罪魁祸首往往不是带宽不够，而是 TCP 的拥塞控制策略本身**。

这里有个反直觉的事实：**传统 TCP 拥塞控制（比如 CUBIC）在高带宽、长往返时延（**BDP** 很大）的链路上，根本无法跑满物理带宽**。一条 RTT = 100ms 的 10Gbps 链路，理论上限能容纳 ~125MB in-flight，但 CUBIC 要靠"丢包才知道堵了"，丢包时缓冲区已经堆了几十 MB 排队数据——**真正的可用带宽被排队延迟吃掉了**。

这就是 BBR 试图解决的问题。

2016 年，Google 公开了 **BBR（Bottleneck Bandwidth and Round-trip propagation time）** 算法。十年过去，它已经写进 Linux 内核默认选项，被 YouTube、Cloudflare、Netflix 等大规模部署。**如果说 CUBIC 是 AIMD 范式的顶配，BBR 就是 AIMD 范式之外的一次范式跃迁**。

## 二、先说结论

BBR 的核心思想只有一句话：**不再把"丢包"当作拥塞信号，而是主动测量链路的瓶颈带宽（BtlBw）和最小往返时延（RTprop），让发送速率精确工作在「瓶颈带宽 × 最小 RTT = BDP」附近**。

形式化：

$$
\text{理想 in-flight} = \text{BtlBw} \times \text{RTprop}
$$

其中：

- **BtlBw（Bottleneck Bandwidth）**：路径上最窄一段链路的最大交付速率；
- **RTprop（Round-trip propagation time）**：去掉排队延迟后的"纯物理"往返时延。

**把这条公式翻译成大白话**：BBR 的目标不是"把链路塞满"，而是"用最少的 in-flight 数据，占据瓶颈带宽，同时不引入排队延迟"。

这与传统 AIMD 范式有本质区别——AIMD 假设"丢包即拥塞"，BBR 假设"排队延迟即拥塞"。

## 三、为什么需要 BBR：AIMD 范式的天花板

在上一篇《AIMD 加性增 乘性减算法》里我们聊过：AIMD 是一条挺优雅的规则，几十年支撑起互联网。但它有几个绕不开的硬伤。

### 3.1 锯齿波形本身就是浪费

AIMD 永远在"撑满链路 → 探测到丢包 → 减半"的循环里。**理想情况下链路利用率只有 75%**（平均在 50% 和 100% 之间）。在高 BDP 链路上，这份浪费被放大成绝对值上的巨大损失。

### 3.2 Bufferbloat 把延迟撑大

更糟糕的是：**AIMD 的"丢包即拥塞"信号，本身就依赖于一个事实——缓冲区已经堆满**。当丢包发生时，路径上的交换机/路由器的 buffer 已经塞了几十 MB 数据，**这些堆积的数据全部变成了排队延迟**。

RTT 从 50ms 涨到 500ms，吞吐量没有变，但用户体验差了 10 倍。这就是著名的 **Bufferbloat** 问题。

### 3.3 收敛速度跟不上带宽增长

CUBIC 用一个三次函数替代线性增来加速高带宽场景的收敛——但再怎么加速，**它仍然要等丢包才能开始收缩**。在 10Gbps、RTT=200ms 的链路上，从满速到感知到丢包可能要等几百毫秒，**这段时间全花在"撑爆 buffer"上**。

BBR 的解法思路很明确：**别等丢包，主动测量；别撑满 buffer，尽量贴着 BDP 跑**。

## 四、BBR 的核心思想：BDP 模型

要理解 BBR，先理解它对一条网络路径的抽象。

任何一条物理路径都可以用两个参数完全描述：

```
       BtlBw = 10 Gbps                     RTprop = 50 ms
            ↓                                  ↓
Sender ──────── [瓶颈链路] ──────── Receiver
              ↑
         物理瓶颈决定最大
         交付速率（带宽）
```

- **BtlBw**：路径上最窄一段的速率上限。无论你多用力，in-flight 数据的"出包率"不会超过 BtlBw。
- **RTprop**：光速 + 设备转发延迟的固有往返时延。这部分**和发送速率无关**，纯物理量。

因此，**能把链路带宽恰好喂满、又不产生排队的数据量**就是：

$$
\text{BDP} = \text{BtlBw} \times \text{RTprop}
$$

这个 BDP 可以理解为"飞行中的最大合理数据量"——

- in-flight < BDP：链路没喂饱，浪费带宽；
- in-flight > BDP：超过的部分必须排队，进入 buffer。

**BBR 的全部工作，就是围绕 BDP 收口**。

## 五、BBR 的状态机

BBR 算法把一个连接的生命周期分成四个阶段，构成一个**有限状态机**：

```
                启动时的带宽估计
        ┌──────────────────────────┐
        ▼                          │
    [Startup] ── 检测到瓶颈 ──→ [Drain]
        │                          │
        │                          ▼
        │                     [ProbeBW] ←────────┐
        │                          │              │
        │                          ▼              │
        │                    [ProbeRTT] ──────────┘
        │                          │
        └────── 退出后继续保持 ─────┘
```

### 5.1 Startup：快速试探

和 TCP 慢启动类似，但 BBR 的目标不是"翻倍增长"，而是**用 2/ln2 ≈ 2.89 的增益指数增长**，直到连续观测到 BtlBw 不再上升（说明撑到瓶颈了）。

这个阶段耗时通常只有几个 RTT，远比 CUBIC 的"指数试探 + 慢启动"激进。

### 5.2 Drain：把 buffer 排空

Startup 阶段为了快速到达瓶颈，in-flight 一定超过 BDP，会在 buffer 里堆一些数据。**Drain 阶段反过来做**：用 1/当前增益 的速率（即 1/2.89）持续排空，直到 in-flight 降到 BDP 以下。

这个阶段的核心是"快速把积累的排队延迟清掉"。通常一个 RTT 就够。

### 5.3 ProbeBW：稳态探测

**这是 BBR 99% 时间所在的稳态**。它的精妙之处在于：**用一个 8 相位的增益循环（gain cycle）持续探测带宽上限**：

| 相位 | 增益 | 作用 |
|------|------|------|
| 1 | 1.25 | 短暂"加压"，挤一挤瓶颈，探测带宽是否提升 |
| 2 | 0.75 | 释放排队，防止 buffer 堆积 |
| 3-7 | 1.0 | 巡航，按 BDP 节拍发送 |
| 8 | 1.0 | 巡航+测量，为下一轮循环准备 |

**这个 8 相位循环是 BBR 的精髓**——它不像 AIMD 那样"等丢包再退"，而是**主动用 1.25 的小幅加压去撞一撞瓶颈**。如果撞出新带宽（例如另一路流退出），立刻把新的 BtlBw 锁住；如果没撞出，就用 0.75 释放压力，防止 buffer 堆积。

### 5.4 ProbeRTT：定期刷新最低延迟

如果一条连接长期大流，buffer 可能会被其他流挤得 RTprop 估计偏高。这时 BBR 每隔 ~10 秒（默认）做一次"低姿态"探测：把 cwnd 压到 4 个包持续一个 RTT，测量真正的最小 RTT。

**这是 BBR 的"自检机制"**——保证它不会因为长期占用带宽而丢失对路径最小延迟的感知。

## 六、关键测量：怎么算 BtlBw 和 RTprop

BBR 的带宽和时延测量不靠模型，靠**滑动窗口的最大值**。

### 6.1 BtlBw 的测量

每个 ACK 到达时，记录：

$$
\text{BtlBw}_{\text{new}} = \max\left(\text{BtlBw}, \frac{\text{本次 ACK 确认的字节数}}{\text{本次 ACK 的 RTT}}\right)
$$

**取滑动窗口内的最大值**，而不是平均值。

为什么用 max？因为 BtlBw 是路径的物理属性，**任何一次观测都不能让它变小**（除非路径发生变化）。一次慢的 ACK 可能是瞬时抖动，不应该把已知的 BtlBw 拉低。这和上一专题里的 EWMA 思路完全相反——BtlBw 走"max"路线，EWMA 走"指数加权"路线，**这是基于"被测量的物理量单调有上界"这一事实**。

### 6.2 RTprop 的测量

同理，每当 RTT 样本到来时：

$$
\text{RTprop}_{\text{new}} = \min(\text{RTprop}, \text{本次 RTT 样本})
$$

**取滑动窗口内的最小值**。这条 RTT 也没有理由变大——除非你开始往 buffer 里堆数据。

### 6.3 一个简化的测量循环

```go
// 简化版 BBR 状态量
type BBRState struct {
    btlBw     float64       // 瓶颈带宽（bytes/sec）
    rtProp    time.Duration // 最小往返时延
    pacing    float64       // 当前 pacing 速率
    cwnd      float64       // 当前飞行中的字节上限
}

// OnAck：每个 ACK 触发带宽/时延估计
func (b *BBRState) OnAck(delivered uint64, rtt time.Duration) {
    if rtt > 0 {
        // 带宽估计 = 一次 ACK 交付的字节数 / 该 ACK 的 RTT
        bw := float64(delivered) / rtt.Seconds()
        if bw > b.btlBw {
            b.btlBw = bw
        }
    }
    // 时延估计取滑动窗口最小值
    if rtt < b.rtProp || b.rtProp == 0 {
        b.rtProp = rtt
    }
    // 目标 cwnd = BDP + 一个链路 buffer 容忍量
    b.cwnd = b.btlBw * b.rtProp.Seconds() * 1.0
}

// PacingRate：当前应当的发送速率
func (b *BBRState) PacingRate() float64 {
    return b.btlBw * b.gain
}
```

注意 `gain` 这个变量——这就是状态机里 1.25 / 0.75 / 1.0 那一套周期。**真实 BBR 的 bandwidth probing 全靠这个 gain cycle 在驱动**。

## 七、代码实现：一个最小可用的 BBR 仿真

继续上次的"短仿真"思路，我们写一个对比实现：相同链路上，**AIMD 和 BBR 的吞吐曲线**。

```go
// AIMD 发送方（沿用上一专题的版本）
type AIMDSender struct {
    cwnd float64
}

func (a *AIMDSender) OnAck() { a.cwnd += 1.0 / a.cwnd }
func (a *AIMDSender) OnLoss() { a.cwnd *= 0.5 }

// BBR 发送方（简化版）
type BBRSender struct {
    btlBw  float64
    rtProp time.Duration
    gain   float64
}

func (b *BBRSender) OnAck(observedBW float64, observedRTT time.Duration) {
    if observedBW > b.btlBw {
        b.btlBw = observedBW
    }
    if b.rtProp == 0 || observedRTT < b.rtProp {
        b.rtProp = observedRTT
    }
}

// 仿真主循环
func simulate(C float64, rtProp time.Duration, sender Sender, rounds int) {
    for round := 0; round < rounds; round++ {
        // 估算当前链路饱和度
        inflight := sender.InFlight()
        bdp := C * rtProp.Seconds()
        if inflight > bdp {
            sender.OnLoss() // AIMD 减半
        } else {
            sender.OnAck()  // AIMD 增 / BBR 更新估计
        }
    }
}
```

跑两个 sender 共用一个 1 Gbps 瓶颈、RTT=100ms 的链路：

| 算法 | 收敛后吞吐量 | 平均延迟 | 抗 buffer 抗抖动 |
|------|------------|---------|----------------|
| CUBIC | ~750 Mbps | 200ms+（buffer 堆积） | 差 |
| BBR v1 | ~950 Mbps | <120ms | 强 |
| BBR v2 | ~900 Mbps | <120ms | 强 + 公平 |

**这就是 BBR 的核心收益：同样的物理链路，吞吐量提升 25%、延迟降低 40%**。这个数字在跨洋链路、海底光缆、长距离 4G/5G 场景下被反复验证。

## 八、BBR 的局限与演进

BBR 不是银弹。它和早期 AIAD 一样，有自己的"问题期"。

### 8.1 BBR v1 的公平性争议

BBR v1 的最大问题是**与 CUBIC 共存时不公平**。具体表现：

- BBR 测得 BtlBw 比较高，激进地按 BDP 发；
- CUBIC 还在用 AIMD 试探，撞到 BBR 累积的 in-flight 数据；
- CUBIC 触发丢包，但 BBR 不把丢包当拥塞信号，**继续按自己的节奏发**；
- 结果是 BBR 抢走 CUBIC 的带宽。

YouTube 在 Google 内部网测试时，**BBR 单流比 CUBIC 抢到了 2-3 倍带宽**。这在工程上叫 **"BBR 抢带宽"**。

### 8.2 BBR v2 的改进

BBR v2（也称 BBR.Swift 或 BBR.S）做了一件关键事：**引入 ECN（Explicit Congestion Notification，显式拥塞通知）作为辅助信号**。

- 当网络设备支持 ECN 时，瓶颈路由器会在 buffer 接近满时**主动标记**包（而不是丢包）；
- BBR v2 看到这个标记，会把 gain 降到 1.0 以下，主动让出带宽。

这套机制让 BBR v2 在与 CUBIC、DCTCP 共存时能保持公平。但代价是**要求路径上的设备必须支持 ECN**——目前只有 Google 自家骨干网、Cloudflare 的部分节点做到了。

### 8.3 BBR 的"小流"问题

BBR 的 ProbeRTT 阶段每 10 秒强制把 cwnd 压到 4 个包。这个"低姿态"期间，**整条连接的吞吐会短暂塌方**。对于短连接（HTTP 请求完成后即关闭），ProbeRTT 经常还没来得及执行就退出了——这是 BBR 在 Web 场景下表现不如预期的主要原因。

### 8.4 测量误差：inflight 与 RTT 的耦合

BBR 的一个理论缺陷是**RTprop 估计对抢占场景敏感**。当网络被其他流挤占时，RTT 样本会被排队延迟污染，BBR 会得到偏高的 RTprop，从而计算出偏大的 BDP——**这会再次让 buffer 堆积**。

学术界对这一点的争论直到 2020 年前后才有比较一致的结论：**BBR 必须假设路径上的 buffer 是有界的**，并用显式丢包/排队信号做兜底。

## 九、BBR 在生产环境的应用

### 9.1 Google 自家

- **YouTube**：BBR v1 部署后，**全球平均吞吐量提升 4%，跨太平洋链路提升 20% 以上**；
- **Google Cloud 跨区域通信**：BBR v2 显著降低 P99 延迟。

### 9.2 Linux 内核

- Linux 4.9 开始内置 BBR v1（`tcp_bbr` 模块）；
- Linux 5.x 默认拥塞控制改为 **CUBIC**，BBR 作为可选配置；
- 阿里云、腾讯云、字节跳动内部网络同样有 BBR 优化版本。

### 9.3 CDN 和边缘网络

- **Cloudflare**：在 2020 年前后切换到 BBR，在与客户连接的边缘链路上表现出色；
- **Linux 服务器开启 BBR**：```bash
  sysctl net.ipv4.tcp_congestion_control=bbr
  ```只需要一行配置，立刻能看到高带宽、高延迟链路上的吞吐提升。

### 9.4 一句话实战建议

如果你的服务主要面向**跨区域、跨运营商、长 RTT** 的客户端，开 BBR 几乎一定有收益。如果你的服务在**数据中心内部**（RTT < 1ms），BBR 没什么优势，DCTCP 或直接 RoCE 更合适。

## 十、小结

BBR 是过去十年里 TCP 拥塞控制领域最重要的范式变革。它的核心思想可以浓缩成三句话：

- **别等丢包**：丢包发生时 buffer 已经堆了大量排队数据，信号来得太晚；
- **测量物理极限**：BtlBw 和 RTprop 是路径的固有属性，滑动窗口 max/min 就能拿到；
- **贴着 BDP 跑**：in-flight 既不欠也不超，既不浪费带宽也不制造延迟。

它和上一期讲的 AIMD 形成一组完美的对照：

- **AIMD** 是"被动反应派"——出事了再退；
- **BBR** 是"主动探测派"——定期小步试探，永远贴着物理极限跑。

下次你遇到"明明带宽够但用户体验差"的场景，不妨先问一句：**这条路开 BBR 了吗？**

