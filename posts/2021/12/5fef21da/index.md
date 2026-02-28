# 手写Parser-Pratt Parser

解析是编译器将一系列标记转换为树表示的过程：

```
                            Add
                 Parser     / \
"1 + 2 * 3"    ------->   1  Mul
                              / \
                             2   3
```

Pratt Parser解析是手写解析最常用的技术之一。

<!--more-->

## 优先级

如果我们给operator的设置优先级值。

对于不同级别的operator，因为`*`的值更高，所以更倾向于把B和C保持在一起。表达式被解析为 `A + (B * C)`。

```
expr:   A       +       B       *       C
power:      3       3       5       5
```

如果表达式相同。我们可以调高 operator右边的power。

```
expr:   A       +       B       +       C
power:      3       4       3       4
```

我们还可以在两端添加了零，因为没有运算符可以从两侧绑定

```
expr:      A       +       B       +       C
power:  0      3      4        3      4       0
```
就语法树而言，第二个+更喜欢它的右边操作数，所以它倾向于抓住C。 当他这样做的时候，第一个+捕获了A和B，
```
expr:     (A + B)     +     C
power:  0          3    4      0
```

## 构建 Pratt Parser

我们将解析基本原子是单字符数字的表达式，并使用标点符号表示运算符。

```java
public class Token {
    public static Token EOF = new Token(Type.EOF,(char)0);
    public Type type;
    public Character text;

    public Token(Type type, Character text) {
        this.type = type;
        this.text = text;
    }

    public boolean notEOF(){
        return type != Type.EOF;
    }

    @Override
    public String toString()
    {
        return "Token{" + "type=" + type + ", text=" + text + '}';
    }

    public static enum Type{
       Atom,Operator,EOF
   }
}
```

表达式的树结构定义如下，Expr分为Value和Composite两种类型。

```java
public class Expr {
    public Type type;
    public Expr[] exprs;
    public Character text;
    private Expr() {
    }
    public static Expr composite(Character text,Expr... exprs) {
        Expr expr = new Expr();
        expr.type = Type.Composite;
        expr.exprs = exprs;
        expr.text = text;
        return expr;
    }
    public static Expr value(Character text) {
        Expr expr = new Expr();
        expr.type = Type.Value;
        expr.text = text;
        return expr;
    }

    @Override
    public String toString() {
        if(type.equals(Type.Value)){
            return text.toString();
        }else {
            StringBuilder sb = new StringBuilder();
            sb.append("(").append(text);
            for (Expr expr: exprs) {
                sb.append(" ").append(expr.toString());
            }
            sb.append(")");
            return sb.toString();
        }
    }

    public static enum Type{
        Value,Composite
    }
}
```

通过lexer我们可以将字符串转为Token Stream。

```java
public class Lexer
{
    LinkedList<Token> tokenList;

    public Lexer(String expr)
    {
        tokenList = expr.codePoints().mapToObj(c -> (char) c)
                .filter(x -> !Character.isWhitespace(x))
                .map(x -> {
                    if ("0123456789".indexOf(x) != -1) {
                        return new Token(Token.Type.Atom, x);
                    }
                    else {
                        return new Token(Token.Type.Operator, x);
                    }
                })
                .collect(LinkedList::new, LinkedList::add, LinkedList::addAll);
        tokenList.add(Token.EOF);
    }

    public Token next()
    {
        return tokenList.pop();
    }

    public Token peek()
    {
        return tokenList.peek();
    }
}
```

接下来，我们使用parser将Token stream转化为Expr。

