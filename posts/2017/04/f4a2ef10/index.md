# Java虚拟机-Instrumentation

java Instrumentation(插桩)指的是可以用独立于应用程序之外的代理（agent）程序来监测和协助运行在JVM上的应用程序。这种监测和协助包括但不限于获取JVM运行时状态，替换和修改类定义等。 

从Java SE5开始使用JVM TI替代了JVM PI和JVM DI。提供一套代理机制，支持独立于JVM应用程序之外的程序以代理的方式连接和访问JVM。java.lang.instrument是在JVM TI的基础上提供的Java版本的实现。

Instrument 设计的目的是将字节码添加到方法中，收集工具使用的数据。出于这个目的，更改纯粹是附加的，因此这些工具不会修改应用程序状态或行为。此类良性工具的示例包括监控代理、分析器、覆盖分析器和事件记录器。

当然，由于这种检测机制提供了修改现有已编译 Java 类的字节码或添加字节码的能力，所以我们也可以用来动态修改运行的程序代码。

<!--more-->

## 获取 Agent

首先我们将介绍下如何获取一个Agent。agent获取分为两种方式:

- premain 静态加载: 将在JVM启动时使用-javaagent参数静态加载代理
- agentmain 动态加载: 使用Jvm Attach API将动态加载代理到JVM

### premain(since JDK 1.5)

在Java SE5时代，Instrument只提供了premain一种方式，即在真正的应用程序（包含main方法的程序）main方法启动前启动一个代理程序。例如使用如下命令：

```sh
## 注意，始终将-javaagent参数放在-jar参数之前
## 后面的agent将不会执行
java -javaagent:agent_jar_path[=options] java_app_name
```

可以在启动名为java_app_name的应用之前启动一个agent_jar_path指定位置的agent jar。 代理 (agent) 是在你的main方法前的一个拦截器 (interceptor)，也就是在main方法执行之前，执行agent的代码。 agent的代码与你的main方法在同一个JVM中运行，并被同一个system classloader装载，被同一的安全策略 (security policy)和上下文 (context)所管理。 实现这样一个agent jar包，必须满足下面两个条件：

- 在这个jar包的manifest文件中包含Premain-Class属性，并且改属性的值为代理类全路径名。
- 代理类必须提供一个public static void premain(String args, Instrumentation inst)或 public static void premain(String args) 方法。

当在命令行启动该代理jar时，VM会根据manifest中指定的代理类，使用于main类相同的系统类加载器加载代理类。在执行main方法前执行premain()方法。如果premain(String args, Instrumentation inst)和premain(String args)同时存在时，**优先使用前者**。

一个java程序中-javaagent这个参数的个数是没有限制的，所以可以添加任意多个java agent。所有的java agent会按照你定义的顺序执行。 例如：

```sh
java -javaagent:MyAgent1.jar -javaagent:MyAgent2.jar -jar MyProgram.jar
```

假设MyProgram.jar里面的main函数在MyProgram中。MyAgent1.jar, MyAgent2.jar, 这2个jar包中实现了premain的类分别是MyAgent1, MyAgent2 程序执行的顺序将会是：

```sh
MyAgent1.premain -> MyAgent2.premain -> MyProgram.main
```

每一个java agent 都可以接收一个字符串类型的参数，也就是premain中的agentArgs，这个agentArgs是通过java option中定义的。例如：

```sh
java -javaagent:MyAgent2.jar=thisIsAgentArgs -jar MyProgram.jar
```
> MyAgent2中premain接收到的agentArgs的值将是 `thisIsAgentArgs`

### agentmain (since JDK 1.6)

由于premain必须在命令行指定代理jar，并且代理类必须在main方法前启动。因此，要求开发者在应用前就必须确认代理的处理逻辑和参数内容等等，在有些场合下，这是比较困难的。

比如正常的生产环境下，一般不会开启代理功能，但是在发生问题时，我们不希望停止应用就能够动态的去修改一些类的行为，以帮助排查问题，这在应用启动前是无法确定的。为解决运行时启动代理类的问题，Java SE1.6开始，提供了在应用程序的VM启动后在动态添加代理的方式，即agentmain方式。 

与Permain类似，agent方式同样需要提供一个agent jar，并且这个jar需要满足：

* 在manifest中指定Agent-Class属性，值为代理类全路径
* 代理类需要提供`public static void agentmain(String args, Instrumentation inst)`或`public static void agentmain(String args)`方法。并且再二者同时存在时以前者优先。

