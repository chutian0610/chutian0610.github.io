# Trino源码学习-自定义函数

Presto Functions 并不能像 Hive UDF 一样动态加载，需要根据 Function 的类型，实现 Presto 内部定义的不同接口，在 Presto 服务启动时进行注册，然后才能在 SQL 执行时进行调用。

<!--more-->

## 函数定义

Presto 内部将 Functions 分为以下三大类：

- Scalar Function，即标量函数。将传递给它的一个或者多个参数值，进行计算后，返回一个确定类型的标量值。
- Aggregation Function，即聚合函数。计算从分组的列中取得值，返回一个值。
- Window Function，即开窗函数。计算从分窗的列取得值，并返回多个值。

对于不同类型的函数，需要遵循不同的规则进行实现。

### 标量函数

Presto 使用注解框架来实现标量函数，标量函数分别需要定义函数名称、输入参数类型和返回结果类型。下面介绍几种开发标量函数常用的注解：

- `@ScalarFunction`：用于声明标量函数的名称和别名
- `@Description`：用于生成函数的功能描述
- `@SqlType`：用于声明函数的返回类型和参数类型
- `@TypeParameter`：用于声明类型变量，它所声明的类型变量可以用于函数的返回类型和参数类型，框架在运行时会自动将变量与具体的类型进行绑定
- `@SqlNullable`：用于表示函数参数或返回结果可能为`NULL`。如果方法的参数不使用此注解，当函数参数包含`NULL`时，则该函数不会被调用，框架自动返回结果`NULL`。当 Java 代码中用于实现函数的方法的返回值为包装类型时，必须要在实现方法上加上该注解，且该注解无法用于 Java 基础类型

下面用一个简单的`is_null`函数来具体说明如何使用以上注解进行标量函数开发。

```java
public class ExampleIsNullFunction
{
    @ScalarFunction(value = "is_null", alias = "isnull")
    @Description("Returns TRUE if the argument is NULL")
    @SqlType(StandardTypes.BOOLEAN)
    public static boolean isNull(@SqlNullable @SqlType(StandardTypes.VARCHAR) Slice string)
    {
        return (string == null);
    }
}
```

以上代码实现的`is_null`函数功能为：判断传入的`VARCHAR`类型参数是否为`NULL`，如果为`NULL`则返回`true`，否则返回`false`。其中：

- `@ScalarFunction(value = "is_null", alias = "isnull")`声明了函数名为`is_null`，函数别名为`isnull`，即在 SQL 中使用`is_null`和`isnull`都可以调用该函数
- `@Description("Returns TRUE if the argument is NULL")`声明了函数描述，使用`show functions`命令可以看到函数的描述
- `@SqlType(StandardTypes.BOOLEAN)`声明了函数的返回类型为`BOOLEAN`
-  因为当函数参数为`NULL`时，我们不能直接返回`NULL`，而是要进行判断，所以要加上`@SqlNullable`避免框架自动返回`NULL`
- `@SqlType(StandardTypes.VARCHAR)`声明了函数的参数类型为`VARCHAR`

注意到，这里使用了 Java 类型`Slice`来接收 SQL 中`VARCHAR`类型的值。框架会自动将 SQL 中的数据类型与"原生容器类型"（Native container type）进行绑定，目前"原生容器类型"只包括：`boolean`、`long`、`double`、`Slice`和`Block`。`VARCHAR`对应的原生容器类型是`Slice`而不是`String`,`Slice`的本质是对`byte[]`进行了封装，为的是更加高效、自由地对内存进行操作。`Block`可以简单的理解为对应 SQL 中的复合类型(数组/struct/map)。具体的对应关系和绑定过程涉及 Presto 的类型系统和函数调用过程，在此不作展开。

进一步地，我们想对 `is_null`函数进行升级，使它能够处理任意类型的参数，这时`@TypeParameter`注解就派上用场了，函数的实现可以改写为：

```java
@ScalarFunction(value = "is_null", alias = "isnull")
@Description("Returns TRUE if the argument is NULL")
public class ExampleIsNullFunction
{
    private IsNullFunctions()
    {
    }

    @TypeParameter("T")
    @SqlType(StandardTypes.BOOLEAN)
    public static boolean isNullSlice(@SqlNullable @SqlType("T") Slice value)
    {
        return (value == null);
    }

    @TypeParameter("T")
    @SqlType(StandardTypes.BOOLEAN)
    public static boolean isNullLong(@SqlNullable @SqlType("T") Long value)
    {
        return (value == null);
    }

    @TypeParameter("T")
    @SqlType(StandardTypes.BOOLEAN)
    public static boolean isNullDouble(@SqlNullable @SqlType("T") Double value)
    {
        return (value == null);
    }

    @TypeParameter("T")
    @SqlType(StandardTypes.BOOLEAN)
    public static boolean isNullBoolean(@SqlNullable @SqlType("T") Boolean value)
    {
        return (value == null);
    }

    @TypeParameter("T")
    @SqlType(StandardTypes.BOOLEAN)
    public static boolean isNullBlock(@SqlNullable @SqlType("T") Block value)
    {
        return (value == null);
    }
}
```

