# Java位运算符

Java中的位运算符包含以下几种:
- `<<`(左移),`>>`(右移)和 `>>>`(无符号右移)属于移位运算符。
- `&`(位与),`|`(位或),`^`(位异或)和`~`(取反)属于逻辑运算符。

<!--more-->

## 移位运算

移位运算符的左侧操作数是要移位的值; 右侧操作数指定移位距离。下面是语法描述:

```java
ShiftExpression：
    AdditiveExpression
    ShiftExpression  << AdditiveExpression
    ShiftExpression  >> AdditiveExpression
    ShiftExpression  >>> AdditiveExpression
```

* `n << s`的意思是n左移s个bit位，这相当于 $n*2^s$,注意溢出。
* `n >> s`的意思是n带符号位右移s个bit位，若左操作数是正数，则高位补“0”，若左操作数是负数，则高位补“1”，这相当于 $n/2^s$。
* `n >>>s`的意思是不带符号右移，无论左操作数是正数还是负数，在高位都补“0”。
    * 如果n是正数，那么结果等同于`n >> s`
    * 如果n是负数，且类型是int，那么结果等同于`(n >> s) + (2 << ~s)`
    * 如果n是负数，且类型是long，那么结果等同`(n >> s) + (2L << ~s)`
    
> `~`是按位补码运算符


shift表达式的类型是左侧操作数的提升类型。

* 如果左手操作数的提升类型是int，则只使用右手操作数的五个最低位作为移位距离.
* 如果左侧操作数的提升类型是long，则只使用右侧操作数的六个最低位作为移位距离.

```java
// 在 Java 中对 5 右移 32 位，还是得到 5.
5 >> 32 == 5

//因为32的二进制表示是100000，后5位是 00000。所以实际上不会移位
```

## 逻辑位运算技巧

### 异或消除

$$
a=0 \oplus a= a \oplus 0 \newline
0=a \oplus a
$$
由上面两个推导出：

$$
a=a \oplus b \oplus b
$$

### 交换两个数

$$
a=a \oplus b \newline
b=a \oplus b \newline
a=a \oplus b \newline
$$

### 移除二进制最后一个1

$$
a=n \land (n-1)
$$

### 获取二进制最后一个1

$$
diff=(n \land (n-1)) \oplus n
$$

### 复合选项

在实际开发中常会用到选项，选项开启和关闭可以有效的控制代码执行逻辑。

对于选项，我们可以简单的用类的一个属性去判断。也可以使用一些编程技巧，通过一个数字来表示组合选项。


一个简单的例子如下:

```java
public class ByteOption {

    public enum Option {
        /*
          option
         */
        OPTION_1(op1),
        OPTION_2(op2),
        OPTION_3(op3),
        OPTION_4(op4),
        OPTION_5(op5),
        OPTION_6(op6),
        OPTION_7(op7)
        ;

        byte value;

        Option(byte i) {
            value = i;
        }
    }
    // 将多个属性设置为不冲突的二进制数
    public static byte op1 =0b00000001;
    public static byte op2 =0b00000010;
    public static byte op3 =0b00000100;
    public static byte op4 =0b00001000;
    public static byte op5 =0b00010000;
    public static byte op6 =0b00100000;
    public static byte op7 =0b01000000;
    private static int ALL = 0b11111111;


    private byte op = 0b00000000;

    public void enableOption(Option option){
        op = (byte) (op | option.value);
    }

    public void disableOption(Option option){
        op = (byte) (op & (option.value^ALL));
    }

    public boolean isEnable(Option option){
        return option.value == (op & option.value);
    }
}
```

## 参考

- [1]  [Java SE语言规范](https://docs.oracle.com/javase/specs/jls/se7/html/jls-15.html#jls-15.22.1)
- [2]  [Java right shift integer by 32](https://stackoverflow.com/questions/32648097/java-right-shift-integer-by-32)

