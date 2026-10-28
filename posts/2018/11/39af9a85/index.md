# 算法之KMP字符串匹配

有一个文本串S，和一个模式串P，现在要查找P在S中的位置，怎么查找呢？

假设主串target为: `a b a c a a b a c a b a c a b a a b b`,模式串pattern: `a b a c a b`\(为了方便查看，每个字符间用空格隔开\)。

用暴力算法匹配字符串过程中，我们会把`target[0]`跟 `pattern[0]` 匹配，如果相同则匹配下一个字符，直到出现不相同的情况，此时我们会丢弃前面的匹配信息，然后把`target[1]`跟 `pattern[0]` 匹配，循环进行，直到主串结束，或者出现匹配成功的情况。这种丢弃前面的匹配信息的方法，极大地降低了匹配效率。

以上面的字符为例子:pattern的前5个字符`abaca`可以匹配target的前5个字符,但是`pattern[5]`和`target[5]`不匹配。下面重新从target[1]开始和pattern匹配。

显然效率很差，因为你要把"搜索位置"移到已经比较过的位置，重比一遍。

<!--more-->

## KMP

KMP算法是一种改进的字符串匹配算法，它的关键是利用匹配失败后的信息，尽量减少模式串与主串的匹配次数以达到快速匹配的目的。在KMP算法中，对于每一个模式串我们会事先计算出模式串的内部匹配信息，在匹配失败时最大的移动模式串，以减少匹配次数。

### 部分匹配表

正确的前缀：字符串中的所有字符，其中一个或多个字符被截断。“S”，“Sn”，“Sna”和“Snap”都是“Snape”的正确前缀。  
正确的后缀：字符串中的所有字符，一个或多个字符在开头处截断。“agrid”，“grid”，“rid”，“id”和“d”都是“Hagrid”的正确后缀。

"部分匹配值"就是"前缀"和"后缀"的最长的共有元素的长度。以"ABCDABD"为例，

1. "A"的前缀和后缀都为空集，共有元素的长度为0；
2. "AB"的前缀为`[A]`，后缀为`[B]`，共有元素的长度为0；
3. "ABC"的前缀为`[A, AB]`，后缀为`[BC, C]`，共有元素的长度0；
4. "ABCD"的前缀为`[A, AB, ABC]`，后缀为`[BCD, CD, D]`，共有元素的长度为0；
5. "ABCDA"的前缀为`[A, AB, ABC, ABCD]`，后缀为`[BCDA, CDA, DA, A]`，共有元素为"A"，长度为1；
6. "ABCDAB"的前缀为`[A, AB, ABC, ABCD, ABCDA]`，后缀为`[BCDAB, CDAB, DAB, AB, B]`，共有元素为"AB"，长度为2；
7. "ABCDABD"的前缀为`[A, AB, ABC, ABCD, ABCDA, ABCDAB]`，后缀为`[BCDABD, CDABD, DABD, ABD, BD, D]`，共有元素的长度为0。

KMP的关键是部分匹配表。这是模式`abababca`的部分匹配表：

```sh
## next[i] 是指 当前子串的相同前缀后缀的最大长度
char:  | a | b | a | b | a | b | c | a |
index: | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
next: | 0 | 0 | 1 | 2 | 3 | 4 | 0 | 1 |
```

以index =3 为例，3的子字符串是`abab`，前缀为`[A, AB, ABA]`，后缀为`[BAB, AB,B]`最长的共有元素长度是2(`AB`).

也意味着在第 j个字符失配时，对应的 $next[j-1]$ 值会告诉你下一步匹配中，模式串应该跳到哪个位置（跳到 $next[j-1]$ 的位置）。如果 $next[j-1]$ 等于0或-1，则跳到模式串的开头字符，若 $next[j-1]= k$ 且 $ k > 0$，代表下次匹配跳到j之前的第 K个字符，而不是跳到开头。

### 如何使用部分匹配表

假设我们将“abababca”模式与“bacbababaabcbab”相匹配。这里是我们的部分匹配表:

```sh
char:  | a | b | a | b | a | b | c | a |
index: | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
next: | 0 | 0 | 1 | 2 | 3 | 4 | 0 | 1 |
```


我们第一次获得部分匹配是在这里：

```sh
## pattern[0]匹配a成功，pattern[1]未匹配c成功
bacbababaabcbab
 |?
 abababca
```

此时部分匹配的长度`index`为1，`next[index-1]`的值为0，因此我们不会先跳过任何一个。我们得到的下一个部分匹配是：

```sh
## pattern[5]未匹配a成功
bacbababaabcbab
    |||||?
    abababca
```

这时`index`为5. `next[index-1]`的值为3.这意味着对当前字符匹配 `pattern[3]`

```sh
## x denotes a skip
bacbababaabcbab
    xx|||?
      abababca
```

此时继续匹配失败，`index`=3，`next[index-1]`的值是1.这意味着对当前字符匹配 `pattern[1]`.

```sh
// x denotes a skip

bacbababaabcbab
      xx|?
        abababca
```

此时，我们的模式比文本中的其余字符长，所以我们知道没有匹配。

## 如何计算部分匹配表

