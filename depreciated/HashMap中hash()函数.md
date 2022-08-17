---
title: 关于HashMap中hash()函数的思考
date: 2019-04-09
categories: Java集合
tags: [Java集合]
---
<meta name="referrer" content="no-referrer" />

## 关于HashMap中hash()函数的思考

### JDK7中hash函数的实现

~~~java
static int hash(int h) {
        h ^= (h >>> 20) ^ (h >>> 12);
        return h ^ (h >>> 7) ^ (h >>> 4);
}
~~~

这段代码是为了对key的hashCode进行扰动计算，防止不同hashCode的高位不同但低位相同导致的hash冲突。简单点说，就是为了把高位的特征和低位的特征组合起来，降低哈希冲突的概率，也就是说，尽量做到任何一位的变化都能对最终得到的结果产生影响。

举个例子来说，我们现在想向一个HashMap中put一个K-V对，Key的值为“hollischuang”，经过简单的获取hashcode后，得到的值为“1011000110101110011111010011011”，如果当前HashTable的大小为16，即在不进行扰动计算的情况下，他最终得到的index结果值为11。由于15的二进制扩展到32位为“00000000000000000000000000001111”，所以，一个数字在和他进行按位与操作的时候，前28位无论是什么，计算结果都一样（因为0和任何数做与，结果都为0）。如下图所示。

![](https://img2018.cnblogs.com/blog/765260/201905/765260-20190508143618422-316713866.jpg)

可以看到，后面的两个hashcode经过位运算之后得到的值也是11 ，虽然我们不知道哪个key的hashcode是上面例子中的那两个，但是肯定存在这样的key，这就产生了冲突。

那么，接下来，我看看一下经过扰动的算法最终的计算结果会如何。

![](https://img2018.cnblogs.com/blog/765260/201905/765260-20190508143648812-811205659.jpg)

从上面图中可以看到，之前会产生冲突的两个hashcode，经过扰动计算之后，最终得到的index的值不一样了，这就很好的避免了冲突。

~~~java
static int indexFor(int h, int length) {
        return h & (length-1);
}
~~~



我们一般对哈希表的散列很自然地会想到用hash值对length取模（即除法散列法），Hashtable中也是这样实现的，这种方法基本能保证元素在哈希表中散列的比较均匀，但取模会用到除法运算，效率很低，HashMap中则通过h&(length-1)的方法来代替取模，同样实现了均匀的散列，但效率要高很多，这也是HashMap对Hashtable的一个改进。

首先，length为2的整数次幂的话，h&(length-1)就相当于对length取模，这样便保证了散列的均匀，同时也提升了效率；其次，length为2的整数次幂的话，为偶数，这样length-1为奇数，奇数的最后一位是1，这样便保证了h&(length-1)的最后一位可能为0，也可能为1（这取决于h的值），即与后的结果可能为偶数，也可能为奇数，这样便可以保证散列的均匀性，而如果length为奇数的话，很明显length-1为偶数，它的最后一位是0，这样h&(length-1)的最后一位肯定为0，即只能为偶数，这样任何hash值都只会被散列到数组的偶数下标位置上，这便浪费了近一半的空间，因此，length取2的整数次幂，是为了使不同hash值发生碰撞的概率较小，这样就能使元素在哈希表中均匀地散列。

### JDK8中hash函数的实现

~~~java
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
~~~

java8将hash函数简化了，但目的和java7是一样的，这种做法可以称为“扰动函数”，目的就是为了是散列值能够解区并分散均匀（两个目的）。

理论上key.hashCode()方法得到的散列值为int是个32为带符号整数，范围从**-2147483648**到**2147483648**。前后加起来大概40亿的映射空间。只要哈希函数映射得比较均匀松散，一般应用是很难出现碰撞的。但问题是一个40亿长度的数组，内存是放不下的。你想，HashMap扩容之前的数组初始大小才16。所以这个散列值是不能直接拿来用的。用之前还要先做对数组的长度取模运算，得到的余数才能用来访问数组下标。

jdk8中没有定义单独的indexFor()函数而是在取模时直接用(n-1)&hash，原理和JDK7是一样的。以初始长度16为例，16-1=15。2进制表示是**00000000 00000000 00001111**。和某散列值做“与”操作如下，结果就是截取了最低的四位值。

但这时候问题就来了，这样就算我的散列值分布再松散，要是只取最后几位的话，碰撞也会很严重。更要命的是如果散列本身做得不好，分布上成等差数列的漏洞，恰好使最后几个低位呈现规律性重复。



这时候“**扰动函数**”的价值就体现出来了，说到这里大家应该猜出来了。看下面这个图:

![](https://img2018.cnblogs.com/blog/765260/201905/765260-20190508143706124-173858503.jpg)

右位移16位，正好是32bit的一半，自己的高半区和低半区做异或，就是为了**混合原始哈希码的高位和低位，以此来加大低位的随机性**。而且混合后的低位掺杂了高位的部分特征，这样高位的信息也被变相保留下来。但明显Java 8觉得扰动做一次就够了，做4次的话，多了可能边际效用也不大，所谓为了效率考虑就改成一次了。

------
参考文献：
知乎问题：[JDK 源码中 HashMap 的 hash 方法原理是什么？](https://www.zhihu.com/question/20733617)