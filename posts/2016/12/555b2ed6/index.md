# Java quickstart

在开始学习java语言之前,先了解Java的一些特性:

* 简单：语言简单，方便掌握语言特性，快速展开工作。
* 面向对象： 面向对象的编程模型。
* 分布式：支持分布式网络环境运行。
* 健壮：java的目的是创建可靠的软件，提供了编译时检查，运行时检查。通过GC简化开发自己对内存的管理。
* 安全：通过在语言和运行时系统中设计的安全功能，Java技术使您可以构建无法从外部入侵的应用程序。
* 体系结构中立，可移植：Java技术旨在支持将部署到异构网络环境中的应用程序。在这样的环境中，应用程序必须能够在各种硬件体系结构上执行。字节码是一种中立的体系结构，是将代码有效地传输到多个硬件和软件平台的中间格式。此外Java通过严格定义基本语言，将可移植性进一步提高了。
* 解释型：解释和运行系统已经被移植在任何机器上，Java解释器可以直接执行Java字节码。
* 高性能：Java解释器可以全速运行而无需检查运行时环境，从而实现了卓越的性能。自动垃圾收集器运行作为一个低优先级后台线程，确保大概率下的更好的性能和内存可用。
* 多线程：Java多线程功能提供了构建具有许多并发活动线程的应用程序的方法。因此，多线程为最终用户带来了高度的交互性。Java平台通过添加复杂的同步原语来支持语言级别的多线程：语言库提供了 Thread类，运行时系统提供了监视和条件锁定原语。此外还有高层级的库api提供同步功能
* 动态：Java编译器在编译时静态检查方面非常严格，而语言和运行时系统在其链接阶段是 动态的。类仅在需要时链接。新的代码模块可以按需链接到各种来源，甚至可以来自网络上的各种来源。

