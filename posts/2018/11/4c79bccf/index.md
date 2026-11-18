# 语法范式

上下文无关的组成部分:

* 终结符号
* 非终结符号
* 一个开始符号
* 一组产生式

例如，下面数学表达式:

$$
expr \to expr+term
$$

$$
expr \to expr-term
$$

$$
expr \to term
$$

$$
term \to term * factor
$$  

$$
term \to term/factor
$$  

$$
term \to factor
$$

$$
factor \to (expr)
$$

$$
factor \to id
$$ 

1. 终结符号(词法单元)是组成串的基本符号，例如上面的 `+`,`-`。
2. 非终结符号是表示串的集合的语法变量,例如上面的term和factor。非终结符号表示的串集合用于定义由文法生成的语言。

<!--more-->

### 左结合 & 右结合

$expr \to term \space opeartor \space expr \space|\space term$

$E \to T \space \lbrack \space op \lbrack \space T \space op \space T \rbrack \rbrack$

上述的操作符operator是右结合的(优先结合右边的部分)。

$expr \to expr \space opeartor \space term \space|\space term$

$E \to \lbrack \lbrack T \space op \space T \rbrack op \space T   \rbrack \space T$

上述的操作符operator是左结合的(优先结合左边的部分)。

### 左递归的消除

注意,下面的文法是左递归的:

$expr \to expr+term$ 

这种文法是不能直接用于自顶向下分析的。expr -> expr(-> expr(...) + term) +term。需要使用递归消除技术，来消除左递归。

左递归分为两种:

* 直接左递归: $P \to Pa$
* 间接左递归: $P \to Aa ,A \to ...\to PB$

对于直接左递归，一般形式是 $P \to PX | Y$.

1. $P \to YP'$ 
2. $P' \to XP'| \varepsilon$

> P 实际上是 Y 开头的后面若干个X

对于间接左递归：

1. 若消除过程中出现了直接左递归，就按照直接左递归的方法，来消除
2. 若产生式右部最左的符号是非终结符，且这个非终结符序号大于等于左部非终结符，则暂不处理（后面会处理到）
3. 若序号小于左部的非终结符，则用之前非终结符序号对应的产生式右部来替换 

```java
// 1.把文法G的所有非终结符按任意顺序排列，并编号
[P1,P2,……,Pn]

// 2.按上面的排列顺序，对这些非终结符进行遍历
for(int i = 1; i <= n; ++i) {
  
  for(int j = 1; i <= i - 1; ++j) {
    // 3.将当前处理的非终结符中的序号小于等于它的非终结符按规则3）进行替换（序号大于的按规则2）处理）
  }
  // 4.消除i序号的非终结符的直接左递归（如果存在的话）
  1）
}
// 5.删除其中不可达的非终结符（从开始符开始，无法再推出的非终结符）
```

例如存在如下文法G，要消除左递归

```
1. S → Qc | c
2. Q → Rb | b
3. R → Sa | a
```

1. 把文法G的所有非终结符按任意顺序排列，并编号`R、Q、S`
2. 按上面的排列顺序，对这些非终结符进行遍历
3. 将当前处理的非终结符中的序号小于等于它的非终结符按规则3）进行替换（序号大于的按规则2）处理）

```
R:
R的右部中的非终结符有S;
S的下标大于R，可以暂时不处理;
所以此时R为：R  →  Sa | a

----------------------------------------------
Q:
Q的右部中的非终结符有R;
R的下标小于Q，将R的右部替换进来;
所以此时Q改写为：Q  →  Sab | ab | b;
S的下标大于Q，可以暂时不处理;
所以此时Q为：Q  →  Sab | ab | b;

-----------------------------------------
S:
S的右部中的非终结符有Q;
Q的下标小于S，将Q的右部替换进来;
所以此时S改写为：S  →  Sabc |abc | bc | c
S的下标等于S，可以暂时不处理;
所以此时S为：S  →  Sabc |abc | bc | c
```

4. 消除i序号的非终结符的直接左递归（如果存在的话）

```
S  →  Sabc |abc | bc | c
∴  X = abc，Y = abc | bc | c
∴ 直接消除左递归的结果是：
S  →  abcS' | bcS' | cS'
S'  → abcS' | ε
```

