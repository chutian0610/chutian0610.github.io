# linux命令Tee

Linux tee命令用于读取标准输入的数据，并将其内容输出成文件。

<!--more-->

## tldr

```sh
## Copy standard input to each file, and also to standard output:
echo "example" | tee path/to/file

## Append to the given files, do not overwrite:
echo "example" | tee -a path/to/file

## Print standard input to the terminal, and also pipe it into another program for further processing:
echo "example" | tee /dev/tty | xargs printf "[%s]"

## Create a directory called "example", count the number of characters in "example" and write "example" to the terminal:
echo "example" | tee >(xargs mkdir) >(wc -c)
```

## usage

tee指令会从标准输入设备读取数据，将其内容输出到标准输出设备，同时保存成文件。 语法如下:

```sh
$ tee --help
Usage: tee [OPTION]... [FILE]...
Copy standard input to each FILE, and also to standard output.

  -a, --append              附加到既有文件的后面，而非覆盖它．
  -i, --ignore-interrupts   忽略中断信号。
  -p                        diagnose errors writing to non pipes
      --output-error[=MODE]   set behavior on write error.  See MODE below
      --help     display this help and exit
      --version  output version information and exit

MODE determines behavior with write errors on the outputs:
  'warn'         diagnose errors writing to any output
  'warn-nopipe'  diagnose errors writing to any output not a pipe
  'exit'         exit on error writing to any output
  'exit-nopipe'  exit on error writing to any output not a pipe
The default MODE for the -p option is 'warn-nopipe'.
The default operation when --output-error is not specified, is to
exit immediately on error writing to a pipe, and diagnose errors
writing to non pipe outputs.
```

