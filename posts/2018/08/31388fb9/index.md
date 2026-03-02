# My Effective Java

{{< admonition note "未完待续" true >}}
持续更新中...
{{< /admonition >}}

我的Effective Java开发笔记

<!--more-->

## Regex

### String.replaceAll

replaceAll(regex, replacement)函数，由于第一个参数支持正则表达式，replacement中出现 `$`,会按照`$1`的分组模式进行匹配。当编译器发现`$`后跟的不是整数的时候，就会抛出`Illegal group reference`的异常。所以在进行replace时，需要对替换字符做转义处理。

```java
String re = xxx; //
re = java.util.regex.Matcher.quoteReplacement(re);
```

## MD5

### 相同字符串MD5值不同

在字符串内容不包含中文的情况，可能不会发现MD5结果不一致的问题，因为非中文情况，GBK与UTF-8的MD5结果是一样的，但是如果有中文，MD5的结果就可能不一样。
```java
// 此处必须指定字符集
byte[] bytesOfMessage = tempStr.getBytes("UTF-8"); 
MessageDigest md5 = MessageDigest.getInstance("MD5");
byte[] theDigest = md5.digest(bytesOfMessage);
```

