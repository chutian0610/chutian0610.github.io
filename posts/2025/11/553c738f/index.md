# JavaCC:Token Manager

JavaCC的词法解析被转化为一组词法状态，每个状态都有一个唯一的标识符用于命名。Token Manager(生成的代码)在运行时总是处于状态之一，默认的一个词法状态是`DEFAULT`。当Token Manager初始化时，默认情况下是以`DEFAULT`状态开始。也可以在Token Manager 构造时，指定起始词法状态[^1]。

<!--more-->

## Lexical States

每个词法状态都包含一个有序的正则表达式列表(顺序同文件定义顺序)。当正则表达式匹配成功时有 4种 action可以执行。

### 正则匹配

当前词法状态下的所有正则表达式都被认为是潜在的匹配候选。令牌管理器消耗输入流中可能与这些正则表达式之一匹配的最大数量的字符。也就是说，令牌管理器偏好最长的可能匹配。如果有多个相同长度的最长匹配，则匹配的正则表达式是语法文件中出现顺序最早的正则表达式。

如上所述，令牌管理器在任何时刻都处于一种状态。此时，令牌管理器仅考虑在此状态下定义的正则表达式以进行匹配。在匹配之后，可以指定要执行的操作以及要移动到的新词法状态。如果未指定新的词法状态，则令牌管理器将保持当前状态。

### 执行操作

|类型|动作|
|:---|:---|
|SKIP|简单地丢弃匹配的字符串（在执行任何词法操作之后）。|
|MORE|继续到下一个状态，带上匹配的字符串沿着。此字符串将作为新匹配字符串的前缀。|
|TOKEN|使用匹配的字符串创建一个Token，并将其发送给解析器（或任何调用方）。|
|SPECIAL_TOKEN|创建不参与分析的特殊Token。保存匹配的Token，以便与下一个匹配的TOKEN一起沿着返回。|

无论何时检测到文件`<EOF>`的结尾，它都会导致创建`<EOF>`Token。而不管词法分析器的当前状态如何。但是，如果在匹配正则表达式的过程中检测到`<EOF>`，或者在匹配MORE正则表达式之后立即检测到undefined，则会报告错误。

> 在`TOKEN_MGR_DECLS`区域中声明的所有变量和方法（见下文）都可以在这里使用。

### 操作中的变量

- `image`变量是一个 StringBuffer[^2],包含自最后一个 SKIP,TOKEN或SPECIAL_TOKEN以来匹配的所有字符，可以自由的修改(但是不能赋值为 null)。如果对image进行了更改，则此更改将传递给后续匹配（如果当前匹配是MORE）。image的内容不会自动分配给匹配Token的image字段。如果希望这样做，必须在TOKEN或SPECIAL_TOKEN正则表达式的词法操作中显式地分配它。


```javacc
<DEFAULT> MORE : { "a" : S1 }
<S1> MORE :
{
  "b" {#^1 int l = image.length()-1; image.setCharAt(l, image.charAt(l).toUpperCase()); }#^2         
    : S2
}

<S2> TOKEN :
{
  "cd" {#^3 x = image; } : DEFAULT 
}
```

在上述示例中，在由`#^1`,`#^2`和`#^3`标记的3个点处的image值为：

```
#^1: "ab"
#^2: "aB"
#^3: "aBcd"
```


- `int lengthOfMatch()`是一个方法，返回当前匹配的长度(不在 MORE上累加)。与上述相同的示例，lengthOfMatch的值为：

```
#^1: 1 (the size of "b")
#^2: 1 (does not change due to lexical actions)
#^3: 2 (the size of "cd")
```

- `int curLexState()`是一个方法，返回当前词法状态的索引。命名为词法状态的常量会生成到`${ParserName}Constants`文件中，因此您可以引用词法状态，而不必担心它们的实际索引值。

- `matchedToken`是一个 Token类型变量，只能在与TOKEN和SPECIAL_TOKEN正则表达式关联的操作中使用。这被设置为将返回给解析器的令牌。您可以更改此变量，从而导致更改后的标记返回到解析器，而不是原始标记。例如: 可以将变量image的值分配给matchedToken.image。这就是对返回Token的修改。

例如在上面的`<S2> TOKEN`中改为下面的实现

```javacc
<S2> TOKEN :
{
  "cd" { matchedToken.image = image.toString(); } : DEFAULT
}
```

返回到解析器的Token将其image字段设置为`aBcd`。如果未执行此分配，则image字段将保持为`abcd`。

### 共享属性

词法操作可以访问一组类级别的声明。这些声明是使用以下语法在JavaCC文件中引入的：

```bnf
token_manager_decls ::=
  "TOKEN_MGR_DECLS" ":"
  "{" java_declarations_and_code "}"
```

这些声明可从所有词法操作访问。例如:

```javacc
TOKEN_MGR_DECLS :
{
  int stringSize;
}
MORE :
{
  "\"" {stringSize = 0;} : WithinString
}

<WithinString> TOKEN :
{
  <STRLIT: "\""> {System.out.println("Size = " + stringSize);} : DEFAULT
}

<WithinString> MORE :
{
  <~["\n","\r"]> {stringSize++;}
}
```

### 特殊Token

特殊Token类似于 Token，不同之处在于它们被允许出现在输入文件中的任何位置（任何两个Token之间）。任何被定义为SPECIAL_TOKEN的正则表达式都可以通过词法和语法规范中的用户操作以特殊的方式访问。这允许在解析期间恢复这些令牌，同时这些令牌不参与解析。

Token类现在有了一个额外的字段：`Token specialToken;`

