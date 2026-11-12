# java:随机数

计算机科学中的随机数分为两种:

- 真随机数: 其生成过程是不可预测、不可重复的，完全基于自然界中物理过程的随机性。
- 伪随机数:  其生成过程是确定性的、可预测的，但通过一个算法使得生成的数字序列看起来像是随机的。
    - 使用一个确定的数学公式。它从一个初始值（称为“种子”）开始，通过算法计算出下一个数，然后用这个数作为新的内部状态，继续计算下一个数，如此反复。

<!--more-->

## Java 随机数

Java 中的随机数有3种使用方式:

- `java.util.Random`: 伪随机数
- `java.util.concurrent.ThreadLocalRandom`: 并发优化的`Random`
- `java.security.SecureRandom`: 真随机数

还有一种常用的方法获取随机数(`Math.random()`)，查看源码可以发现 Math 实际上使用的还是 Random。

```java
public static double random() {
    return RandomNumberGeneratorHolder.randomNumberGenerator.nextDouble();
}
private static final class RandomNumberGeneratorHolder {
    static final Random randomNumberGenerator = new Random();
}
```

## 线性同余伪随机算法

$$
X_{n+1} = (a \times X_n + c) \mod m
$$

其中：

- $X_n$) 是序列中的第 n 个值，称为**当前状态**。
- $X_{n+1}$ 是序列中的第 n+1 个值，即**下一个随机数**。
- a 是**乘数**，一个非零整数。
- c 是**增量**。
- m 是**模数**，决定了随机数的范围（0 到 m-1）。

参数a, c, m以及初始值 $X_0$（称为**种子**）共同决定了整个随机数序列。

线性同余生成器的质量几乎完全取决于参数 a, c, m的选择。糟糕的参数会导致非常差的随机性，例如：

- **周期极短**：序列很快就开始重复。
- **相关性高**：生成的数字之间有明显的模式。

为了获得尽可能长的周期(最长可达m)和良好的统计特性，参数通常需要满足以下条件(当 $c \neq 0$ 时)：
1.  c 和 m 互质(最大公约数为 1)。
2.  a - 1 可以被 m 的所有质因数整除。
3.  如果 m 是 4 的倍数，那么 a - 1 也必须是 4 的倍数。

历史上一个著名的、曾被广泛使用的例子是 **C语言的 `rand()`** 实现（在某些系统中）：
*  a = 1103515245
*  c = 12345
*  m = $2^{31}$

## Random

Random正是使用线性同余算法。

```java
public class Random implements java.io.Serializable {
    ... ...
    /**
     * The internal state associated with this pseudorandom number generator.
     * (The specs for the methods in this class describe the ongoing
     * computation of this value.)
     */
    private final AtomicLong seed;

    private static final long multiplier = 0x5DEECE66DL;
    private static final long addend = 0xBL;
    private static final long mask = (1L << 48) - 1;
    ...
    protected int next(int bits) {
        long oldseed, nextseed;
        AtomicLong seed = this.seed;
        do {
            oldseed = seed.get();
            // 线性同余算法
            nextseed = (oldseed * multiplier + addend) & mask;
        } while (!seed.compareAndSet(oldseed, nextseed));
        return (int)(nextseed >>> (48 - bits));
    }
    ...
}
```

我们可以通过下面的代码 check(种子相同，结果一定相同):

```java
long seed  = 123;
System.out.println(new Random(seed).nextInt());
System.out.println(new Random(seed).nextInt());
```

## SecureRandom

无论使用何种随机种子、进行何种精心计算和保护，都存在可能被推测出的风险。Java提供了 SecureRandom 随机数生成类，可以安全的生成真随机数。

- SecureRandom最多可生成128位随机值，提供了更高的安全性。
- SecureRandom 不使用固定种子值。而是从操作系统/dev/random 随机数文件中不断获取新的种子值。

> SecureRandom并不是线程安全的，可以使用synchronized关键字同步，或者 使用ThreadLocal 为每个线程保存一个 SecureRandom实例。

但是，通常场景SecureRandom算法的耗时是Random的10倍以上。所以在实际使用中，应该仅在安全性较高的业务场景生成随机数时，使用 SecureRandom。

### SecureRandom 几种策略

- SHA1PRNG: SHA1PRNG是一种伪随机数生成器算法，在Java SecureRandom中，它被作为Windows下默认的随机数生成算法。该算法基于SHA-1算法，但通过添加额外的步骤来提高随机性。

