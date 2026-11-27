# JavaCC 实现一个计算器

[上一篇文章](/posts/2025/11/66ec4c97/)我们简单介绍了 JavaCC的语法和使用。本文将以实现一个计算器为例，介绍JavaCC 在实际代码中的使用。

<!--more-->

## 左递归


四则运算中加减乘除是标准的左递归。

$$
expr \to expr+term
$$

$$
expr \to expr-term
$$

$$
term \to term * factor
$$  

$$
term \to term/factor
$$

JavaCC 本身是不能处理左递归的，需要我们自己通过递归下降实现左递归消除。

$$
expr \to term(+|-) term
$$

注意，不要错写成右递归形式，会导致计算错误。

$$
expr \to term(+|-) expr
$$

一个错误的语法文件如下:

```
options
{
    STATIC=false;
    LOOKAHEAD=2;
}
PARSER_BEGIN(CalculatorError)
package io.github.chutian0610.javacc.tutorial02.error;
public class CalculatorError
{
}
PARSER_END(CalculatorError)
SKIP :
{
    " "
|   "\r"
|   "\t"
|   "\n"
}
TOKEN:
{
    < NUMBER: (<DIGIT>)+ ( "." (<DIGIT>)+ )? >
|   < DIGIT: ["0"-"9"] >
}
double calculate():
{
    double a;
}
{
    a=expr() <EOF>  { return a; }
}
double expr():
{
    double a;
    double b;
}
{
    a=term()
    (
        "+" b=expr()    { a += b; }
    |   "-" b=expr()    { a -= b; }
    )*
                        { return a; }
}
double term():
{
    double a;
    double b;
}
{
    a=unary()
    (
        "*" b=term()    { a *= b; }
    |   "/" b=term()    { a /= b; }
    )*
                        { return a; }
}
double unary():
{
    double a;
}
{
    "-" a=primary()     { return -a; }
|   a=primary()         { return a; }
}
double primary():
{
    Token t;
    double a;
}
{
    t=<NUMBER>          { return Double.parseDouble(t.toString()); }
|   "(" a=expr() ")"    { return a; }
}
```

通过测试可以发现运算符是右结合的:

```java
@Test
public void testCalculateError() throws ParseException {
    CalculatorError parser = new CalculatorError(new StringReader("1-2-3"));
    Assertions.assertNotEquals(-4,parser.calculate());
}
```


## Calculator.jj

下面是一个四则运算计算器的语法规则:

- 支持加减乘除和算术优先级判断
- 支持负数
- 支持括号

```javacc
options
{
    STATIC=false;
}
PARSER_BEGIN(Calculator)
package io.github.chutian0610.javacc.tutorial02.simple;
public class Calculator
{
}
PARSER_END(Calculator)
SKIP :
{
    " "
|   "\r"
|   "\t"
|   "\n"
}
TOKEN:
{
    < NUMBER: (<DIGIT>)+ ( "." (<DIGIT>)+ )? >
|   < DIGIT: ["0"-"9"] >
}
// 语法入口
double calculate():
{
    double a;
}
{
    a=expr() <EOF>  { return a; }
}
double expr():
{
    double a;
    double b;
}
{
    a=term()   // 递归下降
    (
        "+" b=term()    { a += b; }
    |   "-" b=term()    { a -= b; }
    )*
                        { return a; }
}
double term():
{
    double a;
    double b;
}
{
    a=unary()
    (
        "*" b=unary()    { a *= b; }
    |   "/" b=unary()    { a /= b; }
    )*
                        { return a; }
}
double unary():
{
    double a;
}
{
    "-" a=primary()     { return -a; }
|   a=primary()         { return a; }
}
double primary():
{
    Token t;
    double a;
}
{
    t=<NUMBER>          { return Double.parseDouble(t.toString()); }
|   "(" a=expr() ")"    { return a; }
}
```

可以看到，借助了产生式中的 Java 代码，我们实现了解析完成后，就能输出计算结果。但是注意到每个产生式内部嵌入了Java 代码，语法文件的灵活性较低。



## jjtree

jjtree 可以将语法解析与业务处理解耦，通过生成 AST 提供更灵活、更强大的语言处理能力。

JJTree是JavaCC的一个预处理器，它在JavaCC源代码中的各个位置插入解析树构建操作。JJTree的输出通过JavaCC运行以创建解析器。默认情况下，JJTree生成代码来为语言中的每个非终结符构造解析树节点。可以修改此行为，以便某些非终端不生成节点，或者为产品扩展的一部分生成节点。

