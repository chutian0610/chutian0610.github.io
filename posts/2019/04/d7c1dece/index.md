# 生命游戏

生命游戏是英国数学家约翰·何顿·康威在1970年发明的细胞自动机。

<!--more-->

生命游戏其实是一个零玩家游戏，它包括一个二维矩形世界，这个世界中的每个方格居住着一个活着的或死了的细胞。一个细胞在下一个时刻生死取决于相邻八个方格中活着的或死了的细胞的数量。如果相邻方格活着的细胞数量过多，这个细胞会因为资源匮乏而在下一个时刻死去；相反，如果周围活细胞过少，这个细胞会因太孤单而死去。

实际中，你可以设定周围活细胞的数目怎样时才适宜该细胞的生存。如果这个数目设定过高，世界中的大部分细胞会因为找不到太多的活的邻居而死去，直到整个世界都没有生命；如果这个数目设定过低，世界中又会被生命充满而没有什么变化。实际中，这个数目一般选取2或者3；这样整个生命世界才不至于太过荒凉或拥挤，而是一种动态的平衡。这样的话，游戏的规则就是：当一个方格周围有2或3个活细胞时，方格中的活细胞在下一个时刻继续存活；即使这个时刻方格中没有活细胞，在下一个时刻也会“诞生”活细胞。

在这个游戏中，还可以设定一些更加复杂的规则，例如当前方格的状况不仅由父一代决定，而且还考虑祖父一代的情况。你还可以作为这个世界的上帝，随意设定某个方格细胞的死活，以观察对世界的影响。

在游戏的进行中，杂乱无序的细胞会逐渐演化出各种精致、有形的结构；这些结构往往有很好的对称性，而且每一代都在变化形状。一些形状已经锁定，不会逐代变化。有时，一些已经成形的结构会因为一些无序细胞的“入侵”而被破坏。但是形状和秩序经常能从杂乱中产生出来。

细胞自动机（又称元胞自动机），名字虽然很深奥，但是它的行为却是非常美妙的。所有这些怎样实现的呢？我们可以把计算机中的宇宙想象成是一堆方格子构成的封闭空间，尺寸为N的空间就有`N*N`个格子。而每一个格子都可以看成是一个生命体，每个生命都有生和死两种状态，如果该格子生就显示蓝色，死则显示白色。每一个格子旁边都有邻居格子存在，如果我们把3*3的9个格子构成的正方形看成一个基本单位的话，那么这个正方形中心的格子的邻居就是它旁边的8个格子。

每个格子的生死遵循下面的原则：

* 当前细胞为死亡状态时，当周围有3个存活细胞时，该细胞变成存活状态。 （模拟繁殖）
* 当前细胞为存活状态时，当周围低于2个（不包含2个）存活细胞时， 该细胞变成死亡状态。（模拟人口稀少）
* 当前细胞为存活状态时，当周围有2个或3个存活细胞时， 该细胞保持原样。
* 当前细胞为存活状态时，当周围有3个以上的存活细胞时，该细胞变成死亡状态。（模拟过度拥挤）


设定图像中每个像素的初始状态后依据上述的游戏规则演绎生命的变化，由于初始状态和迭代次数不同，将会得到令人叹服的优美图案。这样就把这些若干个格子（生命体）构成了一个复杂的动态世界。运用简单的3条作用规则构成的群体会涌现出很多意想不到的复杂性为，这就是复杂性科学的研究焦点。

细胞自动机有一个通用的形式化的模型，每个格子（或细胞）的状态可以在一个有限的状态集合S中取值，格子的邻居范围是一个半径r，也就是以这个格子为中心，在距离它r远的所有格子构成了这个格子的邻居集合，还要有一套演化规则，可以看成是一个与该格子当前状态以及邻居状态相关的一个函数，可以写成`f:S*S^((2r)^N-1)->S`。这就是细胞自动机的一般数学模型。

下面是一个简单的元胞自动机代码片段: 

