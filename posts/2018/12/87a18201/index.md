# RFC4180:CSV

## CSV 格式的定义

1. 每条记录都位于一条单独的行上，由行分隔符\(CRLF\)分开。例如：

```
aaa,bbb,ccc CRLF
zzz,yyy,xxx CRLF
```

2. 文件中的最后一条记录可能有也可能没有结束符标志，例如：

```
aaa,bbb,ccc CRLF
zzz,yyy,xxx
```

3. 第一行可能存在标题行，包含记录字段的对应名称，标题行的标题数与记录字段数相同。例如:

```
field_name,field_name,field_name CRLF
aaa,bbb,ccc CRLF
zzz,yyy,xxx CRLF
```

4. 在标题和记录中，可能有一个或多个字段，使用`,`号分隔，最后一个字段记录后面不能有分隔符。例如:

```
AAA,BBB,CCC
```

5. 每个字段可能包含在双引号，也可能不包含在内\(microsoft excel 没有使用双引号\)。如果字段没有用双引号括起来，那么双引号可能不会出现在字段内。例如：

```
"aaa","bbb","ccc"CRLF
ZZZ,YYY,XXX
```

6. 如果字段包含换行符\(CRLF\)，双引号，和逗号，那么字段值应该被包含在双引号中，例如:

```
"aaa","b CRLFbb","ccc" CRLF
zzz,yyy,xxx
```

7. 如果使用双引号括起字段，要在一个字段内使用双引号必须通过在前面添加另一个双引号的方式进行转义。例如:

```
"aaa","b""bb","ccc"
```

## ABNF 语法

```
file = [header CRLF] record *(CRLF record) [CRLF]
header = name *(COMMA name)
record = field *(COMMA field)
name = field
field = (escaped / non-escaped)
escaped = DQUOTE *(TEXTDATA / COMMA / CR / LF / 2DQUOTE) DQUOTE
non-escaped = *TEXTDATA
COMMA = %x2C
CR = %x0D ;as per section 6.1 of RFC 2234 [2]
DQUOTE =  %x22 ;as per section 6.1 of RFC 2234 [2]
LF = %x0A ;as per section 6.1 of RFC 2234 [2]
CRLF = CR LF ;as per section 6.1 of RFC 2234 [2]
TEXTDATA =  %x20-21 / %x23-2B / %x2D-7E
```