- JJTree定义了所有解析树节点都必须实现的Java接口Node。该接口提供了一些操作方法，例如设置节点的父节点，以及添加子节点和检索子节点。
- JJTree以两种模式之一运行，Simple和Multi。在Simple模式下，每个解析树节点都是具体类型SimpleNode+NodeId，在Multi模式下，解析树节点的类型是从节点的名称派生的。如果你不提供节点类的实现，JJTree将为你生成基于SimpleNode的示例实现。
- 虽然JavaCC是一个自顶向下的解析器，但JJTree是自底向上构造解析树的。为了做到这一点，它使用一个堆栈，在节点创建后将其推入堆栈。当它为它们找到父节点时，它从堆栈中弹出子节点并将它们添加到父节点，最后推送新的父节点本身。堆栈是开放的，这意味着您可以从语法操作中访问它：您可以推送，弹出或以其他方式操作其内容。

JJTree为两种基本的节点类型提供了修饰，并提供了一些语法简写以方便使用。

### 确定节点

一个确定的节点是由特定数量的子节点构成的。那么多节点将从堆栈中弹出，并成为新节点的子节点，然后将新节点推送到堆栈本身。

```javacc
#ADefiniteNode(INTEGER EXPRESSION)
```

例如: `#AddNode(2)`

> 确定节点可能会导致 firstToken 和 lastToken 获取异常。

### 条件节点

当且仅当其条件计算结果为true时，条件节点才会使用其节点范围内推到堆栈上的所有子节点来构造。如果它的计算结果为false,，则不构造节点，并且所有子节点都保留在节点堆栈上。

你可以像这样标注一个条件节点：

```javacc
#ConditionalNode(BOOLEAN EXPRESSION)
```
条件节点描述符表达式可以是任何boolean表达式。

条件节点有两种常见的缩写：

- 不定节点: `#IndefiniteNode` =>`#IndefiniteNode(true)`
- 大于节点 `#GTNode(>1)` => `#GTNode(jjtree.arity() > 1)`(出现多次)

默认情况下，JJTree将每个非终结符视为一个不定节点，并从其产品的名称派生节点的名称。您可以使用以下语法为它指定一个不同的名称：

```
void P1() #MyNode : { ... } { ... }
```

当解析器识别出一个P1非终结符时，它开始一个不定节点。它标记了堆栈，因此在扩展中由非终结符在堆栈上创建和推送的任何解析树节点将被弹出并成为节点P1的子节点。如果要禁止为生产创建节点，可以使用以下语法：

```
void P2() #void : { ... } { ... }
```

下面举个例子,不确定节点P3开始，标记堆栈，然后解析P4节点、一个或多个P5节点和P6节点。它们推送的任何节点都会弹出并成为P3的子节点。

```
void P3() : {} {
    P4() ( P5() )+ P6()
}
```

## 基于 jjtree 的计算器

首先给 JavaCC 规则增加上AST节点,。

```javacc
// https://github.com/chutian0610/javacc-demo/blob/master/tutorial02/src/main/jjtree/CalculatorTree.jjt
options
{
    STATIC=false;
    // 决定了生成的语法树节点是否持有解析器对象的引用
    NODE_USES_PARSER=false;
    // 生成 Visitor 接口
    VISITOR=true;
    // 是否为每个语法规则生成独立的节点类。
    MULTI=true;
    // 存储 first token & last token
    TRACK_TOKENS=true;
}
PARSER_BEGIN(CalculatorTree)
package io.github.chutian0610.javacc.tutorial02.tree;
public class CalculatorTree
{

}
PARSER_END(CalculatorTree)
SKIP :
{
    " "
|   "\r"
|   "\t"
|   "\n"
}
TOKEN:
{
    < NUMBER: (<DIGIT>)+ ( "." (<DIGIT>)+ )? >
|   < DIGIT: ["0"-"9"] >
}
void calculate() #StartNode : {}
{
    expr() <EOF>
}
void expr() #void : {}
{
    term()
    (
        "+" term() #AddNode(2)
    |   "-" term() #SubtractNode(2)
    )*
}
void term() #void : {}
{
    unary()
    (
        "*" unary() #MulNode(2)
    |   "/" unary() #DivNode(2)
    )*
}
void unary() #void : {}
{
    "-" primary() #MinusNode
|   primary()
}
void primary() #void: {}
{
    <NUMBER> #Number
|   "(" expr() ")"
}
```

