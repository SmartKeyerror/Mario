---
layout: post
cover: false
navigation: true
title: DevOps基础(1)--Shell脚本编程
date: 2019-06-17 09:31:51
tags: ['Shell', 'DevOps']
subclass: 'post tag-fiction'
author: smartkeyerror
categories: smartkeyerror
---

由于Docker容器以及Kubernetes容器编排服务的蓬勃发展， 服务器以及业务服务的运维不再是运维工程师的专属， 业务的开发工程师也必须加入到运维的领域之中， 与运维工程师合作， 形成一套完整、高效的自动化运维与部署的系统。 而在我看来， 传统的运维工程师将会逐渐被应用开发工程师所取代， 因为Kubernetes赋予了开发人员强大的负载均衡、自动横向拓展以及高效管理的相关功能。 而在这些宏大的系统建设之前， Shell编程是无论如何都离不开的话题。

<!---more--->

#### 1. Shell变量
作为一个后台开发人员， Shell脚本既陌生由熟悉， 毕竟Linux命令哪个后台开发不会接触呢？ 将一个又一个的Linux命令收集起来， 并使用一些粘合剂进行组合， 最终就得到了Shell脚本。

Shell和Python语言一样， 是一个弱类型语言， 也就是说一个变量可以对其进行任意的类型赋值:

```bash
smart@Zero:~$ foo="bar"
smart@Zero:~$ echo $foo
bar
smart@Zero:~$ foo=10
smart@Zero:~$ echo $foo
10
```

在Terminal中， Shell命令就是一个天然的类似于Python的IPython环境， 如果我们想要对Python的某些语法进行测试的话， 需要进入Python或者IPython环境中， 而对于Shell而言， 打开Terminal就是自己工作的海洋。

在Shell中， 变量的赋值与其它语言没什么区别， 只不过获取变量的方式稍有不同而已。 我们可以认为`foo`变量是值`"bar"`的一个引用， 而要获取变量值， 需要借助引用名加上`$`符号， 非常类似C的指针。

在Shell编程的推荐使用方法中， 使用`${foo}`的方式获取变量内容， 多加一个大括号， 这样一来能够更加清楚的界定变量名称的范围， 不至于出现一些奇奇怪怪的问题。

除了我们自己定义的变量以外， 在Linux运行时， 还会预先定义一系列的环境变量。 环境变量说的简单一些就是定义在某一个文件中， 供整个Linux使用的变量， 可以认为是一种最高层的全局变量。

```bash
smart@Zero:~$ env
...
WORKON_HOME=/home/smart/.virtualenvs
HOME=/home/smart
PATH=/home/smart/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:/usr/local/go/bin/
```

获取当前系统的环境变量也简单， 敲`env`即可。 在上面的结果中， `WORKON_HOME`是virtualenvwrapper的工作目录， 是我定义在`~/.bashrc`中的， 而`HOME`和`PATH`变量， 则是Linux操作系统定义的。

获取系统的环境变量和获取自己定义的变量一样， `$`符+变量名:

```bash
smart@Zero:~$ echo $HOME
/home/smart
```

值得一提的就是`PATH`变量， 在安装一些软件时， 例如Java， Go时， 都需要将一些变量加入到`PATH`中， 为什么这么做？ Linux系统会在`PATH`变量值的路径中寻找可执行的二进制文件， 而当我们把诸如`GOPATH`的变量值假如到`PATH`变量中以后， 在任何的目录下， 都可以使用Go的相关命令， 这就是`PATH`变量的作用。


#### 2. 获取系统函数的返回值
诸如`cat`, `du`, `date`等命令， 实际上就是函数， 只不过是由C编写并通过某种方式暴露给用户而已。

`date`函数用以获取当前时区的时间:

```bash
smart@Zero:~$ date
2019年 05月 12日 星期日 10:43:48 CST
```

在编写Shell脚本时， 很多时候都需要将函数的运行结果保存在某一个变量中， 所以Shell提供了两种方式进行结果的赋值:
1. 使用variable=&#96date&#96
2. 使用variable=$(date)

```bash
smart@Zero:~$ foo=`date`
smart@Zero:~$ echo $foo
2019年 05月 12日 星期日 10:49:21 CST
smart@Zero:~$ echo $foo
2019年 05月 12日 星期日 10:49:21 CST
```

如果查看`date`的manual手册的话， 会发现它还支持日期的格式化:

```bash
smart@Zero:~$ foo=`date +%y%m%d%H%M%S`
smart@Zero:~$ echo $foo
190512105105
```

此外， Shell还提供了对上一个命令所执行结果的获取， 使用`$?`进行获取。 这是什么意思？ 在Shell中， 一条命令如果正常执行的话， 返回值将会是0， 如果命令执行时出现了某些错误的话， 返回值将会大于0， 且小于255。

