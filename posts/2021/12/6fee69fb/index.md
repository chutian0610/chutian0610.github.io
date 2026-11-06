# 时间格式字符串-年底的惊喜

这一个常在元旦附近出没的Bug，主要原因是Java 日期格式FormatString 中的yyyy 被写成了YYYY。
要注意的是，对于年份来说，大写的Y和小写的y其意义是不同的。y 是Year,  Y 表示的是Week year

经过试验，得出的结果如下：Week year 意思是当天所在的周属于的年份，一周从周日开始，周六结束，只要本周跨年，那么这周就算入下一年。

> 注意上面的Week year 指format时的结果，对于YYYY格式使用parse, 会得到意想不到的结果。

```java
SimpleDateFormat upperFormater = new SimpleDateFormat("YYYY-MM-dd HH:mm:ss");
SimpleDateFormat lowerFormater = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
System.out.println(lowerFormater.parse("2021-12-30 09:00:00"));
String lower = lowerFormater.format(lowerFormater.parse("2021-12-30 09:00:00"));
System.out.println(lower);
String upper = upperFormater.format(lowerFormater.parse("2021-12-30 09:00:00"));
System.out.println(upper);
System.out.println(upperFormater.parse("2021-12-30 09:00:00"));
```