5. 删除其中不可达的非终结符，这里就是Q、R了

## BNF

在BNF表示法中，实体按照以下格式以其他实体的形式定义：

```BNF
<a> :: = <b> <c>
```

这种表示法意味着实体`<a>`由实体`<b>`后跟实体`<c>`组成。由<和>括起来的实体在语法的其他地方进一步定义。<和>之间没有出现的任何东西都代表着它自己。

如果一个实体可以由多个序列定义，则序列由垂直线分隔：

```BNF
<a> :: = <b> <c> | <d> "BEGIN"
```

在此示例中，实体`<a>`由`<b>`后跟实体`<c>`或`<d>`后跟字符`BEGIN`组成。双引号包围的代表字符。

```BNF
<personal-part> ::= <initial> "." | <first-name>
```

正在定义的实体也可以出现在定义中，在这种情况下，定义是递归的。例如：

```BNF
<a> :: = <b> <c> | <d> "BEGIN"| <a> <e>
```

> 一个BNF 的分析例子: [Algol-BNF](https://web.archive.org/web/20060925132043/https://www.lrz-muenchen.de/~bernhard/Algol-BNF.html)  

## EBNF

EBNF是一种表达形式语言语法的代码。EBNF由终端符号和非终端生成规则组成，这些规则是管理终端符号如何组合成合法序列的限制。

|用法|符号|
|:---|:---|
|定义|=|
|级联，连接符号|,|
|终止|;|
|OR|&#124;|
|可选的|[ ... ]|
|重复|{ ... }|
|分组|( ... )|
|字符串，内部不可出现`"`|" ... "|
|字符串,，内部不可出现`'`|' ... '|
|注释|(* ... *)|
|特殊序列|? ... ?|
|除了|-|

EBNF 文法部分的特殊序列，它是在问号包围内的任意文本，其解释超出了 EBNF 标准的范围。例如，空格字符可以用如下规则定义:

```EBNF
space = ? US-ASCII character 32 ?;
```

### 描述自身

```EBNF
letter = "A" | "B" | "C" | "D" | "E" | "F" | "G"
       | "H" | "I" | "J" | "K" | "L" | "M" | "N"
       | "O" | "P" | "Q" | "R" | "S" | "T" | "U"
       | "V" | "W" | "X" | "Y" | "Z" | "a" | "b"
       | "c" | "d" | "e" | "f" | "g" | "h" | "i"
       | "j" | "k" | "l" | "m" | "n" | "o" | "p"
       | "q" | "r" | "s" | "t" | "u" | "v" | "w"
       | "x" | "y" | "z" ;
digit = "0" | "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9" ;
symbol = "[" | "]" | "{" | "}" | "(" | ")" | "<" | ">"
       | "'" | '"' | "=" | "|" | "." | "," | ";" ;
character = letter | digit | symbol | "_" ;

identifier = letter , { letter | digit | "_" } ;
terminal = "'" , character , { character } , "'"
         | '"' , character , { character } , '"' ;

lhs = identifier ;
rhs = identifier
     | terminal
     | "[" , rhs , "]"
     | "{" , rhs , "}"
     | "(" , rhs , ")"
     | rhs , "|" , rhs
     | rhs , "," , rhs ;

rule = lhs , "=" , rhs , ";" ;
grammar = { rule } ;
```

### Pascal风格的例子

```EBNF
 (* a simple program syntax in EBNF − Wikipedia *)
 program = 'PROGRAM', white space, identifier, white space,
            'BEGIN', white space,
            { assignment, ";", white space },
            'END.' ;
 identifier = alphabetic character, { alphabetic character | digit } ;
 number = [ "-" ], digit, { digit } ;
 string = '"' , { all characters - '"' }, '"' ;
 assignment = identifier , ":=" , ( number | identifier | string ) ;
 alphabetic character = "A" | "B" | "C" | "D" | "E" | "F" | "G"
                      | "H" | "I" | "J" | "K" | "L" | "M" | "N"
                      | "O" | "P" | "Q" | "R" | "S" | "T" | "U"
                      | "V" | "W" | "X" | "Y" | "Z" ;
 digit = "0" | "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9" ;
 white space = ? white space characters ? ;
 all characters = ? all visible characters ? ;
```

#### 代码示例

```Pascal
PROGRAM DEMO1
 BEGIN
   A:=3;
   B:=45;
   H:=-100023;
   C:=A;
   D123:=B34A;
   BABOON:=GIRAFFE;
   TEXT:="Hello world!";
 END.
```

## ABNF

### ABNF 结构

```ABNF
rule = definition ; comment CR LF
```

### 定义

|规则|定义|含义|
|:---|:---|:---|
|ALPHA|%x41-5A / %x61-7A|大写和小写ASCII字母(A-Z，a-z)|
|DIGIT|%x30-39|十进制数字(0-9)|
|HEXDIG|DIGIT / "A" / "B" / "C" / "D" / "E" / "F"|十六进制数字(0-9，A-F)|
|DQUOTE|%x22|双引号|
|SP|%x20|空格Space|
|HTAB|%x09|tab|
|WSP|SP / HTAB| Space and horizontal tab|
|LWSP|*(WSP / CRLF WSP)|Linear white space (past newline)|
|VCHAR|%x21-7E|可见字符|
|CHAR|%x01-7F|任何ASCII字符，不包括NUL|
|OCTET|%x00-FF|8 bits of data|
|CTL|%x00-1F / %x7F|Controls|
|CR|%x0D|回车|
|LF|%x0A|换行|
|CRLF|CR LF|标准换行|
|BIT|"0" / "1"|二进制数字|

### 操作符

|用法|例子|
|:---|:---|
|注释|`;comment` 从`;`到行尾表示注释的开始和结束|
|级联，连接|`Rule1 Rule2`表示Rule1 Rule2的连接|
|终止|;|
|OR|/|
|值范围|`%c##-##`,例如 OCTAL = %x30-37|
|分组|( ... )|
|可变重复|`<a>*<b>element`表示元素的重复。可选项`<a>`提供要包含的最少元素数（默认值为0）。可选项`<b>`给出了要包含的最大元素数（默认值为无穷大）.`*element`:零个或更多的元件，`*1element`:零个或一个元件，`1*element`:一个或多个元件，`2*3element`:两个或三个元素|
|具体重复|`<a>element`表示明确的元素数量，,相当于`<a>*<a>element`,2DIGIT:两个数字|
|可选顺序|`[Rule]`表示可选元素|
|字符串，内部不可出现`"`|" ... "|
|字符串,，内部不可出现`'`|' ... '|
|除了|-|

以下运算符具有从最严格的绑定到最松散绑定的给定优先级：

* 字符串
* 注释
* 值范围
* 重复
* 分组，可选
* 级联
* 替代

### 例子

```ABNF
postal-address   = name-part street zip-part

name-part        = *(personal-part SP) last-name [SP suffix] CRLF
name-part        =/ personal-part CRLF

personal-part    = first-name / (initial ".")
first-name       = *ALPHA
initial          = ALPHA
last-name        = *ALPHA
suffix           = ("Jr." / "Sr." / 1*("I" / "V" / "X"))

street           = [apt SP] house-num SP street-name CRLF
apt              = 1*4DIGIT
house-num        = 1*8(DIGIT / ALPHA)
street-name      = 1*VCHAR

zip-part         = town-name "," SP state 1*2SP zip-code CRLF
town-name        = 1*(ALPHA / SP)
state            = 2ALPHA
zip-code         = 5DIGIT ["-" 4DIGIT]
```
## 参考

- [1] [grammars-bnf-ebnf](http://matt.might.net/articles/grammars-bnf-ebnf/)  
- [2] [Backus–Naur form](https://en.wikipedia.org/wiki/Backus%E2%80%93Naur_form)
- [3] [Extended Backus-Naur form](https://en.wikipedia.org/wiki/Extended_Backus%E2%80%93Naur_form)
- [4] [Augmented Backus-Naur form](https://en.wikipedia.org/wiki/Augmented_Backus%E2%80%93Naur_form)