```bash
# 执行一条正常的命令
smart@Zero:~$ echo "Hello World"
Hello World
smart@Zero:~$ echo $?
0

# 执行一条会抛出错误的命令
smart@Zero:~$ ls -alh NotExistFile
ls: cannot access 'NotExistFile': No such file or directory
smart@Zero:~$ echo $?
2
```

由于NotExistFile是一个不存在的文件， 所以`ls`命令会产生一个标准错误并输出至屏幕中， 此时的退出状态码将会为2。 一些常见的退出状态码如下:

| 状态码 | 描述                 | 状态码 | 描述                 |
| ------ | -------------------- | ------ | -------------------- |
| 0      | 命令成功结束         | 126    | 命令不可执行         |
| 1      | 一般性未知错误       | 127    | 没找到命令           |
| 2      | 不合适的shell命令    | 130    | 通过Ctrl+C退出的命令 |


#### 3. 流程控制
既然是一种语言， 又怎么能少的了流程控制。 在Shell脚本中， 使用最为广泛的恐怕就是`if-then`判断了。

##### 3.1 if-then
条件语句的基本模板:

```bash
if command-1
then
  command-2
else
  command-3
fi
```

需要特别注意的是， 这里的条件判断是command-1这条命令的执行结果: 如果command-1执行的退出状态码为0的话， 执行then语句块的内容， 否则退出。

```bash
#!/bin/bash

if ls -alh NotExistFile
then
  echo "The ls command exec successed"
else
  echo "Some error happened when exec ls"
fi
```

由于`ls -alh NotExistFile`的退出状态码为2， 所以将会输出"Some error happened when exec ls"。 如果我们想要true/false的条件语句， 使用`[[  ]]`。 例如， 如果变量`foo`的值为"bar"的话， 打印一条语句， 否则什么都不做:

```bash
#!/bin/bash

bar="foo"
if [[ ${bar} = "foo" ]]
then
  echo "Right"
fi
```

与传统的语言都不同的是， 判断两个变量是否相等使用的是单个`=`号， 而不是`==`， 需要注意。

Shell也提供了一些参数来帮助我们进行条件判断， 例如`-n str`表示检查str的长度是否大于0， `-z str`表示检查str的长度是否为0， `-d file`用以检测file是否存在并且是一个目录, `-e file`判断file是否存在...

```bash
if [[ -n ${bar} ]]
then
  echo "The length of the bar is not zero"
fi

if [[ -d "/home/smart" ]]
then
  echo "/home/smart exist, and it's a directory"
fi
```

##### 3.2 case语句
有时候变量的值会有多种， 如果一个一个的写`if`的话太麻烦了， 所以就有了`case`语句， 基本模板:

```bash
case variable in
A | B) command-1 ;;
C) command-2 ;;
D) command-3 ;;
*) default-command ;;
esac
```

注意一下语法格式就好， 没有什么特别复杂的地方。

##### 3.3 while语句
while语句的基本模板:

```bash
while condition
do
  command
done
```

condition的种类与`if-then`语法相同， 既可以判断命令的退出状态码， 也可以使用`[[  ]]`的形式来进行true/false判断:

```bash
# 一个无限循环
while [[ -n ${bar} ]]; do
    echo "The length of bar is not zero"
done
```

##### 3.4 for循环
`for`循环的语法格式更贴近于Python， 其模板为:

```bash
for var in list
do
  command
done
```

例如使用通配符来生成文件列表， 然后遍历， 当遍历的文件是一个目录时， 打印它:

```bash
for file in /home/smart/*
do
  if [[ -d ${file} ]]
  then
    echo "${file} is a directory"
  fi
done
```

也可以使用C语言风格的循环语句:

```bash
for (( i = 0; i < 10; i++ )); do
    echo "${i}"
done
```

#### 4. 处理用户输入与重定向
向脚本传递用户的参数是一个shell脚本最基本的操作， 脚本获取参数的方式也与其它语言不同。 诸如Java， 参数是以字符数组的方式传递给main函数的。

在shell中， 使用`$1`来获取第一个参数， `$2`获取第二个参数, ...， `$n`获取第n个参数。 而`$0`比较特殊， 代表了执行该脚本的路径名称。

```bash
#!/bin/bash
# test.sh
echo ${0}, ${1}, ${2}, ${3}
```

在赋予了普通用户对该脚本的执行权限后， 执行该脚本: `./test.sh A B C`， 将会得到输出:

```bash
./test.sh, A, B, C
```

对于`$0`， 如果只想要获取脚本名称的话， 可以使用`$(basename ${0})`。 获取参数个数使用`$#`， 获取所有参数使用`$*`或者是`$@`， 前者如果使用`"$*"`进行引用的话， 将会作为一个字符整体对待， 而`$@`不管在何种情况下， 都是参数所组成的列表， 所以`$@`更多的用于参数的迭代。