```java
package info.victor;

import java.util.Random;

public class CeilsMatrix {
    public int getRows() {
        return rows;
    }

    public int getCols() {
        return cols;
    }

    private final int rows;
    private final int cols;

    /**
     * 返回当前细胞状态快照
     * @return
     */
    public boolean[][] getCurrentMatrix() {
        boolean[][] snapshot = new boolean[rows][cols];
        for(int i = 0;i < currentMatrix.length;i++){
            for(int j = 0;j < currentMatrix[i].length;j++){
                snapshot[i][j] = currentMatrix[i][j];
            }
        }
        return snapshot;
    }

    private boolean[][] currentMatrix;

    /**
     * 细胞容器初始化方法
     * @param rows
     * @param cols
     */
    public CeilsMatrix(int rows, int cols) {
        this.rows = rows;
        this.cols = cols;
        currentMatrix = new boolean[rows][cols];
    }

    /**
     * 此函数初始化容器，同时按照density[0,1.0] 的比例,设置细胞的生命状态
     * @param rows
     * @param cols
     * @param density
     */
    public CeilsMatrix(int rows, int cols,double density){
        this.rows = rows;
        this.cols = cols;
        currentMatrix = new boolean[rows][cols];
        // 随机
        Random RANDOM = new Random();
        for (int i = 0; i < rows; ++i) {
            for (int j = 0; j < cols; ++j) {
                currentMatrix[i][j] = (RANDOM.nextDouble() >= density);
            }
        }
    }

    /**
     * 更新所有细胞状态
     */
    public void updateCeilsStatues(){
        for (int i=0; i< rows;i++){
            for (int j=0;j<cols;j++){
                updateCeilStatus(i,j);
            }
        }
    }

    /**
     * 更新某个细胞的状态
     * @param i 行
     * @param j 列
     */
    private void updateCeilStatus(int i,int j){
        boolean[] neighbourStatus = getNeighbours(currentMatrix,i,j,rows,cols);
        int aliveCount = 0;
        for (boolean alive : neighbourStatus) {
            if (alive) {
                ++aliveCount;
            }
        }
        if (!currentMatrix[i][j]) {
            // current is dead
            if (aliveCount == 3) {
                currentMatrix[i][j] = true;
            } else {
                // do nothing
            }
        } else {
            // current is alive
            if (aliveCount < 2) {
                // 人口稀少
                currentMatrix[i][j] = false;
            } else if (aliveCount == 2 || aliveCount == 3) {
                // do nothing
            } else {
                // 过度拥挤
                currentMatrix[i][j] = false;
            }
        }
    }

    /**
     * 此处求索引映射值,通过取余处理了边界溢出
     * @param n 索引
     * @param border 边界
     * @return
     */
    private static int normalize(int n, int border) {
        int normalized = n % border;
        return (normalized >= 0) ? normalized : border + normalized;
    }

    /**
     * 获取某个细胞的邻居状态
     * @param ceilMap 细胞矩阵
     * @param row 细胞所在横座标
     * @param col 细胞所在纵座标
     * @param ROW 矩阵边界[行]
     * @param COL 矩阵边界[列]
     * @return
     */
    private static boolean[] getNeighbours(boolean[][] ceilMap, int row,
                                   int col,int ROW,int COL) {
        return new boolean[] {
                ceilMap[normalize(row - 1, ROW)][normalize(col - 1, COL)],
                ceilMap[normalize(row - 1, ROW)][normalize(col, COL)],
                ceilMap[normalize(row - 1, ROW)][normalize(col + 1, COL)],
                ceilMap[normalize(row, ROW)][normalize(col - 1, COL)],
                ceilMap[normalize(row, ROW)][normalize(col + 1, COL)],
                ceilMap[normalize(row + 1, ROW)][normalize(col - 1, COL)],
                ceilMap[normalize(row + 1, ROW)][normalize(col, COL)],
                ceilMap[normalize(row + 1, ROW)][normalize(col + 1, COL)]
        };
    }
}
```

>  [代码地址](https://github.com/chutian0610/code-lab/tree/main/projects/gameoflife)