可以看到，`@TypeParameter`的使用有点类似 Java 中泛型的用法，类型变量`T`在声明完之后就可以在`@SqlType`注解中使用。在实际的调用过程中，框架会将`T`与实际 SQL 类型进行绑定，然后再去调用以对应的原生容器类型为参数的实际方法。

### 聚合函数

聚合的过程一般涉及多行，有一个累积计算的过程，又由于 Presto 是一个分布式的计算引擎，数据分布在多个节点，所以需要用状态对象来维护和记录中间计算结果。

引入状态之后，Presto 将聚合的过程抽象为三个步骤：

- `input(state, value)`: 首先，`input` 阶段分别在不同的 worker 中进行，将行值进行累积计算到`state`中；
- `combine(state1, state2)`: `combine`阶段将上一步得到的`state`进行两两结合；
- `output(state, out)`: 经过前两步，最终会得到一个`state`，在`output`阶段对最终的`state`进行处理输出。

在实现方面，聚合函数的开发使用了和标量函数类似的注解框架，但是由于状态概念的引入，需要定义一个继承于`AccumulatorState`接口的状态接口，对于简单的聚合，该接口只需要新增聚合所需的`getter`和`setter`，框架会自动生成相关的实现和序列化代码；如果聚合过程中需要记录复杂类型（`LIST`、`MAP`或自定义的类）的状态，则需要额外实现`AccumulatorStateFactory`接口和`AccumulatorStateSerializer`接口，并在状态接口上使用`@AccumulatorStateMetadata`注解，在注解中指定`stateFactoryClass`和`stateSerializerClass`。

下面以实现求`DOUBLE`类型的列均值的聚合函数`avg_double`为例来说明如何进行简单聚合函数的开发。`avg_double`的聚合状态只需要记录累积和与加数个数，所以状态接口的定义如下：

```java
public interface LongAndDoubleState
        extends AccumulatorState
{
    long getLong();

    void setLong(long value);

    double getDouble();

    void setDouble(double value);
}
```

使用定义好的状态接口进行聚合函数实现：

```java
@AggregationFunction("avg_double")
public class AverageAggregation
{
    @InputFunction
    public static void input(LongAndDoubleState state, @SqlType(StandardTypes.DOUBLE) double value)
    {
        state.setLong(state.getLong() + 1);
        state.setDouble(state.getDouble() + value);
    }

    @CombineFunction
    public static void combine(LongAndDoubleState state, LongAndDoubleState otherState)
    {
        state.setLong(state.getLong() + otherState.getLong());
        state.setDouble(state.getDouble() + otherState.getDouble());
    }

    @OutputFunction(StandardTypes.DOUBLE)
    public static void output(LongAndDoubleState state, BlockBuilder out)
    {
        long count = state.getLong();
        if (count == 0) {
            out.appendNull();
        }
        else {
            double value = state.getDouble();
            DOUBLE.writeDouble(out, value / count);
        }
    }
}
```

可以看到聚合函数的实现使用了以下注解：

-  `@AggregationFunction`声明了聚合函数的名称，也可以指定函数的别名
-  `@InputFunction`、`@CombineFunction`和`@OutputFunction`分别用来标记聚合的三个步骤，其中`@OutputFunction`注解需要声明聚合函数返回结果的数据类型
-  `BlockBuilder`类为结果输出类，聚合计算出的最终结果值将通过`BlockBuilder`进行输出

#### 聚合状态

聚合状态都会从内存中对象物化为 Block(Presto的底层存储)。

```java
public interface AccumulatorStateSerializer<T>
{
    Type getSerializedType();

    void serialize(T state, BlockBuilder out);

    void deserialize(Block block, int index, T state);
}
```

在实现AccumulatorState后，也可以按需实现GroupedAccumulatorState(表示 group by 后的聚合状态)。

### 窗口函数

窗口函数在查询结果的行上进行计算，执行顺序在`HAVING`子句之后，`ORDER BY`子句之前。在 Presto SQL 中，窗口函数的语法形式如下：

```sql
windowFunction(arg1,....argn) OVER([PARTITION BY<...>] [ORDER BY<...>] [RANGE|ROWS BETWEEN AND])
```

