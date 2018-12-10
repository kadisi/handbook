---
description: shell 字符串 
---

# 字符串处理技巧

${#VALUE} 计算变量VALUE字符串的字符数量

```
VALUE=abcd
echo ${#VALUE}

out: 4

``` 

* ${VALUE%.\*}或${VALUE%%.\*}：删除VALUE字符串中以分隔符“.”匹配的右边字符，保留左边字符。

```
VALUE=abcd.12e4
echo ${VALUE%%.*}

out: abcd
```

* ${VALUE#\*.}或${VALUE##\*.}：删除VALUE字符串中以分隔符“.”匹配的左边字符，保留右边字符。

```
VALUE=abcd.12e4
echo ${VALUE##*.}

out: 124
```

* ${VALUE/OLD/NEW}或${VALUE//OLD/NEW}：用NEW子串替换VALUE字符串中匹配的OLD子串。

```
VALUE=abcd.12e4
echo ${VALUE/cd/zzzz}

out: abzzzz.12e4

```

## 补充

	“*”表示通配符，用于匹配字符串将被删除的字串。

	“.”表示字符串中分隔符，可以为任意一个或多个字符。

	“%”表示从右向左匹配，

	“#”表示从左向右匹配，

	“\”表示替换，都属于非贪婪匹配，即匹配符合通配符的最短结果。

	与“%”、“#”和“/”类似的有“%%”、“##”和“//”，都属于贪婪匹配，即匹配符合通配符的最长结果。

* ${VALUE:OFFSET}或${VALUE:OFFSET:LENGTH}：从VALUE字符串的左边开始中截取子串。从0 开始


```
VALUE=abcd.12e4
echo ${VALUE:1:2}

out: bc

```

* ${VALUE:0-OFFSET}或${VALUE:0-OFFSET:LENGTH}：从VALUE字符串的右边开始中截取子串。

```

VALUE=abcd.12e4
echo ${VALUE:0-4:3} #12e

echo ${VALUE:0-4:2} #12

```

## 补充
	
	左边第一个字符从“0”开始，
	
	右边第一个字符从“0-1”开始。 表示偏移OFFSET个字符开始，LENGTH表示要截取字符的长度。如果没有LENGTH变量，表示偏移OFFSET个字符开始到字符串结束。

	OFFSET 一直是往右的方向

* ${VALUE:-WORD}：当变量未定义或者值为空时，返回值为WORD的内容，否则返回变量的值。

```
unset VALUE
echo ${VALUE:-zhangjie} # 输出 zhangjie
echo $VALUE # 输出 没有值

```


* ${VALUE:=WORD}：当变量未定义或者值为空时，返回WORD的值的同时并将WORD赋值给VALUE，否则返回变量的值。


```
unset VALUE
echo ${VALUE:=zhangjie} # 输出 zhangjie
echo $VALUE # 输出 zhangjie 

```

* ${VALUE:+WORD}：当变量已赋值时，其值才用WORD替换，否则不进行任何替换。但是不改变原来的值

```
unset VALUE
echo ${VALUE:+zhangjie} # 输出为空
echo $VALUE # 输出为空
VALUE=lisi
echo ${VALUE:+zhangjie} # 输出为 zhangjie
echo $VALUE # 输出为lisi 并没有改变

```

