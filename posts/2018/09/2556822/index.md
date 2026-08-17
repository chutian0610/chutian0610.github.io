# 算法之Manacher算法

给定一个字符串 s，找到 s 中最长的回文子串？

<!--more-->

回文是一个正读和反读都相同的字符串，例如，“aba” 是回文，而 “abc” 不是。由于回文分为偶回文（比如 bccb）和奇回文（比如 bcacb），而在处理奇偶问题上会比较繁琐，所以这里我们使用一个技巧，具体做法是：在字符串首尾，及各字符间各插入一个字符（前提这个字符未出现在串里）。举个例子：s="abbahopxpo"，转换为s_new="#a#b#b#a#h#o#p#x#p#o#"

定义一个辅助数组`int p[]`，其中`p[i]`表示以 i 为中心的最长回文的半径，例如：

|i|0|1|2|3|4|5|6|7|8|9|10|11|12|13|14|15|16|17|18|
|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|
|s\[i\]|#|a|#|b|#|b|#|a|#|h|#|o|#|p|#|x|#|p|#|
|p\[i\]|1|2|1|2|5|2|1|2|1|2|1|2|1|2|1|4|1|2|1|

可以看出，`p[i] - 1`正好是原字符串中最长回文串的长度。**`p[i]`-1**的最大值就是我们的最终结果。

接下来可以得到如下结论:

1. 设置两个变量，mx 和 id 。mx 代表以 id 为中心的最长回文的右边界，也就是mx = id + `p[id]`。
2. 如果mx > i，那么P[i] >= MIN(P[2 * id - i], mx - i)。
3. 对于 mx <= i 的情况，无法对 P[i]做更多的假设，只能P[i] = 1，然后再去匹配了。

对于第二点的分析如下:

当 mx - i > P[i] 的时候（j=2*id-i），以S[i]为中心的回文子串包含在以S[id]为中心的回文子串中，由于 i 和 j 对称，以S[j]为中心的回文子串必然包含在以S[id]为中心的回文子串中，所以必有 P[i] = P[j]，如下。

|i|0|1|2|3|4|5|6|7|8|9|10|11|12|13|14|15|16|17|18|
|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|
|s\[i\]|#|a|#|b|#|b|#|a|#|h|#|o|#|p|#|x|#|p|#|
|p\[i\]|1|2|1|2|5|2|1|2|1|2|1|2|1|2|1|4|1|2|1|
||||||id||||mx|
||||j||||i|

当 P[i] >= mx - i 的时候，以S[i]为中心的回文子串不一定完全包含于以S[id]为中心的回文子串中，但是基于对称性可知，下图中两个绿框所包围的部分是相同的，也就是说以S[i]为中心的回文子串，其向右至少会扩张到mx的位置，也就是说 P[i] >= mx - i。至于mx之后的部分是否对称，就只能老老实实去匹配了。

|i|0|1|2|3|4|5|6|7|8|9|10|11|12|13|14|15|16|17|18|
|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|:---|
|s\[i\]|#|a|#|b|#|b|#|a|#|h|#|o|#|p|#|x|#|p|#|
|p\[i\]|1|2|1|2|5|2|1|2|1|2|1|2|1|2|1|4|1|2|1|
||||||id||||mx|
|||j||||||i|

对于 mx <= i 的情况，无法对 P[i]做更多的假设，只能P[i] = 1，然后再去匹配了。

对于manacher 算法:

* 时间复杂度为 $O(n)$
* 空间复杂度为 $O(n)$

```java
public class Manacher {
    public String longestPalindrome(String s) {
        String newStr = preProcess(s);
        // rad[i]表示以i为中心的回文的最大半径，i至少为1，即该字符本身
        int [] rad = new int[newStr.length()];
        // right表示已知的回文中，最右的边界的坐标
        int right = -1;
        // id表示已知的回文中，拥有最右边界的回文的中点坐标
        int id = -1;
        // 2.计算所有的rad
        // 这个算法是O(n)的，因为right只会随着里层while的迭代而增长，不会减少。
        for (int i = 0; i < newStr.length(); i ++) {
            int r = 1;
            if (i <= right) {
                // 2.1.确定一个最小的半径, 新的点落在已有回文中，
                r = Math.min(rad[id] - (i - id), rad[2 * id - i]);
            }
            // 2.2.尝试更大的半径
            while (i - r >= 0 && i + r < newStr.length() && newStr.charAt(i - r) == newStr.charAt(i + r)) {
                r++;
            }
            // 2.3.更新边界和回文中心坐标
            if (i + r - 1> right) { // i的边界 在边界外
                right = i + r - 1;
                id = i;
            }
            rad[i] = r;
        }
        // 3.扫描一遍rad数组，找出最大的半径
        int maxLength = 0;
        int maxValuePos = 0;
        int Pos = 0;
        for (int r : rad) {
            if (r > maxLength) {
                maxLength = r;
                maxValuePos = Pos;
            }
            Pos++;
        }
        int realLen =  maxLength - 1;
        String huiwen;
        StringBuffer realHuiwen = new StringBuffer();
        if (realLen == 1) {
            return newStr.substring(maxValuePos,maxValuePos+1);
        } else {
            huiwen = newStr.substring((maxValuePos + 1 - rad[maxValuePos]), maxValuePos + rad[maxValuePos]);
            //去掉辅助字符
            for (int j = 0; j < huiwen.length(); j++) {
                if (j % 2 != 0)
                    realHuiwen = realHuiwen.append(huiwen.charAt(j));
            }
            return realHuiwen.toString();
        }
    }

    // 为了避免奇数回文和偶数回文的不同处理问题，在原字符串中插入'#'，将所有回文变成奇数回文
    public String preProcess(String str) {
        StringBuilder newStr = new StringBuilder();
        newStr.append('#');
        for (int i = 0; i < str.length(); i ++) {
            newStr.append(str.charAt(i));
            newStr.append('#');
        }
        return newStr.toString();
    }
} 
```