由此可见，窗口函数语法由关键字`OVER`触发，且包含三个子句：

1.  `PARTITION BY`: 指定输入行分区的规则，类似于聚合函数的`GROUP BY`子句，不同分区里的计算互不干扰(窗口函数的计算是并发进行的，并发数和`partition`数量一致)，缺省时将所有数据行视为一个分区
2.  `ORDER BY`: 决定了窗口函数处理输入行的顺序
3.  `RANGE|ROWS BETWEEN AND`: 指定窗口边界，不常用，缺省时的窗口为当前行所在的分区第一行到当前行

窗口函数的开发需要实现`WindowFunction`接口，`WindowFunction`接口中声明了两个方法：

-  `void reset(WindowIndex windowIndex)`: 处理新分区时，都会调用该方法进行初始化，`WindowIndex`包含了已排序的分区的所有行
-  `void processRow(BlockBuilder output, int peerGroupStart, int peerGroupEnd, int frameStart, int frameEnd)`: 窗口函数的实现方法，`BlockBuilder`为结果输出类，计算出来的值将通过`BlockBuilder`进行输出；`peerGroupStart`和`peerGroupEnd`为当前处理的行所在的分区的开始和结束的位置；`frameStart`和`frameEnd`为当前处理行所在的窗口的开始和结束位置。

实现一个返回窗口中第一个值的窗口函数`first_value(x)`的代码如下：

```java
@WindowFunctionSignature(name = "first_value", typeVariable = "T", returnType = "T", argumentTypes = "T")
public class FirstValueFunction
        extends WindowFunction
{
    private final int argumentChannel;
    private WindowIndex windowIndex;

    public FirstValueFunction(List<Integer> argumentChannels)
    {
        this.argumentChannel = getOnlyElement(argumentChannels);
    }

    @Override
    public void reset(WindowIndex windowIndex)
    {
        this.windowIndex = windowIndex;
    }

    @Override
    public void processRow(BlockBuilder output, int peerGroupStart, int peerGroupEnd, int frameStart, int frameEnd)
    {
        if (frameStart < 0) {
            output.appendNull();
            return;
        }

        //Outputs a value from the index
        windowIndex.appendTo(argumentChannel, frameStart, output);
    }
}
```

其中：

-  `@WindowFunctionSignature`注解声明了窗口函数的名称，为了处理任意数据类型的字段，这里还声明了类型变量`T`，并将返回类型和参数类型都指定为`T`
-  构造函数中的`argumentChannels`为参数字段所在列的索引值
-  `processRow`方法中，每次只需要通过列索引`argumentChannel`和当前行所在的窗口起始索引`frameStart`，就能确定窗口中的第一个值

## 函数注册

Presto 函数由`MetadataManager`中的`FunctionRegistry`进行管理，开发的函数要生效必须要先注册到`FunctionRegistry`中。函数注册是在 Presto 服务启动过程中进行的，有以下两种方式进行函数注册。

## 内置函数注册

内置函数指的是 Presto 自带的函数库中的函数，函数的实现位于`presto-main`模块中，在`FunctionRegistry`初始化时进行注册。具体的注册过程使用了建造者模式，不同类型的函数注册只需要调用`FunctionListBuilder`对象对应的方法进行注册，关键代码如下：

```java
FunctionListBuilder builder = new FunctionListBuilder()
                .window(RowNumberFunction.class)
                .aggregate(ApproximateCountDistinctAggregation.class)
                .scalar(RepeatFunction.class)
                .function(MAP_HASH_CODE)
                ......
```

## 插件函数注册

内置函数满足不了使用需求时，就需要自行开发函数来拓展函数库。开发者自行编写的拓展函数一般通过插件的方式进行注册。`PluginManager`在安装插件时会调用插件的`getFunctions()`方法，将获取到的函数集合通过`MetadataManager`的`addFunctions`方法进行注册：

```java
public void installPlugin(Plugin plugin)
    {
        ......
       for (Class<?> functionClass : plugin.getFunctions()) {
            log.info("Registering functions from %s", functionClass.getName());
            metadata.addFunctions(extractFunctions(functionClass));
        }
        ......
    }
```

所以用做拓展函数库的插件，需要实现`getFunctions()`方法，来返回拓展的函数集合，例：

```java
public class ExampleFunctionsPlugin
        implements Plugin
{
    @Override
    public Set<Class<?>> getFunctions()
    {
        return ImmutableSet.<Class<?>>builder()
                .add(ExampleNullFunction.class)
                .add(IsNullFunction.class)
                .add(IsEqualOrNullFunction.class)
                .add(ExampleStringFunction.class)
                .add(ExampleAverageFunction.class)
                .build();
    }
}
```

