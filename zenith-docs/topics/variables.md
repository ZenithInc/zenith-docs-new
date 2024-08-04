# 变量

这一篇文档，我们来描述如何去定义以及使用变量。Shell 中的变量大致分为自定义变量、位置变量以及环境变量。将在下文中分别加以描述。

### 自定义变量 {id="custom-variable"}

首先，来说一下变量命名的规则:

- 变量是由任何字母、数字和下划线组成的字符创，且不能以数字开头。
- 变量名称是严格区分大小写的，例如 Var1 和 var1 是不同的。
- 变量、等号、值中间不能出现任何空格。

变量如何定义和使用呢？
```shell
say='Hello World'
echo $say ## Output: Hello World
```
Shell 并不是一种十分严谨的脚本语言，不管你的变量类型是什么，对于 Shell 来说，都会作为字符串处理，比如下面这个例子:
```shell
num1=123
num2=456
echo $num1+$num2 ## 123+456
```
下面是一些错误的示例，在写脚本的过程中需要注意:
```shell
## 赋值语句等号的两边都不能有空格
count = 123  ## error
count= 123 ## error
count =123 ## error
## 变量名不能使用数字开头
1count=2	## error
```
另外，单个变量中可以存储多个值，并且使用数字作为索引进行遍历或单个引用，我们称之为数组。定义和使用如下:
```shell
arr_var=(a b c d e f g)
echo $arr_var  ## 输出第一个元素
echo ${arr_var[1]}  ## 输出第二个元素，索引从 0 开始
echo ${arr_var[*]}  ## 输出所有的元素: a b c d e f g
arr_var[3]=dd  ## 对制定索引重新赋值
echo ${arr_var[3]} ## 输出重新赋值之后的值: dd
```
> 需要注意的是，在 Ubuntu 中，默认使用的是 DASH 而不是 BASH, 以上数组语法会不支持。所以要执行脚本需要指定为 `bash script.sh` ，而不是 `sh script.sh` 。


### 位置变量 {id="position-variable"}

什么是位置变量呢？当一条命令或脚本执行时，后面可能会有多个传递的参数，而我们可以使用 Shell 内建的这些变量来引用这些参数。
```shell
cat p.sh ## Output:
#!/bin/bash
echo $0
echo $1
sh p.sh p1 ## Output:
p.sh  ## $0 表示第一个参数，即 Shell 文件名本身