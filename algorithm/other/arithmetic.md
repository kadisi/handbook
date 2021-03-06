---
计算机知识: 算法 算术运算
---

# 算术运算

## 算术运算

## 加法

见 leetcode 371

加法计算法则 跟 异或 和 与 运算有关

计算机的加法和乘法都是模拟人做加法和乘法的方法来设计和实现cpu算术运算模块的， 下面的例子

```text
    1 1 0 1
        1 1
+
----------------
  1 0 0 0 0
```

这个计算可以分为两个部分， 一部分是按位相加的，另一部分是进位的。 按位相加 其实对于二进制来说是异或运算了，相同为0 相异为1, 如 1 + 1 = 0 ， 0 + 1 = 1， 1 + 0 = 1， 0 + 0 = 0 那么什么情况下进位， 1 + 1 的时候才进位， 我们应该马上会想到是与运算， 进的哪一位加到高位继续参加下一步的运算。 那么我们会得到下面的公式：

`a + b = (a&b) << 1 + a ^ b`

但是a + b 的实现里还是用到了加法， 所以这是一个递归，那么什么是递归截止的终止条件呢， 就是进位的数为0的情况， 也就是\(a&b\)&lt;&lt;1 为0 时候 于是我们得到了如下代码：

```text
func getSum(a int, b int) int {
    if a == 0 {
        return b
    }
    return getSum((a&b)<<1, a^b)
}
```

## 减法

减法计算法则跟补码有关

加法是进位， 减法按照正常人的思维 则是需要借位,是这样吗？ 按照我们小时候对加减法的学习经验是这样的，但是计算机并不是这么处理的。 我们需要了解另一个概念： **补码**

**补码**

我们知道， 计算机对于有符号数， 用最高位作为符号位，`0` 代表 `+`, `1`代表`-`, 其余数位用做数值位 ，代表数值, 怎么推导一个数字的补码呢？

补码规定：正数和0的补码就是其原码，负数的补码是其正数的原码取反 再 加1

举个例子： 求-10 的补码:

十进制 10 的原码 （按照8位距离）为 0000 1010, 其反码为 1111 0101，再加1 为 1111 0110,

因此-10 的补码 为 1111 0110

那么我们再回到减法计算上来：

a = b - c 实际上等同于 a = b + \(-c\)

那么我们计算下面的式子：

**12 - 5**

= 0000 1100 + （5 的二进制 0000 0101 的取反 1111 1010 再加1 为 1111 1011）

= 0000 1100 + 1111 1011

=\(1\)0000 0111 = 7

**7 - 9**

= 0000 0111 + \(9 的二进制 0000 1001 取反 1111 0110 再加1为 1111 0111\)

= 0000 0111 + 1111 0111 = 1111 1110

= \(求正数 1111 1110 减一 为 1111 1101 取反 0000 0010 为 2\)

= -2

## 乘法

乘法的原理其实是跟我们做乘法的思想有点类似， 只不过计算机做乘法是通过位移和加法运算的。 举例：

**5 \* 3**

= 0000 0101 \* 0000 0011

5 乘以 3 ，代表了三个五 相加， 根据3 的二进制表示 0011 可以理解为 1 个五 + 2个五

按照这种思想 可以这么理解 3 的第 0 位 为 1， 因此需要将5左移0 位， 还是5

3 的第1位 为 1，因此需要将5 左移1位， 得 0000 1010 为10 两者相加 5 + 10 = 15

同样 对于

**3 \* 5**

= 0000 0011 \* 0000 0101

5的第0位为1 ,需要将3左移0位， 为 3

5的第1位为0， 不做操作

5的第2位为1， 需要将3左移2位， 为 0000 1100 为 12

3 + 12 为 15

golang 实现两个数相乘， 不用 加减乘除

```text
// i是从右边开始算， 范围是[0,31]闭区间
var MASK int32 = 0x01

func getBitValue(value int32, i uint) int32 {
    return (value >> i) & MASK
}

func Muilty(v1, v2 int32) int32 {
    var value int32 = 0
    var i uint = 0
    for ; i<=31; i= uint(Sum(int32(i), 1)) {
        if getBitValue(v2, i) == 1 {
            value = Sum(value, (v1 << i))
        }
        if v2 >> i == 0 {
            break
        }
    }
    return value
}

func Sum(a, b int32) int32{
    if a == 0 {
        return b
    }
    return Sum((a&b)<<1, a^b)
}
```

## 除法

除法是通过被除数不断的减去除数 直到结果为0 或者小于除数来实现的， 减去除数的次数称作为 `商`, 最后一次减法的差 成为 `余数` 也就是说

被除数 / 除数 = 商 . 余数 / 除数

在讨论二进制钱 我们首先看人们在纸上笔算完成十进制除法的过程， 下面的算式 描述了 575 / 25 的过程

```text
          商
        ________           ________
    除数| 被除数        25 | 575
```

第一步是比较除数和被除数的最高两位， 看被除数的最高两位中有几个除数， 这个例子的答案为2 ， （即 2 \* 25 = 50）并用57 减去 2 \* 25 在商的最高位上写下2， 得到下面的算式

```text
        2
      ______
   25| 575
       50
     --------
        75
```

将被除数的下一个5 移到 7 的后面， 并比较除数和75，由于75正好是25的整数倍，因此应在商的下一位写下数字3，并得到

```text
        23
      ______
   25| 575
       50
     --------
        75
        75
    --------
         0
```

因为已经处理到被除数的最后一位， 且75正好是除数的整数倍，因此除法结束，商为23， 余数位0

相比于二进制， 我们举例来说明

**209/5 = 1101 0001 / 0000 0101**

```text
从209的最高位开始
先取得209的最高位 1 （101 0001）最高位为1， 小于 5的二进制 （101） , 小于 101 因此没法整除，置为 0
第二位  11（01 0001） 小于 二进制 101 ， 同样置为0
第三位 110（1 0001）大于101， 得到的商为1， 余数 1 (1 0001)
1 (1 0001) 左移1位， 11 （ 0001）, 小于101 ,置为0
继续左移一位 110 （001） 大于101 得到的商为1， 余数为1 (001)
1 (001) 左移1位， 得 10 （01）小于101， 置为0
继续左移1位  100（1）, 小于101， 置为 0
继续左移 1001 大于101， 商为1， 余数为 100

于是乎结果为 00101001 余数位 100
```

通过上面我们发现，计算机计算加减乘除，都是转换为加法和位移运算完成的

leetocode 29

