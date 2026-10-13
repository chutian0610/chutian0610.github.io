# linux命令split

split命令可以将一个大文件分割成很多个小文件，有时需要将文件分割成更小的片段，比如为提高可读性，生成日志等。

<!--more-->

```sh
$ split --help
Usage: split [--help][--version][-b <字节>][-C <字节>][-l <行数>][要切割的文件][输出文件名] [-d] [-a <后缀长度>] 
Output fixed-size pieces of INPUT to PREFIXaa, PREFIXab, ...; default
size is 1000 lines, and default PREFIX is `x'.  With no INPUT, or when INPUT
is -, read standard input.

Mandatory arguments to long options are mandatory for short options too.
  -a, --suffix-length=N   后缀长度,默认是2
  -b, --bytes=SIZE        指定每多少字节切成一个小文件
  -C, --line-bytes=SIZE   指定最多多少字节切分成一个小文件，在切割时将尽量维持每行的完整性
  -d, --numeric-suffixes  使用数字作为后缀，而不是字符
  -l, --lines=NUMBER      指定每多少行切成一个小文件
      --verbose           print a diagnostic just before each
                            output file is opened
      --help     display this help and exit
      --version  output version information and exit

SIZE may be (or may be an integer optionally followed by) one of following:
KB 1000, K 1024, MB 1000*1000, M 1024*1024, and so on for G, T, P, E, Z, Y.
```

举个例子:

```$ ll -h
total 357M
-rwxrwxrwx 1 victor victor 357M May 23 13:46 log
$split -C 100M log sublog -d -a 1
$ ll -h
total 1.1G
-rwxrwxrwx 1 victor victor 357M May 23 14:16 log
-rwxrwxrwx 1 victor victor 100M May 23 14:17 sublog0
-rwxrwxrwx 1 victor victor 100M May 23 14:17 sublog1
-rwxrwxrwx 1 victor victor 100M May 23 14:17 sublog2
-rwxrwxrwx 1 victor victor  57M May 23 14:17 sublog3
```

那么切分好的文件怎么合并？

```sh
cat sublog* > log
```

