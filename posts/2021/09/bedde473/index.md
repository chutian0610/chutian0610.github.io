# MurmurHash

MurmurHash 是一种非加密hash功能，适用于基于hash的一般查找。由Austin Appleby在2008年发明， 并出现了多个变种，目前托管在[github](https://github.com/aappleby/smhasher)。该名称来自两个基本操作，乘 multiply 和 旋转 rotate(该算法实际上使用shift和xor而不是rotate)，用于其内循环。与其它流行的哈希函数相比，对于规律性较强的key，MurmurHash的随机分布特征表现更良好。

Redis在实现字典时用到了两种不同的哈希算法，MurmurHash便是其中一种（另一种是djb），在Redis中应用十分广泛，包括数据库、集群、哈希键、阻塞操作等功能都用到了这个算法。该算法最新版本是MurmurHash3，基于MurmurHash2改进了一些小瑕疵，使得速度更快，实现了32位（低延时）、128位HashKey，尤其对大块的数据，具有较高的平衡性与低碰撞率。

<!--more-->
## 算法伪码

下面是MurmurHash的伪码:

```python
algorithm Murmur3_32 is
    // Note: In this version, all arithmetic is performed with unsigned 32-bit integers.
    //       In the case of overflow, the result is reduced modulo 232.
    input: key, len, seed

    c1 ← 0xcc9e2d51
    c2 ← 0x1b873593
    r1 ← 15
    r2 ← 13
    m ← 5
    n ← 0xe6546b64

    hash ← seed

    // 按照 chunk(4byte)读取数据
    for each fourByteChunk of key do
        k ← fourByteChunk

        k ← k × c1
        k ← k ROL r1 // 这里的 ROL 指循环左移(汇编指令)，将输入的无符号数循环左移N位
        k ← k × c2

        hash ← hash XOR k
        hash ← hash ROL r2
        hash ← (hash × m) + n

    with any remainingBytesInKey do
        remainingBytes ← SwapToLittleEndian(remainingBytesInKey)
        // Note: Endian swapping is only necessary on big-endian machines.
        //       The purpose is to place the meaningful digits towards the low end of the value,
        //       so that these digits have the greatest potential to affect the low range digits
        //       in the subsequent multiplication.  Consider that locating the meaningful digits
        //       in the high range would produce a greater effect upon the high digits of the
        //       multiplication, and notably, that such high digits are likely to be discarded
        //       by the modulo arithmetic under overflow.  We don't want that.

        remainingBytes ← remainingBytes × c1
        remainingBytes ← remainingBytes ROL r1
        remainingBytes ← remainingBytes × c2

        hash ← hash XOR remainingBytes

    hash ← hash XOR len

    hash ← hash XOR (hash >> 16)
    hash ← hash × 0x85ebca6b
    hash ← hash XOR (hash >> 13)
    hash ← hash × 0xc2b2ae35
    hash ← hash XOR (hash >> 16)
```

## 代码实现

```java
public class MurmurHash {

    /** 32bits 的长度是 4Byte */
    private static final int CHUNK_SIZE = 4;

    // 使用模拟退火算法求出了最合适的参数
    private static final int C1 = 0xcc9e2d51;
    private static final int C2 = 0x1b873593;

    /**
     * JVM采用大端方式存多字节的数据,获取对应的小端存储值.
     * @param input 字节数组
     * @param offset 偏移量起点
     * @return
     */
    private static int getIntLittleEndian(byte[] input, int offset) {
        return input[offset + 3]& 0xff << 24
        | input[offset + 2]& 0xff << 16
        | input[offset + 1] & 0xff << 8
        |input[offset] & 0xff;
    }

    /**
     * K mix
     * @param k
     * @return
     */
    private static int mixK(int k) {
        k *= C1;
        k = Integer.rotateLeft(k, 15);
        k *= C2;
        return k;
    }

    /**
     * h mix
     * @param h
     * @param k
     * @return
     */
    private static int mixHash(int h, int k) {
        h ^= k;
        h = Integer.rotateLeft(h, 13);
        h = h * 5 + 0xe6546b64;
        return h;
    }

    /**
     * byte 转 Int
     * @param value
     * @return
     */
    private static int toInt(byte value){
        return value & 0xff;
    }

    // Finalization mix - force all bits of a hash block to avalanche
    private static int fmix(int hash, int length) {
        hash ^= length;
        hash ^= hash >>> 16;
        hash *= 0x85ebca6b;
        hash ^= hash >>> 13;
        hash *= 0xc2b2ae35;
        hash ^= hash >>> 16;
        return hash;
    }
    // --------------------------- 入口函数 ---------------------------
    /**
     * Murmur3_32的Hash求值函数
     * @param seed seed 种子（无符号数）
     * @param key 待求hash的key值（二进制数组）
     * @param offset start offset of the array to hash
     * @param len length length of the array to hash
     * @return
     */
    public static int murmurhash3_x86_32(byte[] key,int offset, int len,int seed){
        // 使用种子作为初始值
        int hash= seed;
        int i;
        // 每次读4个byte
        for (i = 0; i + CHUNK_SIZE <= len; i += CHUNK_SIZE) {
            int k= mixK(getIntLittleEndian(key,offset+i));
            hash = mixHash(hash, k);
        }
        // 当没有读净buffer时，处理尾部
        int k1 = 0;
        for (int shift = 0; i < len; i++, shift += 8) {
            k1 ^= toInt(key[offset + i]) << shift;
        }
        hash ^= mixK(k1);
        return fmix(hash, len);
    }
    // --------------------------- 包装方法 ---------------------------
    /**
     * 字符串基于Murmur3_32的Hash求值函数
     * @param seed 种子（无符号数）
     * @param key 待求hash的key值
     * @param charset key的字符集
     * @return 32位Hash值
     */
    public static int murmurhash3_x86_32_Str(int seed, String key, Charset charset){
        byte[] data = key.getBytes(charset);
        return murmurhash3_x86_32(data,0,data.length,seed);
    }
}
```

[github Demo 地址](https://github.com/chutian0610/java-algorithms-impl/blob/master/src/main/java/info/victorchu/algorithms/hash/MurmurHash.java)

## 生产使用

在生产环境中使用 MurmurHash 可以通过Apache Commons Codec库。

```xml
<dependency>
    <groupId>commons-codec</groupId>
    <artifactId>commons-codec</artifactId>
</dependency>
```

## 参考

- [1] [MurmurHash - Wikipedia](https://en.wikipedia.org/wiki/MurmurHash)
- [2] [guava Murmur3_32HashFunction](https://github.com/google/guava/blob/master/guava/src/com/google/common/hash/Murmur3_32HashFunction.java)
- [3] [guava Murmur3_128HashFunction](https://github.com/google/guava/blob/master/guava/src/com/google/common/hash/Murmur3_128HashFunction.java)
- [4] [MurmurHash3 cpp](https://github.com/aappleby/smhasher/blob/master/src/MurmurHash3.cpp)