- 此字段指向紧贴在当前Token（特殊或其他）之前的特殊Token。
- 如果紧贴在当前Token之前的是常规Token（而不是特殊Token），则此字段设置为null。

常规Token的next字段继续具有相同的含义，即，它们指向下一个常规Token，除了在`<EOF>`Token的情况下，其中next字段是null。

特殊Token的next字段指向紧跟在当前Token之后的特殊Token。如果紧跟在当前Token之后的令牌是常规令牌，则将next字段设置为null。

```java
// 打印所有在常规token t之前的特殊token

if (t.specialToken == null) {
  return;
}

// walk back the special token chain until it reaches the first special token after the previous regular token
Token tmp_t = t.specialToken;
while (tmp_t.specialToken != null) {
  tmp_t = tmp_t.specialToken;
}

// now walk the special token chain forward and print them in the process
while (tmp_t != null) {
  System.out.println(tmp_t.image);
  tmp_t = tmp_t.next;
}
```

### 常见示例

- [多行注释](https://github.com/chutian0610/javacc-demo/blob/master/tutorial03/src/main/javacc/Calculator.jj#L53-L75)

```javacc
MORE : {
    "/*" : IN_COMMENT
}

<IN_COMMENT> MORE : {
    < ~[] >  // 匹配除文件结束外的所有字符
}

<IN_COMMENT> SPECIAL_TOKEN : {
    "*/"
     {
        System.out.println("\n=== 发现多行注释 ===");
        System.out.println("注释内容: " + matchedToken.image);
        System.out.println("开始行: " + matchedToken.beginLine);
        System.out.println("结束行: " + matchedToken.endLine);
        System.out.println("开始列: " + matchedToken.beginColumn);
        System.out.println("结束列: " + matchedToken.endColumn);
        System.out.println("注释长度: " + matchedToken.image.length());
        System.out.println("=== 注释结束 ===\n");
     }
    : DEFAULT

}
```

- 字符串处理:

```javacc
// 定义词法状态
TOKEN_MGR_DECLS : {
  private StringBuilder stringBuffer = new StringBuilder();
  private int stringStartLine;
  private int stringStartColumn;
}

// 进入字符串状态
TOKEN : {
    < DQUOTE: "\"" > : IN_STRING
    {
        // 记录字符串开始位置
        stringStartLine = matchedToken.beginLine;
        stringStartColumn = matchedToken.beginColumn;
        stringBuffer = new StringBuilder();
        stringBuffer.append("\"");
        
        System.out.println("开始解析字符串 (第" + stringStartLine + "行)");
    }
}

// 在字符串状态中的处理
<IN_STRING> TOKEN : {
    < STRING_CHAR: 
        ~["\"","\\","\n","\r"]  // 非特殊字符
    >
    {
        stringBuffer.append(matchedToken.image);
    }
  | < ESCAPE_SEQUENCE: "\\" >
    {
        stringBuffer.append("\\");
        SwitchTo(IN_ESCAPE);  // 切换到转义序列状态
    }
  | < STRING_END: "\"" > : DEFAULT
    {
        stringBuffer.append("\"");
        String stringLiteral = stringBuffer.toString();
        strings.add(stringLiteral);
        
        System.out.println("\n=== 完整字符串 ===");
        System.out.println("原始: " + stringLiteral);
        System.out.println("内容: " + stringLiteral.substring(1, stringLiteral.length()-1));
        System.out.println("位置: 第" + stringStartLine + "行, 第" + stringStartColumn + "列");
        System.out.println("================\n");
    }
  | < UNTERMINATED_STRING: "\n" | "\r" >
    {
        System.err.println("错误: 第" + stringStartLine + "行的字符串未正确结束");
        SwitchTo(DEFAULT);
    }
    // 转义序列状态
<IN_ESCAPE> TOKEN : {
    < ESCAPED_CHAR: 
        ["n","t","b","r","f","\\","'","\"","0","u"] 
    >
    {
        stringBuffer.append(matchedToken.image);
        SwitchTo(IN_STRING);  // 回到字符串状态
        
        System.out.println("  转义序列: \\" + matchedToken.image);
    }
  | < UNICODE_ESCAPE: ["0"-"9","a"-"f","A"-"F"] > : IN_UNICODE
    {
        stringBuffer.append(matchedToken.image);
    }
  | < INVALID_ESCAPE: ~[] >
    {
        System.err.println("警告: 第" + stringStartLine + "行发现无效转义序列: \\" + matchedToken.image);
        stringBuffer.append(matchedToken.image);
        SwitchTo(IN_STRING);
    }
}

// Unicode 转义状态（处理 \uXXXX）
<IN_UNICODE> MORE : {
    <HEX_DIGIT: ["0"-"9","a"-"f","A"-"F"]>
    {
        stringBuffer.append(matchedToken.image);
        // 收集4个十六进制数字
        if (stringBuffer.charAt(stringBuffer.length()-1) != 'u') {
            int lastDigits = 0;
            for (int i = stringBuffer.length()-1; i >= 0; i--) {
                if (stringBuffer.charAt(i) == 'u') {
                    lastDigits = stringBuffer.length() - i - 1;
                    break;
                }
            }
            if (lastDigits == 4) {
                SwitchTo(IN_STRING);
                System.out.println("  Unicode转义: \\u" + 
                    stringBuffer.substring(stringBuffer.length()-4));
            }
        }
    }
}
}
```

[^1]: 所有在语法中作为扩展单元出现的正则表达式都被认为处于DEFAULT词法状态
[^2]: 和Token的image字段不同