跟 premain 不同的是，agentmain 需要在 main 函数开始运行后才启动，JDK6中提供了Attach API(Java Tools API) 用于将代理加载到应用程序。使用示例如下:

```java
VirtualMachine jvm = VirtualMachine.attach(jvmPid);
jvm.loadAgent(agentFile.getAbsolutePath());
jvm.detach();
```

## 字节码修改

要修改已有类的代码，我们主要是通过传入的 Instrumentation 实例注册 ClassFileTransformer 转换器来实现。在转换器实现内我们通过类名和类加载器共同来判断是否是我们需要修改的类，然后去修改字节码，可以使用一些类库轻松实现字节码的修改。如果我们注册的是具有重新转换能力的转换器，则可以使用 retransformClasses 立即修改转换类，否则会在类定义、加载或重新定义时调用。


### Instrumentation

在上面列出的方法签名的参数中，不管是静态加载的 premain 还是动态加载的 agentmain，都传递了一个 Instrumentation 实例给我们。一旦我们的代理获得了 Instrumentation 实例，就可以随时调用该实例上的方法。该实例主要包含以下方法：

|方法名|描述|
|:---|:---|
|addTransformer(ClassFileTransformer transformer)|注册提供的转换器。|
|addTransformer(ClassFileTransformer transformer, boolean canRetransform)|注册提供的转换器，当 canRetransform 为 true 时，代表该转换器具有重新转换能力。|
|removeTransformer(ClassFileTransformer transformer)|取消注册提供的转换器。|
|getAllLoadedClasses()|返回 JVM 当前加载的所有类的数组。|
|getInitiatedClasses(ClassLoader loader)|返回 loader 启动加载器所有类的数组。|
|isModifiableClass(Class theClass)|测试一个类是否可以通过 retransformation 或 redefinition 来修改。|
|isRetransformClassesSupported()|返回当前 JVM 配置是否支持类的重新转换。|
|retransformClasses(Class… classes)|重新转换提供的类集合。|
|isRedefineClassesSupported()|返回当前 JVM 配置是否支持重新定义类。|
|redefineClasses(ClassDefinition… definitions)|使用给定的类文件重新定义提供的类集合。|
|isModifiableModule(Module module)|测试是否可以使用 redefineModule 修改模块。|
|redefineModule(..)	|重新定义模块以扩展它读取的模块集、它导出或打开的包集或其使用或提供的服务。|
|getObjectSize(Object objectToSize)	|返回指定对象消耗的存储量的特定于实现的近似值。|
|appendToBootstrapClassLoaderSearch(JarFile jarfile)|指定一个 JAR 文件，其中包含要由引导程序类加载器定义的检测类。|
|appendToSystemClassLoaderSearch(JarFile jarfile)|指定一个 JAR 文件，其中包含要由系统类加载器定义的检测类。|

### ClassFileTransformer

如果我们要转换一个类，则可以通过注册一个自定义的 ClassFileTransformer 来实现，Java 虚拟机会在加载、重新定义或重新转换类时调用该实例的 transform 方法。转换器是在 Java 虚拟机定义类之前被调用。

ClassFileTransformer 可以实现为具有重新转换能力的转换器，通过在注册时将 canRetransform 参数传入为 true 告诉注册器自己有重新转换的能力。

一旦使用 addTransformer 注册了转换器，将在每个新的类定义时和每个类重新定义时调用转换器。在每个类重新转换时，也将调用具有重新转换能力的转换器。

- 对新类定义的请求是使用 ClassLoader.defineClass 或其原生等效方法进行的。
- 类重新定义的请求是使用 Instrumentation.redefineClasses 或其原生等效方法进行的。
- 类重新转换的请求是使用 Instrumentation.retransformClasses 或其原生等效方法进行的。

在处理请求期间，转换器是在类的文件字节被验证和应用之前调用。当有多个转换器时，转换操作是通过转换器调用链组成的。也就是说，一次调用返回的字节数组成为下一次调用的输入（通过 classfileBuffer 参数）。转换器将按以下顺序生效：

- 没有再转换能力的转换器
- 没有再转换能力的原生转换器
- 有再转换能力的转换器
- 有再转换能力的原生转换器

对于再转换，不调用没有再转换能力的转换器，而是重用前一次转换的结果。其他情况，该转换方式始终被调用。在这些所有分组中，转换器都是按照注册的顺序被调用。本机转换器由 Java 虚拟机工具接口（JVMTI）中的 ClassFileLoadHook 事件提供）。传给第一个转换器的输入（classfileBuffer 参数）是：

