# CPU分支预测对程序性能的影响

执行下面这段代码，你会发现排序的数组总是比未排序的数组计算快。

<!--more-->

```java
public class BranchPrediction
{
    public static int[] generateData(){
        // Generate data
        int arraySize = 32768;
        int[] data = new int[arraySize];

        Random rnd = new Random(0);
        for (int c = 0; c < arraySize; ++c)
            data[c] = rnd.nextInt() % 256;
        return data;
    }
    public static int[] generateSortedData(){
        // Generate data
        int[] data = generateData();
        Arrays.sort(data);
        return data;
    }
    public static void main(String[] args)
    {
        int[] data = generateData();
        int[] sortedData = generateSortedData();
        int sum1 = 0;
        long total1 =0L;
        long flag1 = System.nanoTime();
        for (int i = 0; i < 100000; ++i)
        {
            for (int datum : data) {
                if (datum >= 128)
                    sum1 += datum;
            }
        }
        long flag2 = System.nanoTime();
        total1 +=flag2-flag1;

        int sum2 = 0;
        long total2 =0L;
        long flag11 = System.nanoTime();
        for (int i = 0; i < 100000; ++i)
        {
            for (int sortedDatum : sortedData) {
                if (sortedDatum >= 128)
                    sum2 += sortedDatum;
            }
        }
        long flag22 = System.nanoTime();
        total2 +=flag22-flag11;
        System.out.println("exec unsorted array cost: "+ (total1 / 1000000000.0) +"s,result= " + sum1);
        System.out.println("exec sorted array cost: "+(total2 / 1000000000.0) +"s,result= " + sum2);
    }
    // exec unsorted array cost: 6.561110566s,result= 565377344
    // exec sorted array cost: 3.197814074s,result= 565377344
}
```

## 原因

出现这种现象的原因是CPU分支预测(可以参考[CPU流水线简介](/posts/2022/12/efefddd2/)一文中分支预测部分)。

注意下面的分支语句:

```java
if (data[c] >= 128)
```

数据在0到255之间均匀分布。当数据排序时，大致前半部分迭代不会进入if语句。之后，它们都会进入if语句。这对分支预测器非常友好，因为分支连续多次走向相同的方向。

```
T = branch taken
N = branch not taken

data[] = 0, 1, 2, 3, 4, ... 126, 127, 128, 129, 130, ... 250, 251, 252, ...
branch = N  N  N  N  N  ...   N    N    T    T    T  ...   T    T    T  ...

       = NNNNNNNNNNNN ... NNNNNNNTTTTTTTTT ... TTTTTTTTTT  (easy to predict)
```
然而，当数据完全随机时，分支预测器变得无用，因为它不能预测随机数据。因此可能会有大约50%的错误预测。
```
data[] = 226, 185, 125, 158, 198, 144, 217, 79, 202, 118,  14, 150, 177, 182, ...
branch =   T,   T,   N,   T,   T,   T,   T,  N,   T,   N,   N,   T,   T,   T  ...

       = TTNTTTTNTNNTTT ...   (completely random - impossible to predict)
```

## 更多的例子

```java
int Sum(int max) {
    int sum=0;
    for(int i=0; i<max; i++) {
        if (<condition>) {
            sum += max;
        }
    }
    return sum;
}
```

对于不同的条件，我们有下面的统计结果展示了分支预测对执行结果的影响。

|Condition|Pattern|Time (ms)|
|:---|:---|:---|
|(i & 0x80000000) == 0|TTT…|322|
|(i & 0xffffffff) == 0|FFF…|276|
|(i & 1) == 0	|T/F T/F … |760|
|(i & 3) == 0	|TFFFTFFF…	|513|
|(i & 2) == 0	|TTFFTTFF…	|1675|
|(i & 4) == 0	|TTTTFFFFTTTTFFFF…|1275|
|(i & 8) == 0	|8T 8F 8T 8F …|752|
|(i & 16) == 0	|16T 16F 16T 16F …|490|

## How

- 避免**关键循环**中依赖于数据的分支。
- **关键循环**尽量减少分支判断(过多的分支会影响预测命中率)。
- (如果可以)使用位运算来优化性能，例如:

```java
if (data[c] >= 128)
    sum += data[c];
// 替换为
int t = (data[c] - 128) >> 31;
sum += ~t & data[c];
```

## 参考

- [1] [Fast and slow if-statements: branch prediction in modern processors](http://igoro.com/archive/fast-and-slow-if-statements-branch-prediction-in-modern-processors/)

- [2] [Branch Prediction in Java](https://www.baeldung.com/java-branch-prediction)

- [3] [why is processing a sorted array faster than processing an unsorted array](https://stackoverflow.com/questions/11227809/why-is-processing-a-sorted-array-faster-than-processing-an-unsorted-array)