> James Gosling和Henry McGilton撰写的白皮书 [The Java Language Environment](https://www.oracle.com/java/technologies/language-environment.html)中解释了上述每个流行词 。

Java开发过程: 

1. 首先将所有源代码编写为以.java扩展名结尾的纯文本文件。
2. 然后，将这些源文件由javac编译器编译为.class文件。
3. .class文件中是Java虚拟机的机器语言(字节码)。然后，Java虚拟机实例运行java应用程序。

> Java平台包含两个组件：Java虚拟机和Java应用程序编程接口（API）.虚拟机用于执行java程序，sdk 提供了系统库和部分高级API库。

<!--more-->

## 面向对象

在了解Java的基础语法前，有必要先来了解下面向对象的编程理念。

对象是理解面向对象技术的关键。现实世界中的对象有两个共同的特征: 它们都有状态和行为。例如狗有状态（名字，颜色，品种，是否饥饿）和行为（吠叫，抓取，摇尾巴）。识别真实世界对象的状态和行为是从面向对象编程的角度思考的好方法。软件对象在概念上类似于现实世界的对象：它们也由状态和相关行为组成。对象将其状态存储在字段中（某些编程语言中的变量），并通过方法（某些编程语言中的函数）公开其行为。方法对对象的内部状态进行操作，并作为对象到对象通信的主要机制。

> 隐藏内部状态并要求所有交互都通过对象的方法执行，这被称为数据封装 —— 这是面向对象编程的基本原则。

将代码映射到单个软件对象中提供了许多好处:

- 模块化：对象的源代码可以独立于其他对象的源代码编写和维护。创建后，对象可以很容易地在系统内部传递。
- 信息隐藏：通过仅与对象的方法交互，其内部实现的细节对外部世界保持隐藏。
- 代码重用：如果某个对象已经存在（可能由其他软件开发人员编写），则可以在程序中使用该对象。
- 可插拔性和调试的易用性：如果某个特定对象出现问题，您只需将其从应用程序中删除，然后插入其他对象作为其替换对象即可。

### 类

在现实世界中，可以发现很多相同的物体。例如，每辆同型号的自行车都是根据同一组蓝图构建的，因此包含相同的组件。在面向对象的术语中，我们说某个自行车是被称为自行车的类的实例。类是创建对象实例的蓝图。

```java
// 自行车类
class Bicycle {
  int speed = 0;

  void speedUp(int increment) {
    speed = speed + increment;   
  }

  void applyBrakes(int decrement) {
    speed = speed - decrement;
  }
}
```

不同种类的物体通常彼此之间有一定数量的共同点。例如，山地自行车，公路自行车和双人自行车都具有自行车的特征。同时，每个种类的物体也有它们与众不同的附加功能，例如，双人自行车有两个座位和两组车把。面向对象的编程允许类从其他类继承常用的状态和行为，被继承的类也被称为继承类的超类，继承类是被继承类的子类。在Java编程语言中，每个类允许有一个直接超类，每个超类都可以有无限数量的子类。

创建子类的语法是在类声明的开头，使用extends关键字，extends后跟要继承的类的名称：

```java
class MountainBike extends Bicycle {
}
```

继承允许子类代码专门关注其独特的功能。这使得子类的代码易于阅读。

### 接口

对象通过公开定义的方法和外部世界交互，方法形成对象与外部世界的接口。在最常见的形式中，接口是一组具有空体的相关方法。如果将自行车的行为指定为接口，则可能如下所示：

```java
interface Bicycle {

    void speedUp(int increment);

    void applyBrakes(int decrement);
}
```

实现这个接口可以用implements关键字:

```java
class ACMEBicycle implements Bicycle {

    int speed = 0;
    void speedUp(int increment) {
         speed = speed + increment;   
    }

    void applyBrakes(int decrement) {
         speed = speed - decrement;
    }
}
```

实现接口意味着在类和外部世界之间形成一个**契约**。这个契约在构建时由编译器强制执行。如果您的类声称要实现某个接口，则该接口定义的所有抽象方法都必须出现在其源代码中，然后该类才能成功编译。

## hello world

```java
public class HelloWorld { // 类定义
    /* 
     * 这是一个注释
     */
    public static void main(String[] args)  // main方法定义
    {    
      System.out.println("Hello World"); // 输出 Hello World
    }
}
```
上面的应用程序由三个主要部分组成：源代码注释，HelloWorld类的定义，以及main方法。

## 注释

注释会被编译器忽略，而不会被编译到字节码中，注释的主要作用是帮助程序员。Java 语言支持 三种类型的注释:

- `// comment` : 单行注释，编译器将忽略从`//`开始到行尾的所有内容。
- `/* comment */`: 多行注释， 编译器将忽略从`/*`开始到`*/`的所有内容。
- `/** comment */`: 文档注释， 编译器将忽略从`/**`开始到`*/`的所有内容，此外，文档注释还支持了一些文档注释语法[<sup>2</sup>](#refer-anchor-2)，可以用于自动生成Java文档。

## 标识符

Java 所有的组成部分都需要名字。类名、变量名以及方法名都被称为标识符。关于 Java 标识符，有以下几点需要注意：

- 所有的标识符都应该以字母（A-Z 或者 a-z）,美元符（$）、或者下划线（_）开始
- 首字符之后可以是字母（A-Z 或者 a-z）,美元符（$）、下划线（_）或数字的任何字符组合
- 关键字不能用作标识符
- 标识符是大小写敏感的
- 合法标识符举例：age、$salary、_value、__1_value
- 非法标识符举例：123abc、-salary

### java 关键字

|类别|关键字|说明|
|:---|:---|:---|
|访问控制|private|私有的|
|访问控制|protected|受保护的|
|访问控制|public|公共的|
|访问控制|default|默认|
|类、方法和变量修饰符|abstract|声明抽象|
|类、方法和变量修饰符|class|类|
|类、方法和变量修饰符|extends|扩充,继承|
|类、方法和变量修饰符|final|最终值,不可改变的|
|类、方法和变量修饰符|implements|实现（接口）|
|类、方法和变量修饰符|interface|接口|
|类、方法和变量修饰符|native|本地，原生方法（非 Java 实现）|
|类、方法和变量修饰符|new|新,创建|
|类、方法和变量修饰符|static|静态|
|类、方法和变量修饰符|strictfp|严格,精准|
|类、方法和变量修饰符|synchronized|线程,同步|
|类、方法和变量修饰符|transient|短暂|
|类、方法和变量修饰符|volatile|易失|
|程序控制语句|break|跳出循环|
|程序控制语句|case|定义一个值以供 switch 选择|
|程序控制语句|continue|继续|
|程序控制语句|do|运行|
|程序控制语句|else|否则|
|程序控制语句|for|循环|
|程序控制语句|if|如果|
|程序控制语句|instanceof|实例|
|程序控制语句|return|返回|
|程序控制语句|switch|根据值选择执行|
|程序控制语句|while|循环|
|错误处理|assert|断言表达式是否为真|
|错误处理|catch|捕捉异常|
|错误处理|finally|有没有异常都执行|
|错误处理|throw|抛出一个异常对象|
|错误处理|throws|声明一个异常可能被抛出|
|错误处理|try|捕获异常|
|包相关|import|引入|
|包相关|package|包|
|基本类型|boolean|布尔型|
|基本类型|byte|字节型|
|基本类型|char|字符型|
|基本类型|double|双精度浮点|
|基本类型|float|单精度浮点|
|基本类型|int|整型|
|基本类型|long|长整型|
|基本类型|short|短整型|
|变量引用|super|父类,超类|
|变量引用|this|本类|
|变量引用|void|无返回值|
|保留关键字|goto|是关键字，但不能使用|
|保留关键字|const|是关键字，但不能使用|

> 注意：Java 的 null 不是关键字，类似于 true 和 false，它是一个字面常量，不允许作为标识符使用。

## 变量

上面有提到过，对象将其状态存储在字段中，而字段就是变量。Java 编程语言定义了以下类型的变量：

- 实例变量(由非静态字段声明):对象将其各个状态存储在"非静态字段"中，即没有static关键字的字段。非静态字段也称为实例变量，因为它们的值对于类的每个实例都是唯一的;
- 类变量(静态字段): 类变量是使用static修饰符声明的任何字段;这告诉编译器，无论该类被实例化了多少次，都存在此变量的一个副本。此外，可以添加final关键字以指示类变量永远不会改变，例如`final static int numGears = 6`。
- 局部变量用于方法体，方法通常将其临时状态存储在局部变量中。没有将变量指定为局部的特殊关键字。局部变量仅对声明它们的方法可见;无法从类的其余部分访问它们。
- 参数用于方法声明，变量是此方法的参数。

如果我们谈论的是`一般字段`（不包括局部变量和参数），我们可以简单地说`字段`。如果讨论适用于上述所有，我们可以简单地说`变量`。

命名变量的规则和约定如下:

- 变量名称区分大小写。变量的名称可以是任何合法的标识符 — Unicode 字母和数字的无限长度序列，以字母、美元符号`$`或下划线字符`_`开头。美元开头的常用于编译器内部使用，例如一些自动生成的字段，变量名称应始终避免使用美元符号`$`或下划线字符`_`开头。
- 变量的名称不能是java关键字。建议字段名使用驼峰命名(如果名称仅包含一个单词，请用所有小写字母拼写该单词。如果它由多个单词组成，请将每个后续单词的第一个字母大写),如果变量存储常量值，将每个字母大写，并用下划线字符分隔后续单词。

### 变量类型

Java 编程语言是静态类型的，这意味着必须先声明所有变量，然后才能使用它们。声明变量需要指明类型和名称，例如:

```java
int age =1;
```

变量的数据类型决定了其可能的值，以及可以对其执行的操作。除了面向对象的类型之外，Java还支持了8种基元类型。基元类型由语言预定义，并由保留关键字命名。基元值不与其他基元值共享状态。

|数据类型|字段默认值|描述|
|:---|:---|:---|
|byte|0|8-bit有符号位的二进制补码,取值范围 $[-128,127]$|
|short|0|16-bit有符号位的二进制补码,取值范围 $[-32768,32767]$|
|int|0|32-bit有符号位的二进制补码，取值范围 $[-2^{31},2^{31}-1]$|
|long|0|64-bit有符号的二进制补码，取值范围 $[-2^{63},2^{63}-1]$|
|float|0.0f|单精度32-bit IEEE754浮点数，精度取值范围 `[1.4E-45 , 3.4028235E38]`,这里的精度取值范围是指正数取值范围，取值范围是`[-3.40E+38 ~ +3.40E+38]`|
|double|0.0d|双精度64-bit IEEE754浮点数，精度取值范围 `[4.9E-324 , 1.7976931348623157E308]`,这里的精度取值范围是指正数取值范围，实际取值范围是`[-1.79E+308 ~ +1.79E+308]`|
|boolean|false|数据类型只有true和false两个值，数据类型表示1-bit信息，但是实际存储没有精确的定义|
|char|'\u0000'|16-bit的Unicode字符，取值范围`['\u0000','\uffff']`(也可以认为是 $[0,2^{16}-1]$ )|

> 编译器不会给局部变量设置默认值，需要手动初始化。

初始化基元类型的变量时不使用关键字。基元类型是语言中内置的特殊数据类型;它们不是从类创建的对象，直接在代码中使用文本表示。例如：

```java
boolean result = true;
char capitalC = 'C';
byte b = 100;
short s = 10000;
int i = 100000;
```

在声明数字时，Java还支持不同的进制。例如:

```java
// 十进制 26
int decVal = 26;
// 十六进制 26
int hexVal = 0x1a;
// 二进制 26
int binVal = 0b11010;
```

浮点数除了浮点表示，还支持使用科学计数法表示

```java
double d1 = 123.4d;
// 科学计数法表示，等同于d1
double d2 = 1.234e2;
float f1  = 123.4f;
```

java还支持在数字文本中使用下划线字符，用于分隔数字文本中的数字组，这可以提高代码的可读性，例如

```java
long creditCardNumber = 1234_5678_9012_3456L;
long socialSecurityNumber = 999_99_9999L;
float pi =  3.14_15F;
long hexBytes = 0xFF_EC_DE_5E;
long hexWords = 0xCAFE_BABE;
long maxLong = 0x7fff_ffff_ffff_ffffL;
byte nybbles = 0b0010_0101;
long bytes = 0b11010010_01101001_10010100_10010010;
```

只能在数字之间放置下划线;不能在以下位置放置下划线：

- 在数字的开头或结尾
- 与浮点文本中的小数点相邻
- 在F或L等后缀之前
- 在0x前缀中间或之前。


除了8种基元字符外，java还通过`java.lang.String`类为字符串提供了特殊支持。将字符串括在双引号内将自动创建一个新对象。字符串对象是不可变的，这意味着一旦创建，它们的值就无法更改。

### 自动类型转换

整型、实型（常量）、字符型数据可以混合运算。运算中，不同类型的数据先转化为同一类型，然后进行运算。转换从小到大。

```
小  ------------------------------------>  大

byte,short,char—> int —> long—> float —> double 
```

数据类型转换必须满足如下规则：

1. 不能对boolean类型进行类型转换。
2. 不能把对象类型转换成不相关类的对象。
3. 在把容量大的类型转换为容量小的类型时必须使用强制类型转换。
4. 强制转换过程中可能导致溢出或损失精度，例如：`int i =128;byte b = (byte)i;`因为 byte 类型是 8 位，最大值为127，所以当 int 强制转换为 byte 类型时，值 128 时候就会导致溢出。
5. 浮点数到整数的转换是通过舍弃小数得到，而不是四舍五入，例如：`(int)23.7 == 23; (int)-45.89f == -45`


### 数组

数组是一个容器对象，它保存单个类型的固定数量的值。数组的长度是在创建数组时确定的。创建后，其长度是固定的，可以使用内置属性length来确定任何数组的大小。

![](objects-tenElementArray.gif)

数组中的每个项目称为一个元素，每个元素都通过其数字索引访问。如上图所示，编号从 0 开始。例如，将在索引 9 处访问第 8 个元素。

与其他类型的变量的声明一样，数组声明有两个组成部分：数组的类型和数组的名称。数组的类型写为`type[]`。其中type是所包含元素的数据类型;括号是特殊符号，指示此变量是数组。声明实际上并不创建数组;它只是告诉编译器此变量将保存指定类型的数组。

- 创建数组的一种方法是使用new运算符。`int[] anArray = new int[10];`，然后再为数组的元素赋值:`anArray[0] = 100;`
- 可以使用快速语法来创建和初始化数组,数组的元素用逗号分隔，数组的长度由大括号之间提供的值数决定：

```java
int[] anArray = { 
    100, 200, 300,
    400, 500, 600, 
    700, 800, 900, 1000
};
```

还可以使用两组或多组括号声明数组（也称为多维数组）。多维数组是一个数组，其组件本身就是数组。这样做的结果是允许行的长度变化。例如:

```java
String[][] names = {
            {"Mr. ", "Mrs. ", "Ms. "},
            {"Smith", "Jones"}
};
```

System类提供了快速复制数组的方法:

```java
public static void arraycopy(Object src, int srcPos,
                             Object dest, int destPos, int length)
```
- src,dest这两个参数指定要从中复制的数组和要复制到的数组。
- srcPos,destPos,length这三个参数指定源数组中的起始位置、目标数组中的起始位置以及要复制的数组元素数。Objectint

在 java.util.Arrays 类中提供了几种用于执行数组操作的常见方法，如复制(copyOfRange)、排序(sort)和搜索数组(binarySearch)。

## 运算符

运算符是对一个、两个或三个操作数执行特定操作，然后返回结果的特殊符号。

|Operators|优先级|
|:---|:---|
|postfix|expr++,expr--|
|unary|++expr, --expr, +expr, -expr, ~ ,!|
|multiplicative|*,/,%|
|additive|+, -|
|shift|<< ,>> ,>>>|
|relational|< ,>, <=, >=, instanceof|
|equality	|==, !=|
|bitwise AND| &|
|bitwise exclusive OR |^|
|bitwise inclusive OR |	&#124;|
|logical AND|&&|
|logical OR	|&#124;&#124;|
|ternary	|? :|
|assignment	|=,+= ,-=, *=, /=, %=,&=, ^=, &#124;=, <<=, >>=, >>>=|

> 在上表，越靠上的运算符优先级越高，同一行上的运算符具有相同的优先级。当优先级相等的运算符出现在同一表达式中时，必须有一个规则来控制首先计算哪个。除赋值运算符外的所有二元运算符都从左到右计算;赋值运算符从右到左计算。

### 算术运算符

|Operator	|Description|例子|
|:---|:---|:---|
|+|加法运算符，也可以用于字符串拼接|a+b|
|-|减法运算符|a-b|
|*|乘法运算符|a*b|
|/|除法运算符|a/b|
|%|取余运算符|a%b|
|++|自增1运算符|a++|
|--|自减1运算符|b--|

### 比较运算符
|Operator	|Description|
|:---|:---|
|== |相等|
|!= |不相等|
|>  |大于|
|>= |大于等于|
|<  |小于|
|<= |小于等于|

### 条件运算符

|Operator	|Description|
|:---|:---|
|&& |逻辑与|
|&#124;&#124;| 逻辑或|
|!|用来反转操作数的逻辑状态。如果条件为true，则逻辑非运算符将得到false。|

### 位运算

|Operator	|Description|
|:---|:---|
|`＆`|如果相对应位都是1，则结果为1，否则为0|
|&#124;|如果相对应位都是 0，则结果为 0，否则为 1|
|`^`|如果相对应位值相同，则结果为0，否则为1	|
|`~`|按位取反运算符翻转操作数的每一位，即0变成1，1变成0|
|`<<`|按位左移运算符。左操作数按位左移右操作数指定的位数|
|`>>`|按位右移运算符。左操作数按位右移右操作数指定的位数|
|`>>>`|按位右移补零操作符。左操作数的值按右操作数指定的位数右移，移动得到的空位以零填充。|

## 表达式&语句

表达式是由变量、运算符和方法调用构造成的。例如`x+y/100`.

- 可以使用括号准确指定表达式的计算方式，例如`(x+y)/100`。
- 如果未显式指示要执行的操作的顺序，则顺序由表达式中运算符的优先级确定。首先执行具有较高优先级的运算符。

语句大致相当于自然语言中的句子。语句构成一个完整的执行单元。以下类型的表达式可以通过用分号终止表达式来组成语句。

- 赋值表达式
- 使用++或--
- 方法调用
- 对象创建表达式

除了表达式语句之外，还有另外两种类型的语句：声明语句和控制流语句。

- 声明语句声明一个变量,例如: `double aValue;`
- 控制流语句调节语句的执行顺序

代码块是大括号之间的一组零个或多个语句。
## 控制流

源文件中的语句通常按其出现的顺序从上到下执行。但是，控制流语句通过使用决策、循环和分支来分解执行流，使程序能够有条件地执行特定的代码块。

### if else语句

程序仅在特定测试的计算结果为True时，执行代码块中语句,否则执行else部分(如果有else部分)。例如:

```java
 if (isMoving) {
    currentSpeed--;
} else { // 可省略
    System.err.println("The bicycle has already stopped!");
} 
```

if 语句后面可以跟 else if…else 语句，这种语句可以检测到多种可能的情况。使用 if，else if，else 语句的时候，需要注意下面几点：

- if 语句至多有 1 个 else 语句，else 语句在所有的 else if 语句之后。
- if 语句可以有若干个 else if 语句，它们必须在 else 语句之前。
- 一旦其中一个 else if 语句检测为 true，其他的 else if 以及 else 语句都将跳过执行。

例如:

```java
if( x == 10 ){
    System.out.print("Value of X is 10");
}else if( x == 20 ){
    System.out.print("Value of X is 20");
}else if( x == 30 ){
    System.out.print("Value of X is 30");
}else{
    System.out.print("这是 else 语句");
}
```
### switch语句

Switch case 语句判断一个变量与一系列值中某个值是否相等，每个值称为一个分支。

```java
switch case 语句语法格式如下：

switch(expression){
    case value :
       //语句
       break; //可选
    case value :
       //语句
       break; //可选
    //你可以有任意数量的case语句
    default : //可选
       //语句
}
```
switch case 语句有如下规则：

- switch 语句中的变量类型可以是： byte、short、int 或者 char。从 Java SE 7 开始，switch 支持字符串 String 类型了，同时 case 标签必须为字符串常量或字面量。
- switch 语句可以拥有多个 case 语句。每个 case 后面跟一个要比较的值和冒号。
- case 语句中的值的数据类型必须与变量的数据类型相同，而且只能是常量或者字面常量。
- 当变量的值与 case 语句的值相等时，那么 case 语句之后的语句开始执行，直到 break 语句出现才会跳出 switch 语句。
- 当遇到 break 语句时，switch 语句终止。程序跳转到 switch 语句后面的语句执行。case 语句不必须要包含 break 语句。如果没有 break 语句出现，程序会继续执行下一条 case 语句，直到出现 break 语句。
- switch 语句可以包含一个 default 分支，该分支一般是 switch 语句的最后一个分支（可以在任何位置，但建议在最后一个）。default 在没有 case 语句的值和变量值相等的时候执行。default 分支不需要 break 语句。

例如:

```java
switch (month) {
    case 1:  monthString = "January";
              break;
    case 2:  monthString = "February";
              break;
    case 3:  monthString = "March";
              break;
    case 4:  monthString = "April";
              break;
    case 5:  monthString = "May";
              break;
    case 6:  monthString = "June";
              break;
    case 7:  monthString = "July";
              break;
    case 8:  monthString = "August";
              break;
    case 9:  monthString = "September";
              break;
    case 10: monthString = "October";
              break;
    case 11: monthString = "November";
              break;
    case 12: monthString = "December";
              break;
    default: monthString = "Invalid month";
              break;
}
```

### while语句

while语句的结构如下:

```
while( 布尔表达式 ) {
  statements;
}
```

例如:

```java
int count = 1;
while (count < 11) {
    System.out.println("Count is: " + count);
    count++;
}
```

只要布尔表达式为 true，循环就会一直执行下去。对于 while 语句而言，如果不满足条件，则不能进入循环。但有时候我们需要即使不满足条件，也至少执行一次。do…while 循环和 while 循环相似，不同的是，do…while 循环至少会执行一次。

```
do {
  statements;
}while(布尔表达式);
```

例如:

```java
int count = 1;
do {
    System.out.println("Count is: " + count);
    count++;
} while (count < 11);
```

### for 语句

for语句重复循环直到满足特定条件的方式。

```
for (initialization; termination;
     increment) {
    statement(s)
}
```

关于 for 循环有以下几点说明：

- 最先执行初始化步骤。可以声明一种类型，但可初始化一个或多个循环控制变量，也可以是空语句。
- 然后，检测布尔表达式的值。如果为 true，循环体被执行。如果为false，循环终止，开始执行循环体后面的语句。
- 执行一次循环后，更新循环控制变量。
- 再次检测布尔表达式。循环执行上面的过程。

例如:

```java
for(int i=1; i<11; i++){
    System.out.println("Count is: " + i);
}
```

for循环的三个表达式是可选的;可以按如下方式创建无限循环

```java
// infinite loop
for ( ; ; ) {
    // your code goes here
}
```

java还对集合和数组提供了一种for-each的增强语法。

```java
int[] numbers = 
    {1,2,3,4,5,6,7,8,9,10};
for (int item : numbers) {
    System.out.println("Count is: " + item);
}
```

### break

break 主要用在循环语句或者 switch 语句中，用来跳出整个语句块。

- break 默认跳出最里层的循环，并且继续执行该循环下面的语句。

```java
for (i = 0; i < arrayOfInts.length; i++) {
  if (arrayOfInts[i] == searchfor) {
      foundIt = true;
      break;
  }
}
```

- 如果 break后面带了label，那么会跳出label所对应的循环。

```java
search:
for (i = 0; i < arrayOfInts.length; i++) {
    for (j = 0; j < arrayOfInts[i].length;j++) {
        if (arrayOfInts[i][j] == searchfor) {
            foundIt = true;
            break search;
        }
    }
}
```

### continue

continue 适用于任何循环控制结构中。作用是让程序立刻跳转到下一次循环的迭代。
- 在 for 循环中，continue 语句使程序立即跳转到更新语句。
- 在 while 或者 do…while 循环中，程序立即跳转到布尔表达式的判断语句。

带label的continue语句跳过用给定lable标记的外部循环的当前迭代.

```java
test:
for (int i = 0; i <= max; i++) {
    int n = substring.length();
    int j = i;
    int k = 0;
    while (n-- != 0) {
        if (searchMe.charAt(j++) != substring.charAt(k++)) {
            continue test;
        }
    }
    foundIt = true;
        break test;
}
```
### return

最后一个分支语句是return语句。该语句从当前方法退出，控制流返回到调用该方法的位置。该语句有两种形式：一种返回值，另一种不返回值。要返回值，只需将值（或计算值的表达式）放在关键字之后。

## 类

一个Java中的类如下所示:

```java
public class Bicycle {
        
    // the Bicycle class has
    // three fields
    public int cadence;
    public int gear;
    public int speed;
        
    // the Bicycle class has
    // one constructor
    public Bicycle(int startCadence, int startSpeed, int startGear) {
        gear = startGear;
        cadence = startCadence;
        speed = startSpeed;
    }
        
    // the Bicycle class has
    // four methods
    public void setCadence(int newValue) {
        cadence = newValue;
    }
        
    public void setGear(int newValue) {
        gear = newValue;
    }
        
    public void applyBrake(int decrement) {
        speed -= decrement;
    }
        
    public void speedUp(int increment) {
        speed += increment;
    }
        
}
```

可以看到类由4个部分组成:

- 类声明
- 定义成员变量
- 定义构造器
- 定义方法

### 类声明

类声明语法如下:

```java
public class MyClass extends MySuperClass implements YourInterface {
    // field, constructor, and
    // method declarations
}
```

类主体（大括号之间的区域）包含：

- 用于初始化新对象的构造函数
- 提供类及其对象状态的字段的声明
- 以及实现类及其对象行为的方法

通常，类声明可以按以下顺序包含以下组件：

- 修饰词，例如public、private等。
- 类的父类（超类）的名称（如果有），前面是关键字 extend。一个类只能扩展（子类）一个父类。
- 由类实现的接口的逗号分隔列表（如果有），前面是关键字 implements。一个类可以实现多个接口。
- 类主体，用大括号括起来，{}。

### 成员变量

有几种变量：

- 类中的成员变量 - 这些变量称为字段。
- 方法或代码块中的变量 - 这些变量称为局部变量。
- 方法声明中的变量 - 这些变量称为参数。

字段声明由三个部分组成，顺序如下：

- 零个或多个修饰符，例如 public，private或static
- 字段的类型。
- 字段的名称。

本着封装的精神，通常将字段设为私有。对字段的操作通常通过方法间接完成。

### 定义方法

一个方法的声明示例如下:

```java
public double calculateAnswer(double wingSpan, int numberOfEngines,
                              double length, double grossTons) throws Exception{
    //do the calculation here
}
```
方法声明有六个组件，顺序如下：

- 修饰符 — 如 public, private.
- 返回类型 - 方法返回的值的数据类型，或者void如果方法不返回值。
- 方法名称 - 字段名称的规则也适用于方法名称，但约定略有不同。
- 括号中的参数列表 - 以逗号分隔的输入参数列表，前面是其数据类型，用括号括起来。如果没有参数，则必须使用空括号。
- 异常列表 - 方法可能抛出的异常
- 方法主体(在大括号之间) - 方法的主体代码

Java支持重载方法，Java可以区分具有不同方法签名的方法。这意味着，如果类中的方法具有不同的参数列表，则可以具有相同的名称。重载方法通过传递到方法中的参数的数量和类型进行区分，不能声明多个具有相同名称以及相同数量和类型的参数的方法，因为编译器无法区分它们。

> 编译器在区分方法时不考虑返回类型，因此即使两个方法具有不同的返回类型，也不能声明具有相同签名的两个方法。

方法的使用是通过对象来调用的。`objectReference.methodName(argumentList);`

例如:

```java
int areaOfRectangle = new Rectangle(100, 50).getArea();
```

#### 方法参数

Java支持使用动态参数列表(三个`.`)，将不确定长度的特定类型参数传递给方法，在方法内部，动态参数被视为一个数组。例如:

```java
public Polygon polygonFrom(Point... corners) {
  // corners是动态参数列表，可以被视为 Point[]
  int numberOfSides = corners.length;
  double squareOfSide1, lengthOfSide1;
  squareOfSide1 = (corners[1].x - corners[0].x)
                    * (corners[1].x - corners[0].x) 
                    + (corners[1].y - corners[0].y)
                    * (corners[1].y - corners[0].y);
  lengthOfSide1 = Math.sqrt(squareOfSide1);
}
```

参数的修改:

- 对基元类型参数值的任何更改都只存在于方法的范围内。当该方法返回时，对参数所做的任何更改都将丢失。(在调用方法时，会传递基元参数的副本)
- 引用数据类型参数（如对象）也按值传递到方法中。这意味着当该方法返回时，传入的引用仍引用以前相同的对象，换句话说，对引用类型参数的修改会作用其上。这也是Java方法的副作用。

### 定义构造函数

构造函数声明看起来像方法声明，不同之处在于它们使用类的名称并且没有返回类型。

与方法一样，Java 方法列表中的参数数量及其类型来区分构造函数。不能为同一类编写两个具有相同数量和类型参数的构造函数，因为平台无法区分它们。

编译器会自动为没有构造函数的任何类提供无参数的默认构造函数。此默认构造函数将调用超类的无参数构造函数。

要创建一个类的新对象，可以通过new 运算符 调用该类的构造函数:

```java
Bicycle myBike = new Bicycle(30, 0, 8);
```

### 方法重载

重载(overloading) 是在一个类里面，方法名字相同，而参数不同。返回类型可以相同也可以不同。每个重载的方法（或者构造函数）都必须有一个独一无二的参数类型列表。最常用的地方就是构造器的重载。

重载规则:

- 被重载的方法必须改变参数列表(参数个数或类型不一样)；
- 被重载的方法可以改变返回类型；
- 被重载的方法可以改变访问修饰符；
- 被重载的方法可以声明新的或更广的检查异常；
- 方法能够在同一个类中或者在一个子类中被重载。
- 无法以返回值类型作为重载函数的区分标准。
### 方法重写

方法的重写规则

- 参数列表与被重写方法的参数列表必须完全相同。
- 返回类型与被重写方法的返回类型可以不相同，但是必须是父类返回值的派生类（java5 及更早版本返回类型要一样，java7 及更高版本可以不同）。
- 访问权限不能比父类中被重写的方法的访问权限更低。例如：如果父类的一个方法被声明为 public，那么在子类中重写该方法就不能声明为 protected。
- 父类的成员方法只能被它的子类重写。
- 声明为 final 的方法不能被重写。
- 声明为 static 的方法不能被重写，但是能够被再次声明(子类中调用时，会隐藏父类的同名static方法)。
- 子类和父类在同一个包中，那么子类可以重写父类所有方法，除了声明为 private 和 final 的方法。
- 子类和父类不在同一个包中，那么子类只能够重写父类的声明为 public 和 protected 的非 final 方法。
- 重写的方法能够抛出任何非强制异常，无论被重写的方法是否抛出异常。但是，重写的方法不能抛出新的强制性异常，或者比被重写方法声明的更广泛的强制性异常，反之则可以。
- 构造方法不能被重写。
- 如果不能继承一个类，则不能重写该类的方法。
### this 关键字

在实例方法或构造函数中，this是对当前对象的引用，即调用其方法或构造函数的对象。可以使用this从实例方法或构造函数中引用当前对象的任何成员。

使用该关键字的最常见原因是字段被方法或构造函数参数遮蔽。例如:

```java
public class Point {
    public int x = 0;
    public int y = 0;
        
    //constructor
    public Point(int a, int b) {
        x = a;
        y = b;
    }
    //constructor
    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }
}
// 使用this
public class Point {
    public int x = 0;
    public int y = 0;
        
    //constructor
    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }
}
```

构造函数的每个参数都隐藏着对象的一个字段 — 构造函数 Point内部 x 是构造函数第一个参数的本地副本。要引用字段 x，构造函数必须使用this.x;

在构造函数中，还可以使用该关键字调用同一类中的另一个构造函数。这样做称为显式构造函数调用。对另一个构造函数的调用必须是构造函数中的第一行。

```java
public class Rectangle {
    private int x, y;
    private int width, height;
        
    public Rectangle() {
        this(0, 0, 1, 1);
    }
    public Rectangle(int width, int height) {
        this(0, 0, width, height);
    }
    public Rectangle(int x, int y, int width, int height) {
        this.x = x;
        this.y = y;
        this.width = width;
        this.height = height;
    }
    ...
}
```

### super 关键字

可以通过super关键字来实现对父类成员的访问，用来引用当前对象的父类。super只能出现在构造方法和实例方法中，代表的是当前对象的父类型实例。和this相似也有两种用法:

- 在构造器中引用父类构造器
- super.成员变量 或者 super.成员方法，常用于区分子类和父类中相同名字的成员变量或者相同名字的成员方法。

### 访问修饰符

Java中，可以使用访问控制符来保护对类、变量、方法和构造方法的访问。Java 支持 4 种不同的访问权限。

- default (即默认，什么也不写）: 在同一包内可见，不使用任何修饰符。使用对象：类、接口、变量、方法。
- private : 在同一类内可见。使用对象：变量、方法。 注意：不能修饰类（内部类除外）
- public : 对所有类可见。使用对象：类、接口、变量、方法
- protected : 对同一包内的类和所有子类可见。使用对象：变量、方法。 注意：不能修饰类（内部类除外）。

|修饰符|当前类|同一包内|子孙类(同一包)|子孙类(不同包)|其他包|
|:---|:---|:---|:---|:---|:---|
|public|Y|Y|Y|Y|Y|
|protected|Y|Y|Y|Y/N|N|
|default|Y|Y|Y|N|N|
|private|Y|N|N|N|N|

对于protected，如果子类与基类不在同一包中：那么在子类中，子类实例可以访问其从基类继承而来的 protected 方法，而不能访问基类实例的protected方法。例如:

```java
package p2;
class MyObject2 {
    protected Object clone() throws CloneNotSupportedException{
       return super.clone();
    }
}
 
package p22;
public class Test2 extends MyObject2 {
    public static void main(String args[]) {
       MyObject2 obj = new MyObject2();
       obj.clone(); // Compile Error         ----（1）
 
       Test2 tobj = new Test2();
       tobj.clone(); // Complie OK         ----（2）
    }
}
//对于(1)而言，clone()方法来自于类MyObject2本身，因此其可见性为包p2及MyObject2的子类，虽然Test2是MyObject2的子类，但在Test2中不能访问基类MyObject2的protected方法clone()，因此编译不通过;
// 对于(2)而言，由于在Test2中访问的是其本身实例的从基类MyObject2继承来的的clone()，因此编译通过。
```

### static 修饰符

- 静态变量：static 关键字用来声明独立于对象的静态变量，无论一个类实例化多少对象，它的静态变量只有一份拷贝。 静态变量也被称为类变量。局部变量不能被声明为 static 变量。
- 静态方法：static 关键字用来声明独立于对象的静态方法。静态方法不能使用类的非静态变量。静态方法从参数列表得到数据，然后计算这些数据。

对类变量和方法的访问可以直接使用 classname.variablename 和 classname.methodname 的方式访问。

### final 修饰符


- final 表示"最后的、最终的"含义，变量一旦赋值后，不能被重新赋值。被 final 修饰的实例变量必须显式指定初始值。final 修饰符通常和 static 修饰符一起使用来创建类常量。
  - 对于final静态变量，必须在声明时赋值或静态代码块中赋值
  - 对于final实例变量，必须在声明时赋值(或初始化代码块中赋值)或构造器中赋值
- 父类中的 final 方法可以被子类继承，但是不能被子类重写。声明 final 方法的主要目的是防止该方法的内容被修改。
- final 类不能被继承，没有类能够继承 final 类的任何特性。

### abstract 修饰符

- 抽象类：抽象类不能用来实例化对象，声明抽象类的唯一目的是为了将来对该类进行扩充。一个类不能同时被 abstract 和 final 修饰。如果一个类包含抽象方法，那么该类一定要声明为抽象类，否则将出现编译错误。抽象类可以包含抽象方法和非抽象方法。
- 抽象方法：抽象方法是一种没有任何实现的方法，该方法的具体实现由子类提供。抽象方法不能被声明成 final 和 static。任何继承抽象类的子类必须实现父类的所有抽象方法，除非该子类也是抽象类。如果一个类包含若干个抽象方法，那么该类必须声明为抽象类。抽象类可以不包含抽象方法。抽象方法的声明以分号结尾，例如：public abstract sample();。

## 接口

在Java中接口是一种引用类型，类似于类，只能包含常量、方法签名、缺省方法、静态方法和嵌套类型。方法主体仅存在于默认方法和静态方法中。接口不能实例化 - 它们只能由类实现或由其他接口扩展。

接口声明由修饰符、关键字interface、接口名称、父接口的逗号分隔列表（如果有）和接口主体组成。例如：

```java
public interface GroupedInterface extends Interface1, Interface2, Interface3 {
    double E = 2.718282;
 
    // method signatures
    void doSomething (int i, double x);
    int doSomethingElse(String s);
}
```
public 访问修饰符指示接口可由任何包中的任何类使用。如果未指定，则接口只能由与接口相同的包中定义的类访问.

一个接口可以扩展其他接口，就像一个类子类一样，也可以扩展另一个类。但是，一个类只能扩展另一个类，而一个接口可以扩展任意数量的接口。接口声明包括它扩展的所有接口的逗号分隔列表。

若要声明实现接口的类，请在类声明中包含implements子句。类可以实现多个接口，因此关键字implements后跟该类实现的接口的逗号分隔列表。然后就是在类中声明并实现接口中的方法签名。

方法签名就是没有方法体的函数，如果想在接口中定义完整的方法，那么需要将方法设置为默认方法(给方法增加default修饰符)。例如:

```java
public interface HandleInvalidTimeZoneClient extends TimeClient {
    default public ZonedDateTime getZonedDateTime(String zoneString) {
        try {
            return ZonedDateTime.of(getLocalDateTime(),ZoneId.of(zoneString)); 
        } catch (DateTimeException e) {
            System.err.println("Invalid zone ID: " + zoneString +
                "; using the default time zone instead.");
            return ZonedDateTime.of(getLocalDateTime(),ZoneId.systemDefault());
        }
    }
}
```

默认方法会被实现该接口的类继承。

接口也像类一样支持static 字段和static 方法。

## 嵌套类

Java可在另一个类中定义一个类。这样的类称为嵌套类。嵌套类分为两类：非静态类和静态类。非静态嵌套类称为内部类。static声明的嵌套类称为静态嵌套类。嵌套类是其封闭类的成员。

- 内部类与实例方法和变量一样，内部类与其封闭类的实例相关联，并可以直接访问该对象的方法和字段。此外，由于内部类与实例相关联，因此它本身无法定义任何静态成员。若要实例化内部类，必须先实例化外部类。然后，使用以下语法在外部对象中创建内部对象：

```java
OuterClass outerObject = new OuterClass();
OuterClass.InnerClass innerObject = outerObject.new InnerClass();
```
- 静态嵌套类与类方法和变量一样，静态嵌套类与其外部类相关联。与静态类方法一样，静态嵌套类不能直接引用其封闭类中定义的实例变量或方法：它只能通过对象引用使用它们。实例化静态嵌套类的方式与实例化顶级类的方式相同：

```java
StaticNestedClass staticNestedObject = new StaticNestedClass();
```

### 内部类访问外部类

在内部类访问外部类时，使用语法为`OutClass.this.xxx`,例如:

```java
public class ShadowTest {

    public int x = 0;

    class FirstLevel {

        public int x = 1;

        void methodInFirstLevel(int x) {
            System.out.println("x = " + x);
            System.out.println("this.x = " + this.x);
            System.out.println("ShadowTest.this.x = " + ShadowTest.this.x);
        }
    }

    public static void main(String... args) {
        ShadowTest st = new ShadowTest();
        ShadowTest.FirstLevel fl = st.new FirstLevel();
        fl.methodInFirstLevel(23);
    }
}
// 以下是输出：

//x = 23
//this.x = 1
//ShadowTest.this.x = 0
```

### 本地类&匿名类

- 本地类是在外部类方法体中声明的局部内部类
- 匿名类是在外部类方法体中声明的局部内部类，且没有名字。

局部类只能访问声明为 final 的局部变量。局部类类似于内部类，因为它们不能定义或声明任何静态成员。静态方法中的局部类，例如在静态方法中定义的类，只能引用封闭类的静态成员。

本地类是非静态的，因为它们可以访问 封闭块的实例成员。因此，他们不能包含大多数类型的静态声明。本地类可以具有静态成员，前提是它们是常量变量(常量变量是基元类型的变量或声明为 final 并使用编译时常量表达式初始化的类型)。

```java
  public void sayGoodbyeInEnglish() {
        class EnglishGoodbye {
            public static final String farewell = "Bye bye";
            public void sayGoodbye() {
                System.out.println(farewell);
            }
        }
        EnglishGoodbye myEnglishGoodbye = new EnglishGoodbye();
        myEnglishGoodbye.sayGoodbye();
    }
```

匿名类类似于本地类，不同处是本地类是类定义而匿名类是一个表达式。匿名类表达式的语法类似于构造函数的调用，除了有一个类定义 包含在代码块中。例如:

```java
interface HelloWorld {
  public void greet();
  public void greetSomeone(String someone);
}
 HelloWorld frenchGreeting = new HelloWorld() {
      String name = "tout le monde";
      public void greet() {
          greetSomeone("tout le monde");
      }
      public void greetSomeone(String someone) {
          name = someone;
          System.out.println("Salut " + name);
      }
  };
```

> 不能在匿名类中声明构造函数。

## 参考

<div id="refer-anchor-1"></div>

- [1] [java8 tutorial](https://docs.oracle.com/javase/tutorial/index.html)

<div id="refer-anchor-2"></div>

- [2] [java8 doc](https://docs.oracle.com/javase/8/docs/technotes/guides/javadoc/index.html)