- 对于新的类定义，传递其 ClassLoader.defineClass 的字节
- 对于类重新定义，是 Instrumentation.redefineClasses 时传入的参数 ClassDefinition 实例的 getDefinitionClassFile() 返回结果
- 对于类重新转换，传递的是新类定义的字节。或者，如果是重新定义，则是最后一次重新定义的字节。

如果实现方法确定不需要转换，则应**返回 null**。否则，它应该创建一个新的 byte[] 数组，将输入的 classfileBuffer 连同所有所需的转换复制到其中，并返回新数组。不得修改输入的 classfileBuffer。

在重新转换和重新定义的情况下，转换器必须支持重新定义语义：如果转换器在初始定义期间更改的类后来被重新转换或重新定义，则转换器必须确保第二个类输出类文件是第一个输出类文件的合法重新定义。

如果转换器抛出异常（它没有捕获），后续转换器仍将被调用，并且仍将尝试加载、重新定义或重新转换。因此，抛出异常与返回 null 具有相同的效果。为了防止在转换器代码中生成未经检查的异常时出现意外行为，转换器可以捕获 Throwable。如果转换器认为 classFileBuffer 不代表有效格式化的类文件，它应该抛出一个 IllegalClassFormatException；虽然这与返回 null 具有相同的效果。它有助于记录或调试格式损坏。

## demo

