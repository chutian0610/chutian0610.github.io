# Java虚拟机-查看本机代码

JVM支持使用PrintAssembly选项来检查JIT编译器生成的代码。对了解代码是如何执行的，以及JIT编译器如何工作和优化代码有很大帮助。

<!--more-->

## PrintAssembly

PrintAssembly使用很简单[<sup>1</sup>](#refer-anchor-1)，只需要在JVM启动参数中添加:

```sh
-XX:+UnlockDiagnosticVMOptions
-XX:+PrintAssembly
## 可以设置输出风格，例如intel
-XX:PrintAssemblyOptions=intel
```

示例代码:

```java
public class PrintAssembly
{
    public static void main(String[] args)
            throws InterruptedException
    {
        List<String> list = new ArrayList<>();
        // 设置较大的循环，用于触发jitC1编译
        for (int i = 0; i < 10_000_000; i++)
        {
            list.add(String.valueOf(i));
        }
        Thread.sleep(1000);
    }
}
```

### hsdis插件

该功能依赖hsdis插件。插件构建方法详见参考[<sup>3</sup>](#refer-anchor-3)。下面简单介绍下macos的构建步骤:

```sh
## !!! 注意如果brew 安装了 gnu binuitls, 不要使用 gnu binuitls

## 下载代码
$ git clone git@gitee.com:mirrors/openjdk.git
## 切换到jvm对应tag
$ git checkout jdk-17+9
$ cd src/utils/hsdis

## Readme 中指定 binutils 2.37
$ wget https://ftp.gnu.org/gnu/binutils/binutils-2.37.tar.gz
$ tar xvf binutils-2.37.tar.gz

$ make BINUTILS=binutils-2.37 ARCH=amd64
## 构建时的问题
## libbfd.a not found : Makefile中 bfd/libbfd.a 改为 bfd/.libs/libbfd.a

#java8
sudo cp build/macosx-amd64/hsdis-amd64.dylib /Library/Java/JavaVirtualMachines/jdk1.8.0_181.jdk/Contents/Home/jre/lib/server/

#java9+
sudo cp build/macosx-amd64/hsdis-amd64.dylib /Library/Java/JavaVirtualMachines/jdk-9.0.4.jdk/Contents/Home/lib/server/
```

### CompileCommand 

CompileCommand 的 print 命令实际上类似于 PrintAssembly JVM 选项。但附加值是能够指定我们要反汇编的确切或前缀名称方法。

```sh
-XX:CompileCommand="print java.util.ArrayList::add"
-XX:CompileCommand="print java.util.ArrayList::*"
```

`java -XX:CompileCommand=help -version` 可以获取命令如何使用，更多的介绍在[官方文档.Java命令行](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html)。

## JITWatch

使用jvm的命令，我们还可以使用[JITWatch](https://github.com/AdoptOpenJDK/jitwatch)。


1. 在java程序添加启动选项(注意安装好hsdis插件):

```sh
-XX:+UnlockDiagnosticVMOptions
## 如果是java8-pre，使用-XX:+TraceClassLoading
-Xlog:class+load=info
-XX:+LogCompilation
-XX:+DebugNonSafepoints
## optional,用于HotSpot 输出反汇编的原生代码
-XX:+PrintAssembly
```

2. 使用jdk 运行 JITWatch

```sh
java -jar jitwatch-ui-<version>-shaded-<OS>-<ARCH>.jar
```

> jit还支持[非GUI模式](https://github.com/AdoptOpenJDK/jitwatch/wiki/Headless-Mode)


## Assembly 日志

我们使用`java.util.ArrayList::add`的机器码(C2)为例:

```sh
## {method} {0x000000012745c208} 'add' '(Ljava/lang/Object;)Z' in 'java/util/ArrayList'
## this:     rsi:rsi   = 'java/util/ArrayList'
## parm0:    rdx:rdx   = 'java/lang/Object'
## [sp+0x40]  (sp of caller)
[Entry Point]
0x00000001149875a0: mov 0x8(%rsi),%r10d
0x00000001149875a4: movabs $0x127000000,%r11
0x00000001149875ae: add %r11,%r10
0x00000001149875b1: cmp %r10,%rax
0x00000001149875b4: jne 0x0000000114445d80  ;   {runtime_call ic_miss_stub}
0x00000001149875ba: xchg %ax,%ax
0x00000001149875bc: nopl 0x0(%rax)
[Verified Entry Point]
0x00000001149875c0: mov %eax,-0x14000(%rsp)
0x00000001149875c7: push %rbp
0x00000001149875c8: sub $0x30,%rsp  ;*synchronization entry
                                    ; - java.util.ArrayList::add@-1 (line 466)
0x00000001149875cc: mov %rdx,0x8(%rsp)
0x00000001149875d1: mov %rsi,(%rsp)
0x00000001149875d5: mov 0x14(%rsi),%r10d  ;*getfield elementData {reexecute=0 rethrow=0 return_oop=0}
                                          ; - java.util.ArrayList::add@13 (line 467)
0x00000001149875d9: incl 0xc(%rsi)  ;*putfield modCount {reexecute=0 rethrow=0 return_oop=0}
                                    ; - java.util.ArrayList::add@7 (line 466)
0x00000001149875dc: nopl 0x0(%rax)
0x00000001149875e0: mov 0xc(%r12,%r10,8),%r11d  ; implicit exception: dispatches to 0x0000000114987765
0x00000001149875e5: mov 0x10(%rsi),%ebp  ;*getfield size {reexecute=0 rethrow=0 return_oop=0}
                                         ; - java.util.ArrayList::add@17 (line 467)
0x00000001149875e8: cmp %r11d,%ebp
0x00000001149875eb: je 0x000000011498768d  ;*if_icmpne {reexecute=0 rethrow=0 return_oop=0}
                                           ; - java.util.ArrayList::add@3 (line 453)
                                           ; - java.util.ArrayList::add@20 (line 467)
0x00000001149875f1: lea (%r12,%r10,8),%r11  ;*aload_2 {reexecute=0 rethrow=0 return_oop=0}
                                            ; - java.util.ArrayList::add@11 (line 455)
                                            ; - java.util.ArrayList::add@20 (line 467)
0x00000001149875f5: mov 0xc(%r11),%r10d
0x00000001149875f9: cmp %r10d,%ebp
0x00000001149875fc: nopl 0x0(%rax)
0x0000000114987600: jae 0x000000011498772c
0x0000000114987606: mov 0x8(%r11),%r10d
0x000000011498760a: cmp $0x15a0,%r10d  ;   {metadata('java/lang/Object'[])}
0x0000000114987611: jne 0x000000011498773c
0x0000000114987617: lea 0x10(%r11,%rbp,4),%rbx
0x000000011498761c: nopl 0x0(%rax)
0x0000000114987620: cmpb $0x0,0x38(%r15)
0x0000000114987625: jne 0x00000001149876b0
0x000000011498762b: mov %rbx,%r10
0x000000011498762e: inc %ebp
0x0000000114987630: mov 0x8(%rsp),%r9
0x0000000114987635: mov %r9,%r8
0x0000000114987638: shr $0x3,%r8
0x000000011498763c: mov %r8d,(%rbx)
0x000000011498763f: mov %r9,%r11
0x0000000114987642: xor %r10,%r11
0x0000000114987645: shr $0x15,%r11
0x0000000114987649: test %r11,%r11
0x000000011498764c: je 0x000000011498766d
0x000000011498764e: test %r9,%r9
0x0000000114987651: je 0x000000011498766d
0x0000000114987653: shr $0x9,%r10
0x0000000114987657: movabs $0x100141000,%rdi
0x0000000114987661: add %r10,%rdi
0x0000000114987664: cmpb $0x4,(%rdi)
0x0000000114987667: jne 0x00000001149876e6  ;*aastore {reexecute=0 rethrow=0 return_oop=0}
                                            ; - java.util.ArrayList::add@14 (line 455)
                                            ; - java.util.ArrayList::add@20 (line 467)
0x000000011498766d: mov (%rsp),%r10
0x0000000114987671: mov %ebp,0x10(%r10)  ;*putfield modCount {reexecute=0 rethrow=0 return_oop=0}
                                         ; - java.util.ArrayList::add@7 (line 466)
0x0000000114987675: mov $0x1,%eax
0x000000011498767a: add $0x30,%rsp
0x000000011498767e: pop %rbp
0x000000011498767f: cmp 0x350(%r15),%rsp  ;   {poll_return} *** SAFEPOINT POLL ***
0x0000000114987686: ja 0x000000011498777d
0x000000011498768c: ret  ;*synchronization entry
                         ; - java.util.ArrayList::add@-1 (line 453)
                         ; - java.util.ArrayList::add@20 (line 467)
0x000000011498768d: xchg %ax,%ax
0x000000011498768f: call 0x0000000114446080  ; ImmutableOopMap {[0]=Oop [8]=Oop }
                                             ;*invokevirtual grow {reexecute=0 rethrow=0 return_oop=1}
                                             ; - java.util.ArrayList::add@7 (line 454)
                                             ; - java.util.ArrayList::add@20 (line 467)
                                             ;   {optimized virtual_call}
0x0000000114987694: mov %rax,%r11
0x0000000114987697: test %rax,%rax
0x000000011498769a: nopw 0x0(%rax,%rax,1)
0x00000001149876a0: jne 0x00000001149875f5  ;*aastore {reexecute=0 rethrow=0 return_oop=0}
                                            ; - java.util.ArrayList::add@14 (line 455)
                                            ; - java.util.ArrayList::add@20 (line 467)
0x00000001149876a6: mov $0xfffffff6,%esi
0x00000001149876ab: call 0x000000011444b600  ; ImmutableOopMap {[8]=Oop }
                                             ;*aastore {reexecute=1 rethrow=0 return_oop=0}
                                             ; - (reexecute) java.util.ArrayList::add@14 (line 455)
                                             ; - java.util.ArrayList::add@20 (line 467)
                                             ;   {runtime_call UncommonTrapBlob}
0x00000001149876b0: mov (%rbx),%r11d
0x00000001149876b3: test %r11d,%r11d
0x00000001149876b6: je 0x000000011498762b
0x00000001149876bc: mov 0x20(%r15),%r10
0x00000001149876c0: mov %r11,%rdi
0x00000001149876c3: shl $0x3,%rdi
0x00000001149876c7: test %r10,%r10
0x00000001149876ca: je 0x000000011498774c
0x00000001149876d0: mov 0x30(%r15),%r11
0x00000001149876d4: mov %rdi,-0x8(%r11,%r10,1)
0x00000001149876d9: add $0xfffffffffffffff8,%r10
0x00000001149876dd: mov %r10,0x20(%r15)
0x00000001149876e1: jmp 0x000000011498762b
0x00000001149876e6: mov 0x40(%r15),%r10
0x00000001149876ea: mov 0x50(%r15),%r11
0x00000001149876ee: lock addl $0x0,-0x40(%rsp)
0x00000001149876f4: cmpb $0x0,(%rdi)
0x00000001149876f7: je 0x000000011498766d
0x00000001149876fd: mov %r12b,(%rdi)
0x0000000114987700: test %r10,%r10
0x0000000114987703: jne 0x000000011498771a
0x0000000114987705: mov %r15,%rsi
0x0000000114987708: movabs $0x104665d60,%r10
0x0000000114987712: call *%r10
0x0000000114987715: jmp 0x000000011498766d
0x000000011498771a: mov %rdi,-0x8(%r11,%r10,1)
0x000000011498771f: add $0xfffffffffffffff8,%r10
0x0000000114987723: mov %r10,0x40(%r15)
0x0000000114987727: jmp 0x000000011498766d  ;*aastore {reexecute=0 rethrow=0 return_oop=0}
                                            ; - java.util.ArrayList::add@14 (line 455)
                                            ; - java.util.ArrayList::add@20 (line 467)
0x000000011498772c: mov $0xffffffe4,%esi
0x0000000114987731: mov %r11,0x10(%rsp)
0x0000000114987736: nop
0x0000000114987737: call 0x000000011444b600  ; ImmutableOopMap {[0]=Oop [8]=Oop [16]=Oop }
                                             ;*aastore {reexecute=1 rethrow=0 return_oop=0}
                                             ; - (reexecute) java.util.ArrayList::add@14 (line 455)
                                             ; - java.util.ArrayList::add@20 (line 467)
                                             ;   {runtime_call UncommonTrapBlob}
0x000000011498773c: mov $0xffffffd6,%esi
0x0000000114987741: mov %r11,0x10(%rsp)
0x0000000114987746: nop
0x0000000114987747: call 0x000000011444b600  ; ImmutableOopMap {[0]=Oop [8]=Oop [16]=Oop }
                                             ;*aastore {reexecute=1 rethrow=0 return_oop=0}
                                             ; - (reexecute) java.util.ArrayList::add@14 (line 455)
                                             ; - java.util.ArrayList::add@20 (line 467)
                                             ;   {runtime_call UncommonTrapBlob}
0x000000011498774c: mov %r15,%rsi
0x000000011498774f: movabs $0x104665d30,%r10
0x0000000114987759: call *%r10  ;*aastore {reexecute=0 rethrow=0 return_oop=0}
                                ; - java.util.ArrayList::add@14 (line 455)
                                ; - java.util.ArrayList::add@20 (line 467)
0x000000011498775c: nopl 0x0(%rax)
0x0000000114987760: jmp 0x000000011498762b
0x0000000114987765: mov $0xfffffff6,%esi
0x000000011498776a: nop
0x000000011498776b: call 0x000000011444b600  ; ImmutableOopMap {}
                                             ;*arraylength {reexecute=0 rethrow=0 return_oop=0}
                                             ; - java.util.ArrayList::add@2 (line 453)
                                             ; - java.util.ArrayList::add@20 (line 467)
                                             ;   {runtime_call UncommonTrapBlob}
0x0000000114987770: mov %rax,%rsi
0x0000000114987773: add $0x30,%rsp
0x0000000114987777: pop %rbp
0x0000000114987778: jmp 0x00000001144f3f00  ;*aastore {reexecute=0 rethrow=0 return_oop=0}
                                            ; - java.util.ArrayList::add@14 (line 455)
                                            ; - java.util.ArrayList::add@20 (line 467)
                                            ;   {runtime_call _rethrow_Java}
0x000000011498777d: movabs $0x11498767f,%r10  ;   {internal_word}
0x0000000114987787: mov %r10,0x368(%r15)
0x000000011498778e: jmp 0x000000011444c700  ;   {runtime_call SafepointBlob}
0x0000000114987793: hlt
0x0000000114987794: hlt
0x0000000114987795: hlt
0x0000000114987796: hlt
0x0000000114987797: hlt
0x0000000114987798: hlt
0x0000000114987799: hlt
0x000000011498779a: hlt
0x000000011498779b: hlt
0x000000011498779c: hlt
0x000000011498779d: hlt
0x000000011498779e: hlt
0x000000011498779f: hlt
[Stub Code]
0x00000001149877a0: movabs $0x0,%rbx  ;   {no_reloc}
0x00000001149877aa: jmp 0x00000001149877aa  ;   {runtime_call}
[Exception Handler]
0x00000001149877af: jmp 0x00000001144e7880  ;   {runtime_call ExceptionBlob}
[Deopt Handler Code]
0x00000001149877b4: call 0x00000001149877b9
0x00000001149877b9: subq $0x5,(%rsp)
0x00000001149877be: jmp 0x000000011444b9a0  ;   {runtime_call DeoptimizationBlob}
0x00000001149877c3: hlt
0x00000001149877c4: hlt
0x00000001149877c5: hlt
0x00000001149877c6: hlt
0x00000001149877c7: hlt
```

### 元信息

```sh
## {method} {0x000000012745c208} 'add' '(Ljava/lang/Object;)Z' in 'java/util/ArrayList'
## this:     rsi:rsi   = 'java/util/ArrayList'
## parm0:    rdx:rdx   = 'java/lang/Object'
## [sp+0x40]  (sp of caller)
[Entry Point]
```

- 第一行是反汇编的方法的名称add和签名：一个类型Object的参数，返回一个boolean，在类java. util.ArrayList中
- 由于这是一个实例方法，因此实际上有2个参数:
  - 参数this存储在寄存器rdx中
  - Object参数存储在寄存器r8中。

### 验证的入口点

```sh
[Entry Point]
0x00000001149875a0: mov 0x8(%rsi),%r10d
0x00000001149875a4: movabs $0x127000000,%r11
0x00000001149875ae: add %r11,%r10
0x00000001149875b1: cmp %r10,%rax
0x00000001149875b4: jne 0x0000000114445d80  ;   {runtime_call ic_miss_stub}
0x00000001149875ba: xchg %ax,%ax
0x00000001149875bc: nopl 0x0(%rax)
[Verified Entry Point]
```

方法的第一个说明在"Verified entry point"部分之后开始。在此标记之前的机器码用于对齐（填充）。接下来我们将查看分号之后的注释。以星号`*`开头的注释表示相关的字节码。

### 同步条目

```sh
0x00000001149875c0: mov %eax,-0x14000(%rsp)
0x00000001149875c7: push %rbp
0x00000001149875c8: sub $0x30,%rsp  ;*synchronization entry
                                    ; - java.util.ArrayList::add@-1 (line 466)
```

注释`; - java.util.ArrayList::add@-1 (line 466)`为我们提供了Java源代码的映射(类名、方法名和方法的字节码偏移量，最后是原始Java源文件的行号)。对于这个开始，因为我们没有关联特定的字节码，所以我们得到了-1偏移量。对于第一个`;*synchronization entry`，它表示函数的开始：准备执行所需的一些指令(堆栈分配，保存一些寄存器，…).

### 获取字段

```sh
0x00000001149875cc: mov %rdx,0x8(%rsp)
0x00000001149875d1: mov %rsi,(%rsp)
0x00000001149875d5: mov 0x14(%rsi),%r10d  ;*getfield elementData {reexecute=0 rethrow=0 return_oop=0}
                                          ; - java.util.ArrayList::add@13 (line 467)
```

注释从当前实例获取字段elementData

### 隐式空校验

```sh
0x00000001149875e0: mov 0xc(%r12,%r10,8),%r11d  ; implicit exception: dispatches to 0x0000000114987765
```

这里我们有一个隐式的null检查，因为我们正在引用对象数组elementData以获取它的长度（Java代码：elementData.length）。如果elementData为空，在这种情况下，JVM必须抛出一个NullPointerException

### 类型检查

```sh
0x000000011498760a: cmp $0x15a0,%r10d  ;   {metadata('java/lang/Object'[])}
```

我们正在验证当前实例elementData类（元数据）是一个对象数组（'java/lang/Object'[]）。为了执行此操作，我们从实例中获取类指针，并将其与JVM加载的类的地址进行比较。

### 安全点

```sh
0x000000011498767f: cmp 0x350(%r15),%rsp  ;   {poll_return} *** SAFEPOINT POLL ***
```

注释`{poll_return}`表示指令执行安全点检查。

## 参考

- [1] [openjdk.PrintAssembly](https://wiki.openjdk.org/display/HotSpot/PrintAssembly)

- [2] [PrintAssembly output explained](https://jpbempel.github.io/2015/12/30/printassembly-output-explained.html)

<div id="refer-anchor-3"></div>

- [3] [How to build hsdis](https://github.com/AdoptOpenJDK/jitwatch/wiki/Building-hsdis)

- [4] [JVM CompileCommand](https://jpbempel.github.io/2016/03/16/compilecommand-jvm-option.html)

