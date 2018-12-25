---
description: shell 字符串
---

# string

## 计算字符串数量

`${#VALUE}` 计算变量VALUE字符串的字符数量

```text
unset VALUE
VALUE=zhangjie.31.beijing
echo ${#VALUE}
#19
```

## 根据分隔符 来 删除 左边 或者 右边的字符串

\*表示通配符，用于匹配字符串将被删除的字串。

注意 '\#' 和 '%' 都表示分隔界限 , 具有非贪婪属性

'\#\#' 和 '%%' 也表示界限，具有贪婪属性

'\#' 代表从坐到右 '%' 代表从右到坐

注意 '\#' 和 '%' 的语法规则

* '\#' 是 \#\*\(分隔符\) 可以这么理解 '\#' 是从左到右， 所以 是 '\#' + 所有\(\*\) + 分隔符 
* '%' 是 %\(分隔符\)\* , 可以这么理解 '%' 是从右 到左， 所以是 '%' + 分隔符 + 所有\(\*\) 

### 删除分隔符 最右边的，保留左边的

删除 以 分隔符 ‘.’ 匹配的第一个右边的字符串，保留左边的

`${VALUE%.*}`

```text
unset VALUE
VALUE=zhangjie.31.beijing

echo ${VALUE%.*}

#zhangjie.31
```

### 删除分隔符 右边的， 保留最最 左边的

删除 以 分隔符 ‘.’ 匹配的字符串，保留**最**左边的

`${VALUE%%.*}`

```text
unset VALUE
VALUE=zhangjie.31.beijing

echo ${VALUE%.*}
#zhangjie
```

### 删除分隔符最左边的， 保留右边的

`${VALUE#*.}` 除VALUE字符串中以分隔符“.”匹配的最左边字符，保留右边字符。

```text
unset VALUE
VALUE=zhangjie.31.beijing
echo ${VALUE#*.}
#31.beijing
```

### 删除分隔符左边的， 保留最右边的

`${VALUE##*.}` 除VALUE字符串中以分隔符“.”匹配的左边字符，保留**最最** 右边字符。

```text
unset VALUE
VALUE=zhangjie.31.beijing
echo ${VALUE##*.}
#beijing
```

## 替换

替换符号 '/' 和 '//'

'/' 代表仅仅替换一次, 属于非贪婪替换

'//' 代表全部替换, 属于贪婪替换

### 非贪婪替换

`${VALUE/OLD/NEW}`：用NEW子串替换VALUE字符串中第一个匹配的OLD子串。

```text
unset VALUE
VALUE=aabbcc.ccbbaa.ddeeff.ffeedd
echo ${VALUE/aa/gg}
#ggbbcc.ccbbaa.ddeeff.ffeedd
```

### 贪婪替换

`${VALUE//OLD/NEW}`：用NEW子串替换VALUE字符串中匹配的OLD子串。

```text
unset VALUE
VALUE=aabbcc.ccbbaa.ddeeff.ffeedd
echo ${VALUE//aa/gg}
#ggbbcc.ccbbgg.ddeeff.ffeedd
```

## 字符串截取

字符串截取 分为从左边数，开始解决多少长度的字符串

和 从右边数第几个位置 开始截取多少长度的字符串

截取的顺序都是往右边截取， 但是区别在于其实位置是从左边还是从右边

从左边数 左边第一个字符从“0”开始， 即第一个下标为0

从右边数 右边第一个字符从“0-1”开始, 即第一个下标为1。

表示偏移OFFSET个字符开始，LENGTH表示要截取字符的长度。如果没有LENGTH变量，表示偏移OFFSET个字符开始到字符串结束。

OFFSET 一直是往右的方向

左边第一个字符从“0”开始，

右边第一个字符从“0-1”开始。 表示偏移OFFSET个字符开始，LENGTH表示要截取字符的长度。如果没有LENGTH变量，表示偏移OFFSET个字符开始到字符串结束。

OFFSET 一直是往右的方向

### 起始位置从左边数，截取后面所有字符串

`${VALUE:OFFSET}` 从VALUE字符串的左边开始中截取子串。从0 开始

```text
unset VALUE
VALUE=aabbcc.ccbbaa.ddeeff.ffeedd
echo ${VALUE:7}
#ccbbaa.ddeeff.ffeedd
# 从左边开始下标是0， OFFSET 表示偏移OFFSET个字符开始， 包括下标为7 的元素
```

### 起始位置从左边数，截取后面固定长度的字符串

`${VALUE:OFFSET:2}` 从VALUE字符串的左边开始截取长度为2的子串。从0 开始

```text
unset VALUE
VALUE=aabbcc.ccbbaa.ddeeff.ffeedd
echo ${VALUE:7:2}
#cc
# 从左边开始下标是0， OFFSET 表示偏移OFFSET个字符开始， 包括下标为7 的元素, 然后截取长度为2的字符串
```

### 起始位置从右边数，截取后面所有字符串

`${VALUE:0-OFFSET}` 从VALUE字符串的右边开始数，第七个字符开始 ，截取右边所有字符串

```text
unset VALUE
VALUE=aabbcc.ccbbaa.ddeeff.ffeedd
echo ${VALUE:0-7}
#.ffeedd
# 注意这个包含 小数点
```

### 起始位置从右边数，截取后面固定长度的字符串

`${VALUE:0-OFFSET:LEN}` 从VALUE字符串的右边开始数，第七个字符开始 ，截取右边长度为LEN的字符串

```text
unset VALUE
VALUE=aabbcc.ccbbaa.ddeeff.ffeedd
echo ${VALUE:0-7:2}
#.f
# 注意这个包含 小数点
```

## 无值时 赋值并替换

`${VALUE:=WORD}` 当变量**未定义或者值为空**时，返回WORD的值的**同时并将WORD赋值给VALUE**，否则返回变量的值。

注意这里有 = ,所以VALUE 如果为空， 则VALUE 赋值为WORD的值, 返回也是WORD的值 如果 VALUE 不为空， 则返回VALUE值

```text
unset VALUE
echo ${VALUE:=zhangjie} 
#zhangjie
echo $VALUE 
#zhangjie
```

## 有值时 替换 但是不改变原值

`${VALUE:+WORD}` 当变量已赋值时，其值才用WORD替换，否则不进行任何替换。**但是不改变原来的值**

```text
unset VALUE
echo ${VALUE:+zhangjie} # 输出为空
echo $VALUE # 输出为空
VALUE=lisi
echo ${VALUE:+zhangjie} # 输出为 zhangjie
echo $VALUE # 输出为lisi 并没有改变
```