demo项目地址: [chutian0610/code-lab](https://github.com/chutian0610/code-lab/tree/main/demos/java-instrumentation)

java instrumentation 示例程序由两个模块组成:

- 一个应用程序 app
- 一个jvm代理 agent
jvm 代理将展示如何通过 instrumentation api 修改app的字节码。

```sh
.
├── agent # agent
├── app # 应用程序 
└── pom.xml
```

### attach maven依赖

对于JDK1.8中找不到依赖的问题，可以通过system依赖的方式解决(idea 中需手动添加Jar路径依赖)。

```xml
<dependency>
    <groupId>com.sun</groupId>
    <artifactId>tools</artifactId>
    <version>1.8</version>
    <scope>system</scope>
    <systemPath>${java.home}/../lib/tools.jar</systemPath>
</dependency>
```

### premain 运行

```sh
## cd demos/java-instrumentation/app/target;
dir=$(pwd);
java -javaagent:"$dir/lib/agent-1.0-SNAPSHOT.jar" -cp $dir/app-1.0-SNAPSHOT.jar:$dir/lib info.victorchu.demos.javainst.Application
```

```log
[Agent] premain method set Instrumentation
[Agent] Transforming class info/victorchu/demos/javainst/Application
Enter method-> info/victorchu/demos/javainst/Application.main
Enter method-> info/victorchu/demos/javainst/Application.count
Enter method-> info/victorchu/demos/javainst/Application.count
Enter method-> info/victorchu/demos/javainst/Application.count
Enter method-> info/victorchu/demos/javainst/Application.count
```

### agentmain 运行

启动app

```sh
## cd demos/java-instrumentation/app/target;
dir=$(pwd);
java -cp $dir/app-1.0-SNAPSHOT.jar:$dir/lib info.victorchu.demos.javainst.Application
```

启动attach程序

```sh
cd demos/java-instrumentation/app/target;
dir=$(pwd);
java -cp $dir/app-1.0-SNAPSHOT.jar:$dir/lib info.victorchu.demos.javainst.Application $pid $dir/lib/agent-1.0-SNAPSHOT.jar
```

可以看到在attach成功后，类被transform了。

```log
count 10 times
pid=1421
count 10 times
pid=1421
count 10 times
pid=1421
[Agent] In agentmain method
[Agent] Transforming class info/victorchu/demos/javainst/Application
Enter method-> info/victorchu/demos/javainst/Application.count
Enter method-> info/victorchu/demos/javainst/Application.count
Enter method-> info/victorchu/demos/javainst/Application.count
Enter method-> info/victorchu/demos/javainst/Application.count
```

## 内存大小计算

使用java.lang.instrument.Instrumentation.getObjectSize()方法，可以很方便的计算任何一个运行时对象的大小(Shadow heap size)，返回该对象本身在内存中的大小。其原理是通过反射，累加计算 Retained heap size。

参考代码项目地址: [chutian0610/code-lab: MemoryUtil.java](https://github.com/chutian0610/code-lab/blob/main/demos/java-instrumentation/agent/src/main/java/info/victorchu/demos/javainst/MemoryUtil.java)

```java
public class MemoryUtil {
    public static long memoryUsageOf(Object obj){
        return InstrumentationHelper.getInstrumentation().getObjectSize(obj);
    }

    public static long deepMemoryUsageOf(Collection<? extends Object> objs){
        long total = 0L;
        Set<Integer> counted = new HashSet(objs.size() * 4);
        for (Object o : objs) {
            total += deepMemoryUsageOfSingle(o,counted);
        }
        return total;
    }
    public static long deepMemoryUsageOf(Object obj){
        return deepMemoryUsageOfSingle(obj,new HashSet());
    }
    private static long deepMemoryUsageOfSingle(Object objP,Set<Integer> visited){
        Deque<Object> toBeQueue = new ArrayDeque<>();
        toBeQueue.add(objP);
        long size = 0L;
        while (toBeQueue.size() > 0) {
            Object obj = toBeQueue.poll();
            size += skipObject(visited, obj) ? 0L : memoryUsageOf(obj);
            Class<?> tmpObjClass = obj.getClass();
            if (tmpObjClass.isArray()) {
                // [I , [F 基本类型名字长度是2
                if (tmpObjClass.getName().length() > 2) {
                    for (int i = 0, len = Array.getLength(obj); i < len; i++) {
                        Object tmp = Array.get(obj, i);
                        if (tmp != null) {
                            // 非基本类型需要深度遍历其对象
                            toBeQueue.add(Array.get(obj, i));
                        }
                    }
                }
            } else {
                while (tmpObjClass != null) {
                    Field[] fields = tmpObjClass.getDeclaredFields();
                    for (Field field : fields) {
                        if (Modifier.isStatic(field.getModifiers()) // 静态变量不计
                                || field.getType().isPrimitive() // 基本类型不重复计
                                || field.getName().startsWith("this") // 内部类实例对外部类实例的引用不再重复计算
                        ) {
                            continue;
                        }
                        field.setAccessible(true);
                        Object fieldValue = null;
                        try {
                            fieldValue = field.get(obj);
                            if (fieldValue == null) {
                                continue;
                            }
                            toBeQueue.add(fieldValue);
                        } catch (IllegalAccessException e) {
                            throw new InternalError("Couldn't read " + field);
                        }
                    }
                    tmpObjClass = tmpObjClass.getSuperclass();
                }
            }
        }
        return size;
    }

    static boolean skipObject(Set<Integer> visited, Object obj) {
        return !visited.add(System.identityHashCode(obj));
    }
}
```

## 附录

### Manifest Attributes

|属性|描述|
|:---|:---|
Premain-Class|如果 JVM 启动时指定了代理，那么此属性指定代理类，即包含 premain 方法的类。如果 JVM 启动时指定了代理，那么此属性是必需的。如果该属性不存在，那么 JVM 将中止。注：此属性是类名，不是文件名或路径。| 
|Agent-Class|如果实现支持 VM 启动之后某一时刻启动代理的机制，那么此属性指定代理类。 即包含 agentmain 方法的类。 此属性是必需的，如果不存在，代理将无法启动。注：这是类名，而不是文件名或路径。|
|Launcher-Agent-Class|如果实现支持将应用程序作为可执行 JAR 启动的机制，则主清单可能包含此属性，以指定要在调用应用程序主方法之前启动的代理的类名。|
|Boot-Class-Path|设置引导类加载器搜索的路径列表。路径表示目录或库（在许多平台上通常作为 JAR 或 zip 库被引用）。查找类的特定于平台的机制失败后，引导类加载器会搜索这些路径。按列出的顺序搜索路径。列表中的路径由一个或多个空格分开。路径使用分层 URI 的路径组件语法。如果该路径以斜杠字符（“/”）开头，则为绝对路径，否则为相对路径。相对路径根据代理 JAR 文件的绝对路径解析。忽略格式不正确的路径和不存在的路径。如果代理是在 VM 启动之后某一时刻启动的，则忽略不表示 JAR 文件的路径。此属性是可选的。 |
|Can-Redefine-Classes|布尔值（true 或 false，与大小写无关）。是否能重定义此代理所需的类。true 以外的值均被视为 false。此属性是可选的，默认值为 false。 |
|Can-Retransform-Classes|布尔值（true 或 false，与大小写无关）。是否能重转换此代理所需的类。true 以外的值均被视为 false。此属性是可选的，默认值为 false。|
|Can-Set-Native-Method-Prefix|布尔值（true 或 false，与大小写无关）。是否能设置此代理所需的本机方法前缀。true 以外的值均被视为 false。此属性是可选的，默认值为 false。|

代理 JAR 文件可能在清单中同时具有 Premain-Class 和 Agent-Class 属性。当使用 -javaagent 选项在命令行上启动代理时，Premain-Class 属性指定代理类的名称，而 Agent-Class 属性将被忽略。同样，如果代理在 VM 启动后的某个时间启动，则 Agent-Class 属性指定代理类的名称（忽略 Premain-Class 属性的值）。

### 模块化的注意点

从代理 JAR 文件加载的类由系统类加载器加载，并且是系统类加载器的未命名模块的成员。系统类加载器通常也定义包含应用程序 main 方法的类。代理类可见的类是系统类加载器可见的类，至少包括：

- 引导层模块导出的包中的类。引导层是否包含所有平台模块将取决于初始模块或应用程序的启动方式。
- 可以由系统类加载器（通常是类路径）定义为未命名模块成员的类。
- 代理安排由引导类加载器定义的任何类，作为其未命名模块的成员。

如果代理类需要链接到不在引导层中的平台（或其他）模块中的类，那么应用程序可能需要在启动确保这些模块在引导层中。例如，在 JDK 实现中，`--add-modules` 命令行选项可用于将模块添加到要在启动时解析的根模块集中。

支持安排由引导类加载器加载的代理类（通过 appendToBootstrapClassLoaderSearch 或上面指定的 Boot-Class-Path 属性），必须只链接到定义到引导类加载器的类。不能保证所有平台类都可以由引导类加载器定义。

如果配置了自定义系统类加载器（通过 getSystemClassLoader 方法中指定的系统属性 java.system.class.loader），则它必须定义 appendToSystemClassLoaderSearch 中指定的 appendToClassPathForInstrumentation 方法。 换句话说，自定义系统类加载器必须支持将代理 JAR 文件添加到系统类加载器搜索的机制。

### 字节码lib

除了 JVM Instrumentation方式，还有很多其他 lib 支持对 Java 字节码的修改:

- [bytebuddy](http://bytebuddy.net/)：Byte Buddy 是一个代码生成和操作库，用于在 Java 应用程序运行时创建和修改 Java 类，无需编译器的帮助。Byte Buddy 是一个相当新的库，但提供了 CGLIB 或 Javassist 提供的任何功能等等。Byte Buddy 可以完全定制到字节码级别，并带有一个富有表现力的领域特定语言（DSL），在操作字节码时，它可能是最安全、最合理的选择，而且代码可读性很高。
- [Javassist](http://jboss-javassist.github.io/javassist/)：Java 中用于编辑字节码的类库；在Java 程序能够在运行时定义一个新类，并在 JVM 加载类文件时修改它。与其他类似的字节码编辑器不同，Javassist 提供了两个级别的 API：源代码级和字节码级。如果用户使用源代码级 API，他们可以在不了解 Java 字节码规范的情况下编辑类文件。
- [CGlib](https://github.com/cglib/cglib)：CGLIB 速度非常快，这是它仍然存在的主要原因之一。一般来说，允许在运行时重写类的库必须避免在重写相应的类之前加载任何类型。 因此，它们不能使用 Java 反射 API 来加载反射中使用的任何类型，所以他们必须通过 IO（这是一个性能破坏者）读取类文件。 这使得 Javassist 或 Proxetta 比 Cglib 慢得多，CGLIB 只是通过反射 API 读取方法并覆盖它们。
- [ASM](http://asm.ow2.org/)： CGLIB、Byte Buddy 和几乎所有其他库都建立在 ASM 之上，ASM 本身在非常低的级别上操作字节码。这对大多数人来说是个障碍，因为您必须了解字节码和一点点 JVMS 才能正确使用它。利用ASM手写字节码时，需要利用一系列visitXXXXInsn()方法来写对应的助记符，所以需要先将每一行源代码转化为一个个的助记符，然后通过ASM的语法转换为visitXXXXInsn()这种写法。第一步将源码转化为助记符就已经够麻烦了，不熟悉字节码操作集合的话，需要我们将代码编译后再反编译，才能得到源代码对应的助记符。第二步利用ASM写字节码时，如何传参也很令人头疼。ASM社区也知道这两个问题，所以提供了工具ASM ByteCode Outline。


## 参考

- [1] [java8.Instrumentation](https://docs.oracle.com/javase/8/docs/api/java/lang/instrument/Instrumentation.html)