```java
public class Parser {
    private Lexer lexer;
    public Parser(Lexer lexer) {
        this.lexer = lexer;
    }
    // 初始时，min_bp = 0
    public Expr expr(int min_bp) {
        // 解析第一个字符
        Expr lhs = exprFirst();
        Token token = null;
        while ((token = lexer.peek()).notEOF()) {
            if (token.type == Token.Type.Operator) {
                // 获取operator的左右优先级
                Pair<Integer, Integer> pair = infix_binding_power(token);
                if(pair == null){
                    throw new RuntimeException("unexpected token"+token);
                }
                if (pair.getLeft() < min_bp) {
                    // 通过min_bp 判断作用域到何处为止
                    break;
                }
                // eat 当前token
                lexer.next();
                // 解析右边节点
                Expr rhs = expr(pair.getRight());
                // 组合为新节点
                lhs = Expr.composite(token.text,lhs,rhs);
            } else {
                throw new RuntimeException("unexpected token");
            }
        }
        return lhs;
    }
    private Expr exprFirst() {
        Expr expr = null;
        Token token = lexer.next();
        if (token.type.equals(Token.Type.Atom)) {
            expr = Expr.value(token.text);
        } else {
            throw new RuntimeException("unexpected token");
        }
        return expr;
    }
    static Map<Character, Pair<Integer, Integer>> infixBindingMap = new HashMap<>();
    static {
        infixBindingMap.put('+', Pair.of(3, 4));
        infixBindingMap.put('-', Pair.of(3, 4));
        infixBindingMap.put('*', Pair.of(5, 6));
        infixBindingMap.put('/', Pair.of(5, 6));
    }
    private Pair<Integer, Integer> infix_binding_power(Token token) {
        Character op = token.text;
        return infixBindingMap.get(op);
    }
}
```

测试一下:

```java
class ParserTest
{
    @Test
    void testParser01(){
        Parser parser = new Parser(new Lexer("1"));
        Assertions.assertEquals("1",parser.expr(0).toString());
    }
    @Test
    void testParser02(){
        Parser parser = new Parser(new Lexer("1 + 2 * 3 "));
        Assertions.assertEquals("(+ 1 (* 2 3))",parser.expr(0).toString());
    }
    @Test
    void testParser03(){
        Parser parser = new Parser(new Lexer("1 + 2 * 3 * 4 + 5 "));
        Assertions.assertEquals("(+ (+ 1 (* (* 2 3) 4)) 5)",parser.expr(0).toString());
    }
}
```

## 扩展Parser

### 右结合的operator

我们在二元operator中增加一个右结合的操作符`=`。

```java
infixBindingMap.put('=', Pair.of(2, 1));
```

注意，运算符更倾向左侧，从而实现右结合。

```java
@Test
void testParser04(){
    Parser parser = new Parser(new Lexer("5 = 1 * 2 + 3 "));
    Assertions.assertEquals("(= 5 (+ (* 1 2) 3))",parser.expr(0).toString());
}
```

### 一元operator

我们在operator中增加2个一元操作符`+`，`-`。

```java
// 设置 一元操作符优先级
static Map<Character, Pair<Integer, Integer>> prefixBindingMap = new HashMap<>();
static {
    // null表示只能绑定右边
    prefixBindingMap.put('+', Pair.of(null, 7));
    prefixBindingMap.put('-', Pair.of(null, 7));
}

private Pair<Integer, Integer> prefix_binding_power(Token token) {
    Character op = token.text;
    return prefixBindingMap.get(op);
}
```

因为第一个Token可能是Atom或是一元操作符，修改exprFirst，增加一元操作符的处理。

```java
private Expr exprFirst() {
    Expr expr = null;
    Token token = lexer.next();
    if (token.type.equals(Token.Type.Atom)) {
        expr = Expr.value(token.text);
    }
    else if (token.type.equals(Token.Type.Operator)){
        Pair<Integer,Integer> pair = prefix_binding_power(token);
        if(pair == null){
          throw new RuntimeException("unexpected token"+token);
        }
        Expr rhs = expr(pair.getRight());
        expr = Expr.composite(token.text,rhs);
    }
    else {
        throw new RuntimeException("unexpected token"+token);
    }
    return expr;
}
```