然后实现 [Visitor 方法](https://github.com/chutian0610/javacc-demo/blob/master/tutorial02/src/test/java/io/github/chutian0610/javacc/tutorial02/tree/CalculatorPrintVisitor.java)。下面是打印节点信息的 Visitor。

```java
public class CalculatorPrintVisitor extends CalculatorTreeDefaultVisitor{

    private String sourceCode;

    public CalculatorPrintVisitor(String sourceCode){
        this.sourceCode = sourceCode;
    }

    public Object defaultVisit(SimpleNode node, Object data){
        int d = (int)data;
        System.out.println(node.toString(getPrefix(d)));
        node.childrenAccept(this, d+1);
        return data;
    }

    public Object visit(ASTNumber node, Object data){
        int d = (int)data;
        System.out.println(node.toString(getPrefix(d))+":"+node.jjtGetFirstToken().image);
        return data;
    }

    public String getPrefix(int level){
        StringBuilder s= new StringBuilder();
        for (int i = 0; i < level; i++) {
            s.append(" ");
        }
        return s.toString();
    }
}
```

[对于`1+2+3`会得到如下的 tree](https://github.com/chutian0610/javacc-demo/blob/master/tutorial02/src/test/java/io/github/chutian0610/javacc/tutorial02/tree/CalculatorTest.java)。

```
start
 AddNode
  AddNode
   Number:1
   Number:2
  Number:3
```

基于树结构[计算表达式](https://github.com/chutian0610/javacc-demo/blob/master/tutorial02/src/test/java/io/github/chutian0610/javacc/tutorial02/tree/CalculateVisitor.java):

```java
public class CalculateVisitor implements CalculatorTreeVisitor{
    @Override
    public Double visit(SimpleNode node, Object data) {
        if(node.getClass() ==SimpleNode.class){
           throw new IllegalStateException("Unexpected Node:"+node.toString());
       } 
       return (Double) node.jjtAccept(this,data);
    }
    @Override
    public Double visit(ASTStartNode node, Object data) {
        return visit((SimpleNode) node.jjtGetChild(0),data);

    }
    @Override
    public Double visit(ASTAddNode node, Object data) {
        Double left = visit((SimpleNode) node.jjtGetChild(0),data);
        Double right =visit((SimpleNode) node.jjtGetChild(1),data);
        return left+right;
    }
    @Override
    public Double visit(ASTSubtractNode node, Object data) {
        Double left = visit((SimpleNode) node.jjtGetChild(0),data);
        Double right =visit((SimpleNode) node.jjtGetChild(1),data);
        return left-right;
    }
    @Override
    public Double visit(ASTMulNode node, Object data) {
        Double left = visit((SimpleNode) node.jjtGetChild(0),data);
        Double right =visit((SimpleNode) node.jjtGetChild(1),data);
        return left*right;
    }
    @Override
    public Double visit(ASTDivNode node, Object data) {
        Double left = visit((SimpleNode) node.jjtGetChild(0),data);
        Double right =visit((SimpleNode) node.jjtGetChild(1),data);
        return left/right;
    }
    @Override
    public Double visit(ASTMinusNode node, Object data) {
        Double left = visit((SimpleNode) node.jjtGetChild(0),data);
        return -left;
    }
    @Override
    public Double visit(ASTNumber node, Object data) {
        String number = node.jjtGetFirstToken().image;
        return Double.parseDouble(number);
    }
}
```

### 优化-公共父类

JJTree 生成的 Class 都是继承自SimpleNode。我们可以通过配置为所有的节点通用添加公共抽象父类:

```
option{
    NODE_CLASS="io.github.chutian0610.javacc.tutorial02.tree.v2.BaseNode";
}
```
### 优化-定制节点类

可以实现一个同名的自定义节点类。JJTree 在生成 Class 时会检查同名类是否存在，如果存在就不会生成。自定义节点类中可以增加一些自定义属性。

```
void unary() : {}
{
    "-" primary() {jjtThis.setPositive(false);}
|   primary() {jjtThis.setPositive(true);}
}
```
语法规则可以参考[V2-Calculator](https://github.com/chutian0610/javacc-demo/blob/master/tutorial02/src/main/jjtree/CalculatorTreeV2.jjt)对应的实现类如下:

```java
class AST_unary extends io.github.chutian0610.javacc.tutorial02.tree.v2.BaseNode {
    public AST_unary(int id) {
        super(id);
    }

    protected boolean positive;

    public boolean isPositive() {
        return positive;
    }

    public void setPositive(boolean positive) {
        this.positive = positive;
    }
}
```

## 参考

- [1] [JJTree的参考文档](https://javacc.github.io/javacc/documentation/jjtree.html)

