---
description: shell if 语法
---

# if语法

## 基本语法

```text
if [ 条件表达式 ]; then
 Command
else
 Command
fi
```

## 条件表达式

### 文件表达式

if \[ -f file \] 如果文件存在

if \[ -d dir \] 如果目录存在

if \[ -s file \] 如果文件存在 并且 非空

if \[ -r file \] 如果文件存在 并且 可读

if \[ -w file \] 如果文件存在 并且 可写

if \[ -x file \] 如果文件存在并且可执行

### 整数变量表达式

if \[ int1 -eq int2 \] 如果int1 等于 int2

if \[ int1 -ne int2 \] 如果int1 不等于 int2

if \[ int1 -ge int2 \] 如果 &gt;= , \(ge : greate equal\)

if \[ int1 -gt int2 \] 如果 &gt; \(gt: greate than\)

if \[ int1 -le int2 \] 如果 &lt;=

if \[ int1 -lt int2 \] 如果 &lt;

### 字符串变量表达式

if \[ $a = $b \] 如果 a 等于 b 字符串允许使用赋值号做等号

if \[ $a != $b \] 如果 a 不等于 b

if \[ -n $string \] 如果string 非空

if \[ -z $string \] 如果string 为空

if \[ $string \] 如果 string 非空 和 -n 类似

### 逻辑表达式

逻辑 非 !

if \[ ! 表达式 \] 条件表达式 相反

逻辑与 -a

if \[ 表达式1 -a 表达式2 \] 表达式1 and 表达式2

逻辑或 -o

if \[ 表达式1 -o 表达式2 \] 表达式1 或（or）表达式2

