## 数组

1. 数组是对象
2. 数组对象的length成员指明数组长度

```java
public class Arrays {
    public static void main(String[] args) {
        int[] a1 = { 1, 2, 3, 4, 5 };
        int[] a2;
        a2 = a1; // 句柄复制，指向相同对象
        for(int i = 0; i < a2.length; i++)
            a2[i]++;
        for(int i = 0; i < a1.length; i++)
            prt("a1[" + i + "] = " + a1[i]);  // 2，3，4，5，6
    }
    static void prt(String s) {
        System.out.println(s);
    }
}
```
## 二维数组初始化
```java
public class Arrays {
    public static void main(String[] args) {
        // 数组元素为基本数据类型
        int[] a1 = { 1, 2, 3, 4, 5 };
        int[] a2 = new int[10];

        // 数组元素为对象类型
        Integer[] a3 = {
                new Integer(1),
                new Integer(2),
                new Integer(3),
        };
        Integer[] a4 = new Integer[10];
    }
}
```

## 三维数组初始化

```java
import java.util.*;

public class MultiDimArray {
    static Random rand = new Random();
    static int pRand(int mod) {
        return Math.abs(rand.nextInt()) % mod + 1;
    }
    public static void main(String[] args) {
        int[][] a1 = {
                { 1, 2, 3, },
                { 4, 5, 6, },
        };
        for(int i = 0; i < a1.length; i++)
            for(int j = 0; j < a1[i].length; j++)
                prt("a1[" + i + "][" + j +
                        "] = " + a1[i][j]);
        // 3-D array with fixed length:
        int[][][] a2 = new int[2][2][4];
        for(int i = 0; i < a2.length; i++)
            for(int j = 0; j < a2[i].length; j++)
                for(int k = 0; k < a2[i][j].length;
                    k++)
                    prt("a2[" + i + "][" +
                            j + "][" + k +
                            "] = " + a2[i][j][k]);
        // 3-D array with varied-length vectors:
        int[][][] a3 = new int[pRand(7)][][];
        for(int i = 0; i < a3.length; i++) {
            a3[i] = new int[pRand(5)][];
            for(int j = 0; j < a3[i].length; j++)
                a3[i][j] = new int[pRand(5)];
        }
        for(int i = 0; i < a3.length; i++)
            for(int j = 0; j < a3[i].length; j++)
                for(int k = 0; k < a3[i][j].length;
                    k++)
                    prt("a3[" + i + "][" +
                            j + "][" + k +
                            "] = " + a3[i][j][k]);
        // Array of non-primitive objects:
        Integer[][] a4 = {
                { new Integer(1), new Integer(2)},
                { new Integer(3), new Integer(4)},
                { new Integer(5), new Integer(6)},
        };
        for(int i = 0; i < a4.length; i++)
            for(int j = 0; j < a4[i].length; j++)
                prt("a4[" + i + "][" + j +
                        "] = " + a4[i][j]);
        Integer[][] a5;
        a5 = new Integer[3][];
        for(int i = 0; i < a5.length; i++) {
            a5[i] = new Integer[3];
            for(int j = 0; j < a5[i].length; j++)
                a5[i][j] = new Integer(i*j);
        }
        for(int i = 0; i < a5.length; i++)
            for(int j = 0; j < a5[i].length; j++)
                prt("a5[" + i + "][" + j +
                        "] = " + a5[i][j]);
    }
    static void prt(String s) {
        System.out.println(s);
    }
}

```

注意：多维数组初始化时（包括二维），必须给出第一维度的长度。