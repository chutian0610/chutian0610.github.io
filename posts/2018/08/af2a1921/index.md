# 算法之bitMap

**bitmap适用于什么场景？**

bitmap的核心思想是将一个元素的可能取值映射为某个空间内的值(常见的是1bit空间，提供0，1两种状态。或是2bit空间，提供 0，1，2，3四种状态)。通过某种顺序的映射表来节省存储空间。

<!--more-->

假设我们要对0-7内的5个元素(4,7,2,5,3)排列（这里假设这些元素没有重复）。那么我们就可以采用BitMap的方法来达到排列的目的。要表示8个数，我们就只需要8个bit（1Bytes）。

* 首先我们开辟1字节（8bit）的空间，将这些空间的所有bit位都置为0。
* 然后遍历这5个元素，首先第1个元素是4，那么就把4对应的位置为1,因为是从零开始的，所以要把第5个位置为1。
* 然后再处理第2个元素7，将第8个位置为1,，接着再处理第3个元素，一直到处理完所有元素，将相应的位置为1，这时候的内存的bit位的状态如下：`001111001`

对于bitmap算法，关键是如何确定目标映射到二进制bit位的map方式。

* 1byte有8个bit位，可以表示8个数，那么可以通过 `Index(N) = N/8 = N >> 3;`快速计算出所在的byte在byte数组的索引位。
* `Position(N) = N%8 = N & 0x07;`可以定位所在byte中的位置。

> 为什么bit & 7会相当于这个效果呢？因为这里一个byte是8位，而8对应的二进制表示形式为1000，那么比它小1的7的二进制形式为0111。在将bit和7进行与运算的时候，所有大于第3位的高位都被置为0，之保留最低的3位。这样，最低的3位数字最小是0，最大是7.就相当于对数字8求模的运算效果。

```java
public class BitMap {
    private byte[] bits;
    //容量
    private int capacity;
    public BitMap(int capacity){
        this.capacity = capacity;
        //1bit能存储8个数据，那么capacity数据需要多少个bit呢，capacity/8+1,右移3位相当于除以8
        bits = new byte[(capacity >>3 )+1];
    }
    public void add(int num){
        // num/8得到byte[]的index
        int arrayIndex = num >> 3;
        // num%8得到在byte[index]的位置
        int position = num & 0x07;
        //将1左移position后，那个位置自然就是1，然后和以前的数据做|，这样，那个位置就替换成1了。
        bits[arrayIndex] |= 1 << position; 
    }
    public boolean contain(int num){
        // num/8得到byte[]的index
        int arrayIndex = num >> 3;
        // num%8得到在byte[index]的位置
        int position = num & 0x07;
        //将1左移position后，那个位置自然就是1，然后和以前的数据做&，判断是否为0即可
        return (bits[arrayIndex] & (1 << position)) !=0;
    }
    public void clear(int num){
        // num/8得到byte[]的index
        int arrayIndex = num >> 3;
        // num%8得到在byte[index]的位置
        int position = num & 0x07;
        //将1左移position后，那个位置自然就是1，然后对取反，再与当前值做&，即可清除当前的位置了.
        bits[arrayIndex] &= ~(1 << position);
    } 
}
```

**这里`bits[i] & 0xFF`将一个byte和 0xFF进行了与运算。将byte强转为int不行吗?**

答案是不行的.其原因在于:

1. byte的大小为8bits而int的大小为32bits
2. java的二进制采用的是补码形式,byte是一个字节保存的,有8个位,即8个0、1。8位的第一个位是符号位,也就是说0000 0001代表的是数字1,1000 0000代表的就是-1,所以正数最大位0111 1111,也就是数字127,负数最大为1111 1111,也就是数字-128。上面说的是二进制原码,但是在java中采用的是补码的形式。

* 一个数为正,则它的原码、反码、补码相同
* 一个数为负,符号位为1,其余各位是对对应正数原码取反,然后整个数加1

当一个byte会转换成int时,由于int是32位,而byte只有8位这时会进行补位,例如补码11111111的十进制数为-1转换为int时变为11111111 11111111 11111111 11111111即0xffffffff但是这个数是不对的,这种补位就会造成误差。和0xff相与后,高24比特就会被清0了,结果就对了。Java中的一个byte,其范围是-128~127的,而Integer.toHexString的参数本来是int,如果不进行&0xff,那么当一个byte会转换成int时,对于负数,会做位扩展,举例来说,一个byte的-1(即0xff),会被转换成int的-1(即0xffffffff),那么转化出的结果就不是我们想要的了。而0xff默认是整形,所以,一个byte跟0xff相与会先将那个byte转化成整形运算,这样,结果中的高的24个比特就总会被清0,于是结果总是我们想要的。

> 出现这种情况的原因是Java中的数是有符号的，而bitmap把符号位也使用了。