### 后缀操作符

接下我们添加后缀一元操作符`!`.先设置后缀操作符的优先级，这里后缀操作符的优先级高于前缀操作符。

```java
static Map<Character, Pair<Integer, Integer>> postfixBindingMap = new HashMap<>();
static {
    postfixBindingMap.put('!', Pair.of(9, null));
}
private Pair<Integer, Integer> postfix_binding_power(Token token) {
    Character op = token.text;
    return postfixBindingMap.get(op);
}
```

接下来添加优先级处理逻辑:

```java
public Expr expr(int min_bp) {
  Expr lhs = exprFirst();
  Token token = null;
  while ((token = lexer.peek()).notEOF()) {
      if (token.type == Token.Type.Operator) {
          // 后缀操作符
          Pair<Integer,Integer> post= postfix_binding_power(token);
          if(post !=null){
              if (post.getLeft() < min_bp) {
                  break;
              }
              lexer.next();
              lhs = Expr.composite(token.text, lhs);
              continue;
          }
          // 后缀操作符
          Pair<Integer, Integer> pair = infix_binding_power(token);
          if(pair == null){
              throw new RuntimeException("unexpected token"+token);
          }
          if (pair.getLeft() < min_bp) {
              break;
          }
          lexer.next();
          Expr rhs = expr(pair.getRight());
          lhs = Expr.composite(token.text,lhs,rhs);
      } else {
          throw new RuntimeException("unexpected token:"+token);
      }
  }
  return lhs;
}
```

### 括号表达式

在Parser中添加对"()"的处理。

```java
private Expr exprFirst() {
  Expr expr = null;
  Token token = lexer.next();
  if (token.type.equals(Token.Type.Atom)) {
      expr = Expr.value(token.text);
  }
  else if (token.type.equals(Token.Type.Operator)){
      // 支持对 '(' 的处理
      if(token.text.equals('(')){
          expr = expr(0);
          Token rBracket = lexer.next();
          Preconditions.checkArgument(rBracket.text.equals(')') && rBracket.type.equals(Token.Type.Operator));
          return expr;
      }
      Pair<Integer,Integer> pair = prefix_binding_power(token);
      if(pair == null){
          throw new RuntimeException("unexpected token"+token);
      }
      Expr rhs = expr(pair.getRight());
      expr = Expr.composite(token.text,rhs);
  }
  else {
      throw new RuntimeException("unexpected token"+token);
  }
  return expr;
}
public Expr expr(int min_bp) {
  Expr lhs = exprFirst();
  Token token = null;
  while ((token = lexer.peek()).notEOF()) {
      if (token.type == Token.Type.Operator) {
          Pair<Integer,Integer> post= postfix_binding_power(token);
          if(post !=null){
              if (post.getLeft() < min_bp) {
                  break;
              }
              lexer.next();
              lhs = Expr.composite(token.text, lhs);
              continue;
          }
          Pair<Integer, Integer> pair = infix_binding_power(token);
          // 如果是 ')' pair返回null，此处兼容
          if(pair != null){
              if (pair.getLeft() < min_bp) {
                  break;
              }
              lexer.next();
              Expr rhs = expr(pair.getRight());
              lhs = Expr.composite(token.text,lhs,rhs);
              continue;
          }
          break;

      } else {
          throw new RuntimeException("unexpected token:"+token);
      }
  }
  return lhs;
}
```

### 其他

其他还可以支持的规则有:

- 数组
- 三目符(?:)

> 本文代码见: [github](https://github.com/chutian0610/code-lab/tree/main/snippets/java-snippets/src/main/java/info/victorchu/snippets/compile/pratt)

## 参考资料

- [1] [Simple but Powerful Pratt Parsing](https://matklad.github.io/2020/04/13/simple-but-powerful-pratt-parsing.html)
- [2] [Top Down Operator Precedence](https://tdop.github.io/)