从上面的详细描述，我们可以得到算法逻辑的重点在于获取部分匹配表。

### 一个朴素的实现是:

- 在一个循环中以 $i = 1\to n - 1$ 的顺序计算前缀函数 $next[i]$ 的值($next[0]$ 被赋值为 0)对于每个i, $j = i\to 0$ 的顺序计算前缀函数
    - 当前长度下前缀和后缀相等，则此时 $next[i]=j$，否则令j 自减 1，继续匹配，直到 $j=0$ 。
    - 如果 j = 0 并且仍没有任何一次匹配，则置 $next[i] = 0$ 并移至下一个下标 
$i + 1$ 。 

```java
static int[] prefix_function(String s) {
    int n = s.length();
    int[] next = new int[n];
    for (int i = 1; i < n; i++) {
        for (int j = i; j >= 0; j--) {
            if (s.substring(0, j).equals(s.substring(i - j + 1, i + 1))) {
                next[i] = j;
                break;
            }
        }
    }
    return next;
}
```
该算法的时间复杂度为 O(n^2)，具有很大的改进空间。

### 优化方案

当i移动到下一个位置时，前缀函数 $next[i+1]$ 的值要么增加一，要么维持不变，要么减少。

$$
c_0 \space c_1 \space c_2 \space c_3 \space ... \space c_{i-2} \space c_{i-1} \space c_i \space c_{i+1}
$$

假设 $next[i] = 2$, 即 $c_0 \to c_1 = c_{i-1} \to c_{i}$。此时比较 $c_2$ 和 $c_{i+1}$。

- 如相等，则 $next[i+1] = 3$
- 如果不相等，则 $next[i+1]<=2$

```java
static int[] prefix_function(String s) {
    int n = s.length();
    int[] next= new int[n];
    for (int i = 1; i < n; i++) {
        for (int j = next[i - 1] + 1; j >= 0; j--) {
            // 注意，j 是从前一个字符的 next 前缀函数判断的
            if (s.substring(0, j).equals(s.substring(i - j + 1, i + 1))) {
                next[i] = j;
                break;
            }
        }
    }
    return pi;
}
```

经过此次优化，计算前缀函数只需要进行 $O(n)$ 次字符串比较.


当然我们可以接着上面的优化思路总结出状态转移方程。并进一步优化。

### 最终计算方法

* 已知`next[i] = j,next[0] = 0`,如何求出`next[i + 1]`
  * 如果`pattern[j]==pattern[i+1]`,则`next[i+1] = next[i]+1=j+1`;
  * 如果`pattern[i+1] != pattern[j]`,
    * 令`j=next[j-1]`,如果此时`pattern[j]==pattern[i+1]`,则`next[i+1] = j+1`;
    * 如果仍不相等,继续递归前缀索引,令`j=next[j-1]`,继续判断`pattern[j]==pattern[i+1]`，直到j<=0，或`pattern[j]=pattern[i+1]`为止。

注意，`j=next[j-1]`很巧妙，利用了之前的`next[?]`。

```java
// 从模式中提取 next 数组
static int[] prefix_function(String s) {
    int n = s.length();
    int[] next = new int[n];
    for (int i = 1; i < n; i++) {
        int j = next[i - 1];
        while (j > 0 && s.charAt(i) != s.charAt(j)) {
            j = next[j - 1];
        }
        if (s.charAt(i) == s.charAt(j)) {
            j++;
        }
        next[i] = j;
    }
    return next;
}
```

## 基于部分匹配表的 IndexOf 方法

java 代码实现见[github](https://github.com/chutian0610/java-algorithms-impl/blob/master/src/main/java/info/victorchu/algorithms/str/match/KMPMatcher.java)

``` java
public int firstIndexOf(String target,String pattern){
    int i=0;int j=0;
    int tLength = target.length();
    int pLength = pattern.length();
    if(tLength < pLength){
        return -1;
    }
    int[] next = new int[pattern.length()];
    getNextArrayV2(pattern,next);

    while (i<tLength && j<pLength){
        if ( target.charAt(i) == pattern.charAt(j)) {
            //如果当前字符匹配成功（即target[i] == pattern[j]），令i++，j++
            i++;
            j++;
        } else {
            if(j>0) {
                //j>0且如果当前字符匹配失败，则令 i 不变，j = next[j-1]
                //next[j-1]即为j所对应的next值
                j = next[j - 1];
            }else {
                // 如果当前字符匹配失败，且j=0，则i++，j不变
                i++;
            }
        }
    }
    if (j == pLength) {
        // 匹配成功
        return i - j;
    }
    else {
        return -1;
    }
}
```

如果某个字符匹配成功，模式串首字符的位置保持不动，仅仅是i++、j++；如果匹配失配，i 不变（即 i 不回溯），模式串会跳过匹配过的`next[j-1]`个字符。整个算法最坏的情况是，当模式串首字符位于i - j的位置时才匹配成功，算法结束。所以，如果文本串的长度为n，模式串的长度为m，那么匹配过程的时间复杂度为 $O(n)$，算上计算next的 $O(m)$ 时间，KMP的整体时间复杂度为 $O(m + n)$。

