# shell语法

## 概览
Linux terminal中的命令行可以看成是一个“shell脚本在逐行执行”

Linux中常见的shell脚本有很多种，常见的有：

- Bourne Shell(/usr/bin/sh或/bin/sh)
- Bourne Again Shell(/bin/bash)
- C Shell(/usr/bin/csh)
- K Shell(/usr/bin/ksh)
- zsh
- …
Linux系统中一般默认使用bash，所以接下来讲解bash中的语法。
文件开头需要写`#! /bin/bash`，指明bash为脚本解释器。
执行方式分为三种：
- ./xx.sh   当前目录中执行
- 绝对目录执行  /home/xxx/xx/xx.sh
- 相对目录执行  ~/xx.sh

## 注释
```bash
# 这是一行注释
echo 'Hello World'  # 井号后的是注释

:<<EOF
第一行
这是多行注释
第三行
EOF
```

其中`EOF`可以换成其他任意字符串，例如
```bash
:<<abc
第一行
第二行
第三行
abc
```

## 变量

**定义变量**
定义变量，不需要加$符号，例如：
```bash
name1='yxc'  # 单引号定义字符串
name2="yxc"  # 双引号定义字符串
name3=yxc    # 也可以不加引号，同样表示字符串
```

**使用变量**
使用变量，需要加上\$符号，或者\${}符号。花括号是可选的，主要为了帮助解释器识别变量边界。
```bash
name=yxc
echo $name          # 输出yxc
echo ${name}        # 输出yxc
echo ${name}acwing  # 输出yxcacwing
echo $nameacwing    # 输出空，原因：nameacwing未定义
```

**只读变量**
使用`readonly`或`declare`可以将变量变为只读
```bash
name=yxc
readonly name
declare -r name # 两种写法均可

name=abc        # 会报错，因为此时name只读
```

**删除变量**
`unset`可删除变量

```bash
name=yxc
unset name
echo $name  # 输出空行
```

**变量类型**
1. 自定义变量（局部变量）：子进程不可访问
2. 环境变量（全局变量）：子进程可访问

自定义变量改成环境变量：
```shell
acs@9e0ebfcd82d7:~$ name=yxc  # 定义变量
acs@9e0ebfcd82d7:~$ export name  # 第一种方法
acs@9e0ebfcd82d7:~$ declare -x name  # 第二种方法
```
环境变量改为自定义变量：
```shell
acs@9e0ebfcd82d7:~$ export name=yxc  # 定义环境变量
acs@9e0ebfcd82d7:~$ declare +x name  # 改为自定义变量
```

**字符串**
字符串可以用单引号，也可以用双引号，也可以不用引号。

单引号与双引号的区别：

- 单引号中的内容会原样输出，不会执行、不会取变量；
- 双引号中的内容可以执行、可以取变量；

```bash
name=yxc # 不用引号
echo 'hello, $name \"hh\"' #单引号字符串，输出hello， $name \"hh\"
echo "hello, $name \"hh\"" #双引号字符串，输出hello， yxc "hh"
```

获取字符串长度
```bash
name="yxc"
echo ${#name}   # 输出3
```

提取子串
```bash
name="hello,yxc"
echo ${name:0:5} #提取从0开始的5个字符
```

TIPS:
1. 定义变量时，等号两边不能有空格
2. 定义变量时默认都是字符串，但当需要变量是整数时，会自动把变量转换为整数
3. `type+命令`可以解释该命令的来源（内嵌命令、第三方命令等），如
    ```bash
    type readonly # readonly is a shell builtin(shell内部命令)
    type ls # ls is aliased to 'ls -color+auto'
    ```

4. 被声明未只读的变量无法被unset删除
5. bash可用来开一个新的进程，可使用exit或Ctrl + d退出
6. 字符串中，不加引号和双引号效果相同

## 默认变量

在执行脚本时，可向脚本传递参数。`$1`是第一个参数， `$2`是第二个参数，以此类推。特殊地，`$0`是文件名（包含路径）。例如：

创建文件`test.sh`：
```bash
#! /bin/bash
echo "文件名：" $0
echo "第一个参数："$1
echo "第二个参数："$2
echo "第三个参数："$3
echo "第四个参数："$4
```

然后执行该脚本：
```bash
acs@9e0ebfcd82d7:~$ chmod +x test.sh 
acs@9e0ebfcd82d7:~$ ./test.sh 1 2 3 4
文件名：./test.sh
第一个参数：1
第二个参数：2
第三个参数：3
第四个参数：4
```

**其他参数相关变量**
| 参数           | 说明                                                |
| ------------ | ------------------------------------------------- |
| `$#`         | 代表文件传入的参数个数，如上例中的值为4                              |
| `$*`         | 由所有参数构成的用空格隔开的字符串，如上例中值为`“$1 $2 $3 $4”`           |
| `$@`         | 每个参数分别用双引号括起来的字符串，如上例中值为`"$1" "$2" "$3" "$4"`     |
| `$$`         | 脚本当前运行的进程ID                                       |
| `$?`         | 上条命令的退出状态（注意不是stdout，而是exit code）。0表示正常退出，其他值表示错误 |
| `$(command)` | 返回`command`这条命令的stdout(可嵌套)                       |
| `command`    | 返回`command`这条命令的stdout(不可嵌套)                      |

TIPS: 向脚本传递参数，参数大于1位时要用大括号括起来

## 数组
bash只支持一维数组，下标从0开始，初始化不需要指明数组大小，并可存放多个不同类型的值。


```bash
# 定义
array=(1 abc "def" 123)

# 也可以直接定义数组中的某个元素的值
array[0]=1
array[1]=abc
array[2]="def"
array[3]=123

#数组中的值
# 格式：${array[index]}
echo ${array[0]}
echo ${array[1]}
echo ${array[2]}
echo ${array[3]}

# 读取整个数组
echo ${array[@]} #第一种写法
echo ${array[*]} #第二种写法

#输出数组长度
echo ${#array[@]}   #第一种写法
echo ${#array[*]}   #第二种写法
```