提到参数处理， 就不得不提及`shift`关键字。在使用`shift`命令时,默认情况下它会将每个参数变量向左移动一个位置。所以,变量\$3的值会移到\$2中,变量\$2的值会移到\$1中,而变量$1的值则会被删除。

`shift`的测试也很简单， 非常清楚的就能够知道它到底做了什么:

```bash
#!/bin/bash

echo "All param: $@"
shift
echo "The first shift: $@"
shift
echo "The second shift: $@"
shift
echo "The third shift: $@"
```

这次多传递一些参数进入该脚本， 得到的输出:

```bash
smart@Zero:~$ ./test.sh A B C D E F
All param: A B C D E F
The first shift: B C D E F
The second shift: C D E F
The third shift: D E F
```

每执行一次shift， 参数列表的首个参数都会被弹出， 如果执行`shift 2`的话， 将会弹出2个参数。

在`Ansible`的ad-hoc模式中， 通常我们会这样执行命令:

```bash
# 将ansible所管理的所有主机进行文件拷贝, 并发数为10
ansible all -m copy -a "src=/home/smart/monitor/ dest=/home/monitor" -f 5
```

在有了shift之后， 就可以很轻松的编写出对应的shell脚本了:

```bash
#!/bin/sh
# simulate_ansible.sh
# 运行: ./simulate_ansible.sh all -m copy -a "src=/home/smart/monitor/ dest=/home/monitor" -f 5

echo "Get params: $@"
while [[ $# -gt 0 ]]; do
  case $1 in
  all)
    echo "The process host group: ${1}"
    shift ;;
  -m)
    echo "Get module name: ${2}"
    shift 2 ;;
  -a)
    echo "Get parameter: ${2}"
    shift 2 ;;
  -f)
    echo "The fork number is: ${2}"
    shift 2 ;;
  *)
    echo "Bad params"
    exit 2
  esac
done
```

在Linux I/O中， 标准输入使用0表示， 标准输出使用1表示， 标准错误使用2表示。 什么是标准输出/错误? 使用`ls`命令得到的结果就是标准输出， 使用`ls NotExistFile`命令得到的结果就是标准错误。

Shell脚本在执行时， 许多时候都是边缘触发或者是定时执行的， 其标准输出与错误我们是看不到的， 所以就需要有日志进行记录。 一个记录标准输出， 一个记录标准错误:

```bash
ls -alh NotExistFile 1>~/monitor/stdout.log 2>~/monitor/stderror.log
```

有时候想偷个懒， 不管是输出还是错误， 都重定向到同一个文件:

```bash
ls -alh &>~/homo/monitor/ls.log
```

#### 5. 函数
shell中的函数并没有很强大的功能， 更像是一个小型的shell脚本。

```bash
# 定义
funcname() {...}
# 调用与参数传递
funcname "foo" "bar"
```

#### 6. 常见的shell脚本头设置
有时会看到在某些shell脚本中有这样的语句:

```bash
set -e
set -x
exec &> test.log
```

`set`以及`exec`主要是对当前脚本的一些全局设置， 所以会放到脚本开始的地方。

`set -e`表示如果当前的脚本在执行某一条命令时的退出状态码不为0时， 则整个脚本退出。 有些类似于异常的抛出与进程终止。

```bash
#!/bin/bash
set -e
ls -alh
ls -alh NotExistFile
echo "Done"  # 永远不会执行到该行命令
```

`set -x`则主要用于进行DEBUG， 在脚本执行时将会打印出每一行命令执行的详细信息。

`exec &> test.log`则表示将当前脚本执行时所产生的所有标准输出与错误均重定向至test.log文件。

#### 7. 子shell
假如我们编写了这样的一个shell脚本:

```bash
#!/bin/bash
cd /home/smart/monitor
```

然后执行该脚本， 会发现当前的目录并没有发生改变， 为什么? 这是因为不管是使用`bash script.sh`执行还是使用`./script.sh`来执行脚本， 脚本的执行都在一个名为子shell的shell环境中执行。 子shell中执行`cd`命令， 并不会影响到当前的shell状态。

#### 8. 小结
从我的工作经验上来看， 如果是开发来兼职做运维工作的话， 以上的内容完全能够解决日常中需要的运维场景。 Shell脚本语言本身比较简单， 其核心仍然是一个又一个的Linux系统命令， Shell语言只是作为粘合剂将这些命令组合起来形成一个整体而已。

PS: 留一张思维导图作为自己的复习参考

![](https://smartkeyerror.oss-cn-shenzhen.aliyuncs.com/Blog/ShellScript.png)