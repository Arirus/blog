---
date : 2018-08-06T22:32:00+08:00
tags : ["玩儿", "笔记", "Linux"]
title : "玩儿 Linux常用命令"

---

上一章我们介绍了 Shell 的基本用法，本章中会继续上期的主题，介绍常用的几类命令。本篇文章有一部分参考了陈皓老师的博客[酷Shell](https://coolshell.cn)，在此表示感谢好了，我们继续。

<!-- more -->

### 基本命令
所谓基本命令是使用 Linux 绕不过去的，就像使用 Windows 文件夹的切换，文件查找。这里一些简单的命令就简单的写写好了：
```Shell
ls # 打印出当前目录下的所有文件（目录也算） 通常使用 ls -la 显示详细信息
rm file #删除file,同时 我们常跟 -rf 表示递归强制删除
mkdir dir # 创建一个目录 $ mkdir dir -m 600 则目录只对 User 有读写权限
touch file # 创建一个文件但是不支持修改其权限
cd base_path # 切换到相应的目录 `.` 表示当前目录 `..` 表示上级目录
```
上面这些命令是最最基本的，没了他们几户是不会操作 Linux 系统的，因此必须要记得。

### 正则表达式
正则表达式是一种用于文本匹配的形式小巧、具有高度针对性的编程语言。
我们把它单独拎出来，因为 Linux 下很多操作都是要依赖正则表达。Linux 上面支持 ERE(extended-regex) BRE(basic-regex) 和 PCRE(perl-compatible-regex) 前者属于 POSIX 规范下的,Unix 系统原生支持的类型，后者属于 Perl 语言衍生出来的，Unix也是支持的。

ERE与BRE最大的区别在于对于某些字符的处理：`(、)、{、}、+、?、|`这7个特殊字符的使用方法上。
在ERE中如果要这些字符**不表示特殊的含义**，就需要把它们转义，在BRE中如果想要这些字符**表示特殊的含义**，就需要把它们转义。`.、\、[、^、$、*、-` 在两类中的一样，**不表示特殊含义**，才需要将其转义。

以下以ERE为例说下规则：

|   正则符号    |       描述      |       事例      |
|   :--------:    |       ----        |       ----    |
|   ^           |       行起始标记   |   ^tux 匹配以tux起始的行     |
|   $           |       行尾标记   |   tux$ 匹配以tux结尾的行     |
|   .           |       匹配任意一个字符   |  Hack. 匹配Hackl和Hacki，但是不能匹配Hackl2和Hackil，它只能匹配单个字符     |
|   []           |       匹配包含在 [字符] 之中的任意一个字符   |   coo[kl] 匹配cook或cool     |
|   [^]           |       匹配除 [^字符] 之外的任意一个字符   |   9[^01] 匹配92、93，但是不匹配91或90     |
|   [-]           |       匹配 [] 中指定范围内的任意一个字符   |   [1-5] 匹配从1～5的任意一个数字     |
|   ?           |       匹配之前的项1次或0次   |   colou?r 匹配color或colour，但是不能匹配colouur     |
|   +           |       匹配之前的项1次或多次   |   Rollno-9+ 匹配Rollno-99、Rollno-9，但是不能匹配Rollno-     |
|   *           |       匹配之前的项0次或多次   |   co*l 匹配cl、col、coool     |
|   ()           |       创建一个用于匹配的子串   |   ma(tri)?x 匹配max或maxtrix     |
|   {n}           |       匹配之前的项n次   |   [0-9]{3} 匹配任意一个三位数，[0-9]{3}可以扩展为 [0-9][0-9][0-9]     |
|   {n,}           |       之前的项至少需要匹配n次   |   [0-9]{2,} 匹配任意一个两位或更多位的数字     |
|   {n,m}           |       指定之前的项所必需匹配的最小次数和最大次数   |   [0-9]{2,5} 匹配从两位数到五位数之间的任意一个数字     |
|      &#124;        |       交替匹配 &#124; 两边的任意一项   |   Oct (1st &#124; 2nd) 匹配Oct 1st或Oct 2nd     |
|      \        |       转义符可以将上面介绍的特殊字符进行转义   |   a\.b 匹配a.b，但不能匹配ajb。通过在 . 之间加上前缀 \ ，从而忽略了 . 的特殊意义     |
注意在字符集和 `[]` 如果要匹配 `.` 可不用转义，当然转义也没问题。

除此之外还有一些需要知道的，这些实在PCRE中定义的：

    `\b` Match the empty string at the edge of a word.
    `\B` Match the empty string provided it’s not at the edge of a word. （上面的反例）
    `\<` Match the empty string at the beginning of word. （第一条的一种情况，类似还有 `\>` ，Vim 不支持）
    `\w` Match word constituent.(匹配数字或者字母 等同于 `[a-zA-Z0-9_]`)
    `\W` Match non-word constituent.(不匹配数字或者字母 等同于 `[^a-zA-Z0-9]`)
    `\s` Match whitespace.(同样还是有 `\S`)

简单来说就是有一下规则：

    － 使用BRE语法的命令有：grep、ed、sed、vim
    － 使用ERE语法的命令有：egrep、awk、emacs

注意使用：`sed -r` 也可以使其支持 `ERE` 的语法。但是 Vim 比较奇特，由于其出现时间要早于 PCRE 因此是不完全兼容 PCRE 规范的，默认支持 `BRE`，在收缩的时候可以使用 `\v` 前缀使用 very magic 模式，支持 `ERE` ，4种模式效果如下
```Shell
after:    \v       \m       \M       \V         matches ~
                'magic' 'nomagic'
          $        $        $        \$         matches end-of-line
          .        .        \.       \.         matches any character
          *        *        \*       \*         any number of the previous atom
          ~        ~        \~       \~         latest substitute string
          ()       \(\)     \(\)     \(\)       grouping into an atom
          |        \|       \|       \|         separating alternatives
          \a       \a       \a       \a         alphabetic character
          \\       \\       \\       \\         literal backslash
          \.       \.       .        .          literal dot
          \{       {        {        {          literal '{'
          a        a        a        a          literal 'a'
```

### 文件系统命令
#### cat 命令
cat 命令是一个平日里经常会使用到的简单命令。它本身表示concatenate（拼接）。
最常用的用法

    cat file1 file2 file3
这样3个文件中的内容便拼接在了一起。

`cat -n fileName` 则会在文件内容之前显示行号

#### find 命令
find 命令的工作方式如下：沿着文件层次结构向下遍历，匹配符合条件的文件，执行相应的操作。

    `$ find base_path` 递归查找当前目录下所有的文件和目录
    `$ find base_path -name "*.txt"` 递归查找当前目录下所有名字包含 ‘.txt’ 的文件和目录
    `$ find base_path \( -name "*.txt" -o -name "*.pdf" \)` 同上，查找条件包含两个
    `$ find base_path ! -name "*.txt" `
    `$ find base_path -path "*/slynux/*"`  路径匹配，因此文件路径包含 slynux 都会被查找出 注意和 `"*slynux*"` 的区别，后者是路径中包含就可以类似：slynux1 也可以被匹配到，前者不可以。注意对于名字匹配，不能包括反斜杠，路径匹配是可以不包含反斜杠。
    `$ find base_path -regextype egrep -regex "pattern"` 用egrep 解析正则的方式解析 "pattern" 注意 find 正则匹配对应的是全匹配因此"base_path/"作为开头一定不能少
    `$ find base_path -type d` 查找当前目录下的所有目录 同理 `-type f`
    `$ find base_path -type f -size +2k` 查找目录下文件大于 2k 的文件。
    `$ find base_path -type f -name "*.swp" -delete` 把查找的文件直接删除
    `$ find base_path -type f -perm 644` 打印出权限为644的文件
    `# find base_path -type f -exec command {} \;` 当前目录下的所有文件都执行 command 命令。两点注意：{} 用来替代被查找到的内容，`\;` 用来结束，因为怕转义因此需要加反斜杠：`$ find . -type f -name "*.sh" -exec chmod 666 {} \;` 将当前目录下sh文件的权限转换为666
    `$ find bash_path \( -name ".git" -prune \) -o \( -type f -print \)`  -prune 跳过相应目录或文件。
注意：find 打印结果默认是以`\n`为结尾进行输出 使用`-print0`可以使用`\0`作为匹配之间的限定符。
给出一个例子，这也是官方推荐写法：
```Shell
find /tmp -name core -type f -print0 | xargs -0 /bin/rm -f
# Find files named core in or below the directory /tmp and delete them, processing filenames in such a way that file or directory names containing single or double quotes, spaces or  newlines  are  correctly  handled.
```

#### xargs 命令
xargs 能够处理 stdin 并将其转换为特定命令的命令行参数。基本用法：`command | xargs`。

```shell
$ echo "splitXsplitXsplitXsplit" | xargs -d X -n 2 # -d 字段分隔符 -n 输入划分成多行
split split
split split
```

##### 结合 find 使用 xargs
```shell
arirus@Arirus-Win:~/shell$ find . -type f -print0 | xargs -0 -I {} echo "file {}"
file ./.swo
file ./.swp
file ./ceho.sh
file ./eq.sh
file ./ex
file ./sleep.sh
```

#### chmod 命令
文件权限和所有权是Unix/Linux文件系统（如ext文件系统）最显著的特性之一。
```shell
arirus@Arirus-Win:~$ ls -la
drwxr-xr-x 1 arirus arirus  512 Aug  7 15:38 .
drwxr-xr-x 1 root   root    512 Feb 24 20:49 ..
-rw-rw-rw- 1 arirus arirus  293 Aug  6 21:07 .bash_profile
-rw-r--r-- 1 arirus arirus 3771 Feb 24 20:49 .bashrc
-rw-rw-rw- 1 arirus arirus    0 May 30 15:57 out.txt
drwx------ 1 arirus arirus  512 Aug  7 16:07 shell
drwx------ 1 arirus arirus  512 Jul 25 10:37 .ssh
```
前面列出来了各个文件的权限；第1位 表示文件类型：d 目录 - 普通文件 l 链接文件 等
2-4位，表示文件所有者的读、写、执行权限，5-7位，表示用户组的读、写、执行权限，最后3位表示其他用户的读、写、执行权限。例如 .ssh 是一个文件夹，当前用户拥有读、写、执行权限，用户组其他用户，别人没有权限。
```shell
$ chmod u=rwx g=rw o=r filename  # User读写执行 Group读写 Others读
$ chmod o+x filename  # Others 增加执行
$ chmod a+w filename  # 所有人 增加写
$ chmod a-r filename  # 所有人 去掉读
$ chmod 764 filename  # User读写执行 Group读写 Others读
```
### 文本操作命令
#### grep 命令
grep 命令作为Unix中用于文本搜索的神奇工具，能够接受正则表达式，生成各种格式的输
出。会只输出符合表达式的行，多数用在过滤信息上。
```shell
$ grep "pattern" filename # 在filename中查找匹配 pattern 的单词所在行，并打印出来
$ echo this is a line. | egrep -o "[a-z]+\." # 可以从标准输入中读取，-E 支持正则表达式搜索， -o 支持打印出相应的单词而不是单词所在的行
$ grep "pattern" -n filename # 打印出所在行的行号
$ grep "bash" . -rn # 在当前的目录下进行递归搜索，并打印出行号
$ echo hello world | grep -i "HELLO" # 搜索时忽略大小写
```
注意在使用 grep 上使用正则表达式可以选择针对不同的方言：-E -G -P 分别对应ERE BRE PCRE
```shell
$ ls -la # 显示当前目录下文件
drwx------ 1 arirus arirus 512 Aug 10 14:40 .
drwxr-xr-x 1 arirus arirus 512 Aug 10 14:40 ..
drw------- 1 arirus arirus 512 Aug 10 09:56 dir
-rw-rw-rw- 1 arirus arirus   5 Aug 10 14:41 file
-rw-rw-rw- 1 arirus arirus   0 Aug 10 13:46 file2 file1
-rw-rw-rw- 1 arirus arirus   5 Aug 10 16:42 file3
-rw-rw-rw- 1 arirus arirus   0 Aug 10 13:47 file4\nfile5
-rw-rw-rw- 1 arirus arirus   0 Aug 10 13:48 file \n file1
$ grep "file" . -r # 递归查找当前目录下的文件，查找字符串 "files"
./file3:file
$ ls -la | grep -E "\bfile[0-9]+\b" --color
#查找到了4个文件 由于 grep 匹配文本是部分匹配 不同于 find 整体匹配，因此第1，3，4个文件被匹配进来了
-rw-rw-rw- 1 arirus arirus   0 Aug 10 13:46 file2 file1
-rw-rw-rw- 1 arirus arirus   5 Aug 10 16:42 file3
-rw-rw-rw- 1 arirus arirus   0 Aug 10 13:47 file4\nfile5
-rw-rw-rw- 1 arirus arirus   0 Aug 10 13:48 file \n file1
$ find -regextype egrep -regex "./file[0-9]+" # 因为find是全匹配 因此可以使用其来做到精确匹配。
./file3
```
毕竟 grep 是文本查找而不是针对单个文件的操作，所以可以理解。
```shell
$ grep -E "\bbash\b" . -rn -A3 --colour -B3 -m2
# 在当前的目录下面递归查找（. -r） 每个文件前两个(-m2)匹配 “bash”(-E "\bbash\b") 的内容 匹配内容用颜色标记(--colour)，同时打印出匹配内容的前3行(-B3)和后3行(-A3) 并标示出行号(-n)
```

### sed 命令
sed 是流编辑器（stream editor）的缩写。 sed 命令众所周知的一个用法是进行文本替换。
#### 匹配替换
```shell
$ sed 's/pattern/replace_string/ng' file  # s命令 将file中匹配 pattern 的部分替换为 replace_string ，最后表示从第几个匹配位置进行替换
$ echo "testtesttesttest" | sed "s/test/Test/2g"
testTestTestTest #从第二个位置起将 `test` 替换为 `Test`,如果不带ng则仅替换一行中的第一个，g则替换所有的
$ echo "testtesttesttest" | sed "s/test/Test/2"
testTesttesttest # 只替换第2个别的不换
$ echo "testtesttesttest" | sed "2s/test/Test/2"
testtesttesttest # 替换第2行的第2个test为Test 因为没有第二行所以不换。

# 多匹配
$ echo "testAtestAtestAtest" | sed "s/test/Test/g; s/A/B/2g" # 同单匹配 中间用；分割开来
TestATestBTestBTest
$ echo "testAtestAtestAtest" | sed  -e"s/test/Test/g" -e"s/A/B/2g" # 作用同上 不过是把两个替换命令拆开而已
# 保存替换到文件
$ sed -i 's/text/replace/' file # 将修改后的文件保存到某个位置
$ sed –i .bak 's/text/replace/' file # 文件保存到file中，同时原先有一个备份到 .bak 中

# 使用ERE来匹配替换
$ echo this is an example | sed -r 's/\w\+/[&]/g'
[this] [is] [an] [example] # 用 & 标记匹配样式的字符串 ，这里匹配到了对象直接用[]包围并返回
# 注意在匹配正则时候加上 -r ，不然某些特殊字符会被转义（因-r 表示使用ERE，而非BRE，{} +？可以被正确的转义）
# 也可以使用 -E 用来匹配ERE

$ cat grep_file
shell/ceho.sh:1:#!/bin/bash
shell/eq.sh:1:#!/bin/bash
shell/sleep.sh:1:#!/bin/bash
$ sed -r "s/\bsh\b/\{&\}/g" grep_file # 将所有的 sh 用 {&} 来替换 使用$表示之前匹配的内容不用再写一遍
shell/ceho.{sh}:1:#!/bin/bash
shell/eq.{sh}:1:#!/bin/bash
shell/sleep.{sh}:1:#!/bin/bash
sed -r "s/(\bsh\b).*(\bbin\b)/\{\1\.\.\.\2\}/g" grep_file # （）括起来的正则表达式（注意转义）所匹配的字符串会可以当成变量来使用，sed中使用的是\1,\2… 这里将sh到bin之间所有的字符串当作搜索匹配条件，替换为{sh...bin}
shell/ceho.{sh...bin}/bash
shell/eq.{sh...bin}/bash
shell/sleep.{sh...bin}/bash
```

#### 添加、插入和删除
除了上面对于文本的替换，还可以对文本进行添加、插入和删除操作
```Shell
# a命令就是append， i命令就是insert，d命令就是delete，它们是用来添加，插入和删除行的。
$ sed "2 i New insert line" grep_file # 在第二行（2）的位置插入（i）
shell/ceho.sh:1:#!/bin/bash
New insert line
shell/eq.sh:1:#!/bin/bash
shell/sleep.sh:1:#!/bin/bash
# 同理 append 在第二行后面添加新的行
$ sed -r "/[a-z]{5}/ a New insert line" grep_file # /pattern/ 匹配内部pattern部分，匹配到便在后面进行append 操作，因为每一行都有 `shell` 因此每一行后面都会加上一个新的行。
shell/ceho.sh:1:#!/bin/bash
'New insert line'
shell/eq.sh:1:#!/bin/bash
'New insert line'
shell/sleep.sh:1:#!/bin/bash
'New insert line'
```
#### 显示
既然是stream editor,那它当然也有显示功能
```Shell
$ sed "" filename # 便可以将文件内容打印出来，因为我们发送了空命令，因此 sed 便什么也不做，把文件内容打印了出来
$ sed "1p" grep_file #便可打印出第一行的内容，同时还把整个文件也打印了出来
shell/ceho.sh:1:#!/bin/bash
shell/ceho.sh:1:#!/bin/bash
shell/eq.sh:1:#!/bin/bash
shell/sleep.sh:1:#!/bin/bash

dsdsf

sww
#如果不需要可以使用 -n 参数（--slient） 不会打印出多余信息
```

#### 小结
sed命令应该是当前介绍的最详细的命令，根本原因在于其比较复杂。毕竟作为一个 editor 如果最基本查找替换，删除等等都需要单独介绍。
这里我们简单的做个基本用法总结：

```
sed -r -i -e "command" -e "command;command" fileName
command :
      "n,mp" 显示n-m行的内容；
      "n,ms/pattern/replaceMent/ig" 将n-m行的第i个pettern替换为replaceMent；
      "n,m[i,a] newContent" 将n-m行[插入,添加] newContent
      "n,md " 将n-m行删除
```

### awk 命令
awk 比sed还要复杂，他的结构几乎可以类比一个类：有初始化，有处理逻辑，有析构
我们先将lsof的前8行内容保存起来，作为模板：
```Shell
$ cat lsof_file
COMMAND PID   USER   FD   TYPE DEVICE    SIZE              NODE NAME
init      1   root  cwd    DIR    0,2     512  5910974511593519 /
init      1   root  rtd    DIR    0,2     512  5910974511593519 /
init      1   root  txt    REG    0,2   87944   281474977546191 /init
init      1   root  mem    REG    0,0                    835535 /init (path dev=0,2, inode=281474977546191)
init      1   root NOFD                                         /proc/1/fd (opendir: Permission denied)
init      3   root  cwd    DIR    0,2     512  5910974511593519 /
init      3   root  rtd    DIR    0,2     512  5910974511593519 /
```

#### 基本操作
如果我们想要打印出某几列可以如下操作：
```Shell
$ awk '{print $1,$5,$6}' lsof_file
#两点需要注意：awk 需要使用 ``引用起来而不能是"";需要使用{}将执行的操作括起来，这里的意思就是打印出文件中第1，5，6列数据。
COMMAND TYPE DEVICE
init DIR 0,2
init DIR 0,2
init REG 0,2
init REG 0,0
init /proc/1/fd (opendir:
init DIR 0,2
init DIR 0,2
```
文本中倒数第3项由于 `TYPE` `DEVICE` `SIZE` `NODE` 都是缺失列，因此会打印出 `NAME` 列，因为各个列之间默认都是使用空格进行分隔，所以awk 并不知道哪些是应该属于同一列的，哪些应该一起显示，所以使用只会根据空格进行分隔。

#### 过滤记录
所谓过滤记录就是有条件的打印：
```Shell
$ awk '$2>1 {print } ' lsof_file #把第2列中大于1的数据打印出来
COMMAND PID   USER   FD   TYPE DEVICE    SIZE              NODE NAME
init      3   root  cwd    DIR    0,2     512  5910974511593519 /
init      3   root  rtd    DIR    0,2     512  5910974511593519 /
# 如果要打印出整行的话，是不需要{print}来表示的 可以简写成 $ awk '$2>1' lsof_file
#当然既然有条件，也是支持逻辑与或的
$  awk '$2>1 || $4=="rtd" ' lsof_file
COMMAND PID   USER   FD   TYPE DEVICE    SIZE              NODE NAME
init      1   root  rtd    DIR    0,2     512  5910974511593519 /
init      3   root  cwd    DIR    0,2     512  5910974511593519 /
init      3   root  rtd    DIR    0,2     512  5910974511593519 /
# 这样便把第二列大于1 或者 第四列等于rtd 的行全部打印出来了
```
注意：上面进行比较都是将比较的对象转换成ascii进行的比较。例如对于第二列的首行 PID 转换成 ascii 要大于1转换的ascii，因此对于第一条shell命令，第一行会打印出来。

#### 内建变量
上面使用了 `$1` 表示第一个参数，其是使用了内建变量，所谓内建变量可以理解成 awk 方法提供的默认变量，这里列出几个最常用的内建变量

|   变量  |   说明  |
| :-----:| ---- |
| $0 |当前记录（这个变量中存放着整个行的内容）|
|$1~$n   | 当前记录的第n个字段，字段间由FS分隔  |
| FS  |  输入字段分隔符 默认是空格或Tab |
|RS   |   输入的记录分隔符， 默认为换行符 |
| NF  |  当前记录中的字段个数，就是有多少列 |
|NR   |  已经读出的记录数，就是行号，从1开始，如果有多个文件话，这个值也是不断累加中。 |
|FNR   | 当前记录数，与NR不同的是，这个值会是各个文件自己的行号  |
|  OFS | 输出字段分隔符， 默认也是空格  |
|ORS   |  输出的记录分隔符，默认为换行符 |
|FILENAME   | 当前输入文件的名字  |

注意FS之后的变量无需使用 `$FS` 来显示相应的值，其值本身即是相应的的值
```Shell
$ cat sed_file
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
# 我们将 passwd 里面的前4行存入到 一个新的文件里面。
$ awk  'BEGIN{FS=":";OFS="\t"} {print NR,$1,$3,$6,FILENAME}' sed_file
#设置 输入分隔符为： 输出分隔符为\t 打印出所在行的行号 文件名 和 1，3，6列的内容
1       root    0       /root   sed_file
2       daemon  1       /usr/sbin       sed_file
3       bin     2       /bin    sed_file
4       sys     3       /dev    sed_file
```
#### awk 结构
上面可以看到我们把输入、出分隔符放到了名为 `BEGIN` 的结构里，awk的结构如下所示：
```Shell
$ awk ' BEGIN{ commands } pattern { commands } END{ commands }' file
```
可以看出他有初始化，逻辑操作，输出三个部分组成，之前我们上面的部分都是在 `pattern { commands }` 来进行操作，而这次我们是在`BEGIN`部分进行的初始化。
```shell
$ gawk 'BEGIN{print "start" }/\<bin\>/ {print NR,$0,FILENAME} END{print "end"}' sed_file
start
1 root:x:0:0:root:/root:/bin/bash sed_file
3 bin:x:2:2:bin:/bin:/usr/sbin/nologin sed_file
end
```
这次除了打印匹配的内容之外，还打印出了 start 和 end ，对于做统计是非常常用的。除此之外还使用了 \pattern\
来进行字符串正则匹配：只有匹配pattern的行才能打印出来
```Shell
$ gawk 'BEGIN{print "start";FS=":" }/\<bin\>/ {print NR,$0,FILENAME;print >$1} END{print "end"}' sed_file
start
1 root:x:0:0:root:/root:/bin/bash sed_file
3 bin:x:2:2:bin:/bin:/usr/sbin/nologin sed_file
end
```
这样在pattern的command中便将查找的内容重定向到了相应的文件中了

#### 外部变量
awk 可使用外部变量（包括环境变量），使用 -v 选项来设置awk变量，注意使用 ENVIRON 需要导出。
```Shell
$ x=5

$ y=10
$ export y

$ echo $x $y
5 10

$ awk -v val=$x '{print $1, $2, $3, $4+val, $5+ENVIRON["y"]}' OFS="\t" score.txt
Marry   2143    78      89      87
Jack    2321    66      83      55
Tom     2122    48      82      81
Mike    2537    87      102     105
Bob     2415    40      62      72
```

本篇中，我们主要介绍了正则表达式，文件查找命令，参数格式化命令，权限变更命令，还有文本相关命令grep，awk，sed 命令
