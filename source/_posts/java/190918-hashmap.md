---
title: JDK1.8 HashMap tableSizeFor()、hash() 方法梳理
date: 2019-9-18 19:00:00
description: "HashMap内部使用了很多巧妙的算法，让人不禁感叹开发者的强大，本文总结tableSizeFor()和hash()的原理，以便更好地理解HashMap及将算法思想应用到工作中。"
categories:
- HashMap
---

### tableSizeFor() 理解

<br>

以下代码片段来自JDK1.8 HashMap:

```
    /**
     * Returns a power of two size for the given target capacity.
     */
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```

首次看到这段代码，完全不知道在做什么，看注释大概意思是“找出大于所给容量最近的2的指数倍”。
例如：

    13 —> 16
    16 -> 16
    17 -> 32

为了理解代码此段代码含义，我编写了如下测试代码:

```
public class Main {
    static final int MAXIMUM_CAPACITY = 1 << 30;
    static int tableSizeFor(int cap) {
        int n = cap - 1;
        System.out.println("\nn = cap - 1");
        System.out.println(fillToLength(Integer.toBinaryString(n)));
        System.out.println("\nn |= n >>> 1");
        print(n, n >>> 1);
        n |= n >>> 1;
        System.out.println(fillToLength(Integer.toBinaryString(n)));
        System.out.println("\nn |= n >>> 2");
        print(n, n >>> 2);
        n |= n >>> 2;
        System.out.println(fillToLength(Integer.toBinaryString(n)));
        System.out.println("\nn |= n >>> 4");
        print(n, n >>> 4);
        n |= n >>> 4;
        System.out.println(fillToLength(Integer.toBinaryString(n)));
        System.out.println("\nn |= n >>> 8");
        print(n, n >>> 8);
        n |= n >>> 8;
        System.out.println(fillToLength(Integer.toBinaryString(n)));
        System.out.println("\nn |= n >>> 16");
        print(n, n >>> 16);
        n |= n >>> 16;
        System.out.println(fillToLength(Integer.toBinaryString(n)));
        System.out.println("\nn + 1");
        System.out.println(fillToLength(Integer.toBinaryString(n + 1)));
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }

    private static void print(int a, int b) {
        System.out.println(fillToLength(Integer.toBinaryString(a)) + " |\n"
                + fillToLength(Integer.toBinaryString(b)) + " =");
    }

    private static String fillToLength(String source) {
        if (source.length() < 32) {
            StringBuilder builder = new StringBuilder(source);
            while (builder.length() < 32) {
                builder.insert(0, '0');
            }
            return builder.toString();
        }
        return source;
    }

    public static void main(String[] args) {
        String source = "1000000000000110000000011000000";
        int cap = Integer.parseInt(source, 2);
        System.out.println("cap: " + cap);
        System.out.println("cap bin: \n" + fillToLength(source));
        int size = tableSizeFor(cap);
        System.out.println("result: " + size);
    }
}
```

打印`tableSizeFor()`执行的每一步，为了便于理解算法思想，中间结果均二进制输出，结果如下：

```
cap: 1073938624
cap bin: 
01000000000000110000000011000000

n = cap - 1
01000000000000110000000010111111

n |= n >>> 1
01000000000000110000000010111111 |
00100000000000011000000001011111 =
01100000000000111000000011111111

n |= n >>> 2
01100000000000111000000011111111 |
00011000000000001110000000111111 =
01111000000000111110000011111111

n |= n >>> 4
01111000000000111110000011111111 |
00000111100000000011111000001111 =
01111111100000111111111011111111

n |= n >>> 8
01111111100000111111111011111111 |
00000000011111111000001111111110 =
01111111111111111111111111111111

n |= n >>> 16
01111111111111111111111111111111 |
00000000000000000111111111111111 =
01111111111111111111111111111111

n + 1
10000000000000000000000000000000
result: 1073741824
```

由输出结果可以看到：
`n |= n >>> 1` 目的是将二进制位为`1`的后面一位也置为`1`，得到连续的 2 个`1`
在 2 个连续`1`的基础上，
`n |= n >>> 2` 得到连续 4 个`1`，
在 4 个连续`1`的基础上，
`n |= n >>> 4` 得到连续 8 个`1`，
在 8 个连续`1`的基础上，
`n |= n >>> 8` 得到连续 16 个`1`，
在 16 个连续`1`的基础上，
`n |= n >>> 16` 得到连续 32 个`1`。
到此如果二进制中最高位`1`，可复制到后面所有低位，及将最高位`1`后面的所有低位设置为`1`，
加 1 后，即为大于所给容量最近的2的指数倍，无论初始化容量多少，此算法拥有同样的时间复杂度o(1)

细心的朋友看到，在进行位运算前，初始数值先 减1，目的是为了避免 2的指数倍的数值 计算容量错误，
例如：没有减1的情况下计算 16 的对应容量初始值

    010000
    n |= n >>> 1
    011000
    n |= n >>> 2
    011110
    n |= n >>> 4
    011111
    ...
    n |= n >>> 16
    011111
    n + 1
    100000 = 32

这样计算出来的容量是所需容量的2倍，因此位运算前减 1 是必要的。

日常开发中我们要有使用位运算解决问题的想法，以下举几个小例子。

<br>

#### 计算 int 数值二进制中1的个数

<br>

```
    int numberOf1(int number) {
        int result = 0;
        while (number != 0) {
            result++;
            number &= (number - 1);
        }
        return result;
    }
```

测试结果：

    15 -> 4
    16 -> 1
    -1 -> 32

<br>

#### 判断两个数是否同为正数或者同为负数

<br>

```
    boolean isSameSymbolNumber(int a, int b) {
        return ((a >>> 31) ^ (b >>> 31)) == 0;
    }
```

<br>

### hash() 理解

<br>

以下代码片段来自JDK1.8 HashMap:

```
    /**
     * Computes key.hashCode() and spreads (XORs) higher bits of hash
     * to lower.  Because the table uses power-of-two masking, sets of
     * hashes that vary only in bits above the current mask will
     * always collide. (Among known examples are sets of Float keys
     * holding consecutive whole numbers in small tables.)  So we
     * apply a transform that spreads the impact of higher bits
     * downward. There is a tradeoff between speed, utility, and
     * quality of bit-spreading. Because many common sets of hashes
     * are already reasonably distributed (so don't benefit from
     * spreading), and because we use trees to handle large sets of
     * collisions in bins, we just XOR some shifted bits in the
     * cheapest possible way to reduce systematic lossage, as well as
     * to incorporate impact of the highest bits that would otherwise
     * never be used in index calculations because of table bounds.
     */
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

大致意思是：向下传播高位比特的影响。许多常见的哈希算法已经合理分布，我们只是以最简单的方式包含hash值最高位的影响，否则这些位将永远不会用于索引计算。

仅通过hashCode就能知道映射到 `table`数组 的哪个位置，计算方式如下：

```
(n - 1) & hash(key)
```



<br><br><br><br>

