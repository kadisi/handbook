---
description: shell eval 
---

# eval 基本用法

基本表达式 `eval command-line` 

其中command-line 是在终端商键入一条普通命令， 然而当在它前面放上eval时候，其结果是shell 在执行命令之前扫描它两次

如

```
pipe="|"


eval ls $pipe wc -l

```

shell 扫描第一次时候， 把pipe 替换出他的值 | , 接着， eval 使它在此扫描命令行， 这时候shell 会把 | 作为管道符了

最终的结果为变成

```
ls | wc -l
```

# eval echo \$$# 去的最后一个参数

例如 last.sh 内容
```
eval echo \$$#

```

执行last.sh

```
./last.sh one tow three four

four
```

第一遍扫描后， shell 把反斜杠去掉了 

当shell 再次扫描时候， 它替换了 $4 的值， 并执行了echo 命令

# 使用eval 创建指向变量的指针

```
x=100
ptrx=x
eval echo \$$ptrx 

100 打印100

eval $ptrx=50  将50存入到ptrx指向的变量x中

echo $x

50 打印50

```

# eval 在 /etc/sysconfig/network-scripts/ifup-eth 的应用

```
IPADDR=127.0.0.2
    
local i=0 val
for idx in '' {0..255} ; do
    ipaddr[$i]=$(eval echo '$'IPADDR$idx)
	i=$((i+1))
done


for idx in {0..256} ; do
	echo ${ipaddr[$idx]}
done
```

仔细研究， 实际上是生成了IPADDR， IPADDR0 到IPADDR255 这么多的变量，并把变量赋值到 ipaddr 这个数组中