- NativePRNGBlocking: Java SecureRandom中，NativePRNG 算法是Linux 下默认的随机数生成算法。NativePRNGBlocking 初始播种时使用`/dev/random`中的 20 个字节初始化内部 SHA1PRNG 实例，当调用 nextBytes()、nextInt() 等：使用内部 SHA1PRNG 实例的输出和从`/dev/random`读取的数据的进行异或。
NativePRNGBlocking每次计算随机数需要从`/dev/random`文件中获取数值。当`/dev/random`的随机数不足时，NativePRNGBlocking将会被阻塞。在桌面应用程序中，`/dev/random`文件很少会受到阻塞，因为它可以收集用户的鼠标、点击等事件。然而，在Web程序中，由于并发度较高，生成`/dev/random`数据可能会出现不足的情况。
- NativePRNGNonBlocking: 为了避免获取随机数被阻塞，NativePRNGNonBlocking选择从`/dev/urandom中获取随机数`。`/dev/urandom` 和 `/dev/random` 之间有一些区别。

| | `/dev/random` | `/dev/urandom` |
| :--- | :--- | :--- |
| **随机性来源** | 系统熵池（环境噪音） | 初始基于熵池，不足时使用密码学安全的PRNG扩展 |
| **阻塞行为** | 是，当熵不足时会阻塞 | 否，永不阻塞 |
| **随机数质量** | 理论上随机性更高，适合极少数长期密钥生成 | **密码学安全**，适合绝大多数场景 |
| **性能速度** | 慢 | 快 |
| **主要应用场景** | 生成极高强度、长寿命的密钥（如GPG/SSL/SSH根密钥） | **会话密钥、应用程序随机数、模拟测试、随机密码等绝大多数需求** |

> 在Linux中，无论是 /dev/random 还是 /dev/urandom，它们都共同使用一个被称为熵池 的随机源。你可以把它想象成一个由系统各种不可预测事件积累起来的"随机性仓库"。这些事件包括键盘敲击的时间间隔、鼠标移动的轨迹、硬盘读写的精确时刻，甚至包括网络数据包到达的纳秒级时间差。

## ThreadLocalRandom

在 Random 中的源码中可以看到，种子值是存储在原子Long 中。在高并发场景，有可能会出现CAS操作的大量竞争失败自旋重试。在多线程中更建议使用`ThreadLocalRandom`。

ThreadLocalRandom 在 Thread 中增加了 3 个参数。

```java
// java.lang.Thread

/** The current seed for a ThreadLocalRandom */
@sun.misc.Contended("tlr")
long threadLocalRandomSeed;

/** Probe hash value; nonzero if threadLocalRandomSeed initialized */
@sun.misc.Contended("tlr")
int threadLocalRandomProbe;

/** Secondary seed isolated from public ThreadLocalRandom sequence */
@sun.misc.Contended("tlr")
int threadLocalRandomSecondarySeed;
```

在 ThreadLocalRandom 使用时，会检查 Thread中属性有没有初始化，如果没有，会使用`localInit()`初始化 Thread 中的属性。

```java
// init seed
private static long initialSeed() {
    String sec = VM.getSavedProperty("java.util.secureRandomSeed");
    if (Boolean.parseBoolean(sec)) {
        byte[] seedBytes = java.security.SecureRandom.getSeed(8);
        long s = (long)(seedBytes[0]) & 0xffL;
        for (int i = 1; i < 8; ++i)
            s = (s << 8) | ((long)(seedBytes[i]) & 0xffL);
        return s;
    }
    return (mix64(System.currentTimeMillis()) ^
            mix64(System.nanoTime()));
}
static final void localInit() {
    int p = probeGenerator.addAndGet(PROBE_INCREMENT);
    int probe = (p == 0) ? 1 : p; // skip 0
    long seed = mix64(seeder.getAndAdd(SEEDER_INCREMENT));
    Thread t = Thread.currentThread();
    UNSAFE.putLong(t, SEED, seed);
    UNSAFE.putInt(t, PROBE, probe);
}
```

用于在每个线程中存储种子。每次调用后，都会给种子增加一个固定值。
```java
final long nextSeed() {
    Thread t; long r; // read and update per-thread seed
    UNSAFE.putLong(t = Thread.currentThread(), SEED,
                    r = UNSAFE.getLong(t, SEED) + GAMMA);
    return r;
}
private static int mix32(long z) {
    z = (z ^ (z >>> 33)) * 0xff51afd7ed558ccdL;
    return (int)(((z ^ (z >>> 33)) * 0xc4ceb9fe1a85ec53L) >>> 32);
}
   public int nextInt() {
        return mix32(nextSeed());
    }
```

