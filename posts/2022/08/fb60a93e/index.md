# AntlrV4的内存泄漏问题

在使用antlr生成的语法解析器处理多个文件后，JVM 最终会产生内存不足异常,[PredictionContextCache](https://github.com/antlr/antlr4/blob/master/runtime/Java/src/org/antlr/v4/runtime/atn/PredictionContextCache.java)中的hashMap和DFA数组_decisionToDFA会不断增长。因为 PredictionContextCache和_decisionToDFA在生成Parser和Lexer中是类共享的。

```java
public class XXXParser extends Parser {
	protected static final DFA[] _decisionToDFA;
	protected static final PredictionContextCache _sharedContextCache =
		new PredictionContextCache();
  // ...
}
```

<!--more-->

## 修复方案

```java
/**
 * fix antlr memory leak
 * @see <a href="https://github.com/antlr/antlr4/issues/499"> Memory Leak </a>
 * @author victorchu
 * @date 2022/8/8 11:29
 */
import org.antlr.v4.runtime.Lexer;
import org.antlr.v4.runtime.Parser;
import org.antlr.v4.runtime.atn.ATN;
import org.antlr.v4.runtime.atn.LexerATNSimulator;
import org.antlr.v4.runtime.atn.ParserATNSimulator;
import org.antlr.v4.runtime.atn.PredictionContextCache;
import org.antlr.v4.runtime.dfa.DFA;

import java.util.concurrent.atomic.AtomicReference;
import java.util.function.BiConsumer;

import static com.google.common.base.Preconditions.checkArgument;
import static java.util.Objects.requireNonNull;

public class RefreshableParserInitializer<L extends Lexer, P extends Parser>
        implements BiConsumer<L, P>
{
    private final AtomicReference<ParserAndLexerATNCaches<L, P>> caches;

    public RefreshableParserInitializer()
    {
        caches = new AtomicReference<>();
    }

    private ParserAndLexerATNCaches<L, P> buildATNCache(L l, P p)
    {
        ATN lexerATN = l.getATN();
        ATN parserATN = p.getATN();
        return new ParserAndLexerATNCaches(new AntlrATNCacheFields(lexerATN), new AntlrATNCacheFields(parserATN));
    }
    // ================= 泛型处理 =====================

    @Override
    public void accept(L l, P p)
    {

        ParserAndLexerATNCaches<L, P> newCache = buildATNCache(l, p);
        ParserAndLexerATNCaches<L, P> oldCaches = caches.getAndSet(newCache);
        newCache.config(l,p);
    }

    private static final class ParserAndLexerATNCaches<L extends Lexer, P extends Parser>
    {
        public ParserAndLexerATNCaches(AntlrATNCacheFields lexer, AntlrATNCacheFields parser)
        {
            this.lexer = lexer;
            this.parser = parser;
        }

        public final AntlrATNCacheFields lexer;
        public final AntlrATNCacheFields parser;

        public void config(L l, P p){
            lexer.configureLexer(l);
            parser.configureParser(p);
        }
    }

    public static final class AntlrATNCacheFields
    {
        private final ATN atn;
        private final PredictionContextCache predictionContextCache;
        private final DFA[] decisionToDFA;

        public AntlrATNCacheFields(ATN atn)
        {
            this.atn = requireNonNull(atn, "atn is null");
            this.predictionContextCache = new PredictionContextCache();
            this.decisionToDFA = createDecisionToDFA(atn);
        }

        @SuppressWarnings("ObjectEquality")
        public void configureLexer(Lexer lexer)
        {
            requireNonNull(lexer, "lexer is null");
            // Intentional identity equals comparison
            checkArgument(atn == lexer.getATN(), "Lexer ATN mismatch: expected %s, found %s", atn, lexer.getATN());
            lexer.setInterpreter(new LexerATNSimulator(lexer, atn, decisionToDFA, predictionContextCache));
        }

        @SuppressWarnings("ObjectEquality")
        public void configureParser(Parser parser)
        {
            requireNonNull(parser, "parser is null");
            // Intentional identity equals comparison
            checkArgument(atn == parser.getATN(), "Parser ATN mismatch: expected %s, found %s", atn, parser.getATN());
            parser.setInterpreter(new ParserATNSimulator(parser, atn, decisionToDFA, predictionContextCache));
        }

        private static DFA[] createDecisionToDFA(ATN atn)
        {
            DFA[] decisionToDFA = new DFA[atn.getNumberOfDecisions()];
            for (int i = 0; i < decisionToDFA.length; i++) {
                decisionToDFA[i] = new DFA(atn.getDecisionState(i), i);
            }
            return decisionToDFA;
        }
    }
}
```

如何使用RefreshableParserInitializer？

```java
 SqlBaseLexer baseLexer = xxx;
 SqlBaseParser baseParser = xxx;

 // 注意，每次解析前调用
 BiConsumer<SqlBaseLexer, SqlBaseParser> initializer = new RefreshableParserInitializer<SqlBaseLexer, SqlBaseParser>(){};
 initializer.accept(baseLexer,baseParser);
```

> 由于 cache有助于提升 Parser的性能，应该选择定时，按照运行次数或者是内存占用来间断地淘汰缓存。  
> 参考代码见[AntlrCacheManager - chutian0610's github](https://github.com/chutian0610/code-lab/blob/main/snippets/java-snippets/src/main/java/info/victorchu/snippets/antlr/AntlrCacheManager.java)


## 资料

- [1] [Memory Leak](https://github.com/antlrjavaparser/antlr-java-parser/issues/5)
- [2] [Memory Leak in PredictionContextCache](https://github.com/antlr/antlr4/issues/499)
- [3] [Add SqlBaseParser initializer hook and refreshable ATN caches](https://github.com/prestodb/presto/pull/14317)

