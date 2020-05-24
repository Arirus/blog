---
date : 2018-08-03T13:59:00+08:00
tags : ["玩儿", "笔记", "Linux"]
title : "玩儿 Linux基本操作"

---

### 为什么要写这个系列

这个系列怎么说呢，和移动开发（Android方向）不怎么沾边，不会这些大致不会有什么影响对于开发。
但是，我个人觉得很有意思，毕竟当前两大移动操作平台都是在 Unix 系统上发展起来，了解一些 Unix 的文件系统，基本操作还是很有意思的。
其次，对于有心在后端发展的话，服务器是必不可少的（当然使用 Mac 性质一样），这些地方都是使用 Linux 系统的"重灾区"，因此我决定要写这个系列。
当然这个系列以实用为主，例如基本命令，shell 脚本，简单工具使用等等，不会涉及复杂的东西。
同时笔者之前有很浅薄的 Linux 基础，本次的系列文章也算是一次系统性的学习了。
本系列所有所有的操作均在 Win10 上提供的 Ubuntu 系统上进行。
<!-- more -->

### Linux 文件系统
Linux 文件系统
第一次使用 Linux 的同学，可能都会对其文件系统产生兴趣，因为其完全和 Win 下的文件系统不一致。Win 下以盘符作为区分逻辑分区。而 Linux 下大部分情况下是没有分区的概念，都储存在单个目录结构中，成为虚拟目录。
```shell
$ tree -L 1  # 根目录下的虚拟目录如下
.
├── bin  # 二进制目录 用户级的GUN工具 ls cat bash grep touch 等等
├── boot # 启动目录
├── dev #设备目录 stdin stderr stdout null 都是定义在这里
├── etc # 系统配置文件目录 ssh vim 等配置文件都写在这里
├── home # 主目录，用户创建目录则生成在这里
├── lib # 库目录 存放系统或应用文件的库文件
├── lib64
├── media
├── mnt # 挂载目录 存放可移动设备的挂载点。因为我是在 Win 带的 Linux 查看的，因此上面挂载了 c d 两个盘符
├── opt
├── proc # 放置了系统的相关信息，当前进程的相关信息
├── root # 超级用户的主目录
├── run
├── sbin # 系统二进制目录 放了只有超级用户才能使用的GUN工具
├── snap
├── srv
├── sys
├── tmp # 临时目录
├── usr # 用户安装软件的目录
└── var # 可变目录，放入经常变化的文件，如放了注入log cache 等内容的文件
```
而在上述 `/usr` 目录下面，又有如下的结构：
```shell
$ tree -L 1
.
├── bin
├── games
├── include
├── lib
├── local
├── sbin
├── share
└── src
```
同时在 `./local` 目录下面也有类似的机构。这里就来简单说说，`/bin` ，`/usr/bin` ，`/usr/local/bin` 三者的区别。通常来说系统级别的工具放在 `/bin` 下面，例如ls chmod ；普通用户执行的大多数程序放在 `/usr/bin` 下面，例如：vim python git 等，还有通过包管理工具安装的也会放到这里；最后就是本地二进制程序，例如自己编译的二进制程序还有一些手动第三方工具的，例如 go hugo 这类工具可以放到 `/usr/local/bin` 下面。
#### 软连接
有时候，我们看到一些目录结构是这样的：
```Shell
lrwxrwxrwx   1 root root         7 Jul 24 23:09 bin -> usr/bin
dr-xr-xr-x.  6 root root      4096 Aug 23 05:02 boot
drwxr-xr-x  20 root root      3000 Apr 11 00:59 dev
drwxr-xr-x. 78 root root      4096 Aug 23 05:01 etc
drwxr-xr-x.  2 root root      4096 Apr 11 00:59 home
lrwxrwxrwx   1 root root         7 Jul 24 23:09 lib -> usr/lib
```
这里 bin 和 lib 后面都跟着一个箭头指向了 usr 下的相应目录。这里这个箭头说明当前目录下的这个文件属于软链接文件。意思相当于“如果你调用我，那么相当于你直接调用我链接的那个文件，我就是一个快捷方式”。这样也会方便的清楚文件和文件直接的关联情况。我们以 golang 的安装为例进行说明：
```shell
$ tar -C /usr/local/lib -xzf go$VERSION.$OS-$ARCH.tar.gz # 将go的安装文件解压至 /usr/local/lib
$ cd go/bin # 这里便放着go的二进制执行文件。
$ cd /usr/local/bin # 这里生成软连接文件
$ ln -s /usr/local/lib/go/bin/go go #这样便把go的源文件链接到当前目录下的go上了，
$ go version # 软连接已生效，已添加到环境变量中了
go version go1.10.3 linux/amd64
```
建议以后安装第三方软件就按照这么个步骤（如果不用包管理软件安装的话），方便管理，而且不用修改环境变量。

### Shell
上面就是 Linux 文件系统的大致情况，那么我们接下来来看 shell 的相关操作。
shell 其通常形式：username@hostname$ 或者 root@hostname# 前者便是普通用户后者表示root用户（具有最高权限）。
```shell
arirus@Arirus-Win:~$ su
Password:
root@Arirus-Win:/home/arirus# cd
root@Arirus-Win:~#
```
#### 初始化
Shell 环境在启动的时候，也是需要初始化的。初始化包括系统变量，环境变量，等等。同样我们也可以在初始化里加一些个人的相关设置，这样会更加方便我们使用。

通常 Shell 分为 `交互/非交换 Shell` 和 `登陆/非登陆 Shell`。关于这4种方式的登陆我们不揪其细节，简单的说使用不同的登陆方式，其文件初始化的顺序则不同。例如 `登陆 Shell` 会按照：

    /etc/profile [^sysconfdir]
    ~/.bash_profile
    ~/.bashrc
    ~/.bash_login
    ~/.profile
的顺序来进行初始化 。而如果使用 `非登陆交互 Shell` 则不会初始化 `.bash_profile`、`.profile`、`.bash_login`。这些文件。简单的说，`.bash_profile` 只在会话开始时被读取一次，而 `.bashrc` 则每次打开新的终端时，都会被读取。
所以通常我会把一些自定义的变量、别名等设置在 `.bashrc` 中。

#### Shell 脚本
脚本：一个比较简单的定义就是一连串的操作的集合。这里我们创建一个shell 脚本。
```shell
arirus@Arirus-Win:~$ touch ex.sh
```
同时在开头加上 `#!/bin/bash`。shell 中 `#` 用以注释行，第一行的 `#!`用于说明使用什么 shell 环境来执行脚本。这里便是说明使用 bash 来执行脚本。整个 Linux 环境下的脚本语言都是这样要求的。这样一个简单脚本便创建完成了。

两种运行脚本的方式：

    1.将脚本作为命令行参数执行
    2.将脚本作为具有执行权限的可执行文件

第一种情况下是不需要 `#!`来进行说明的，第二种作为可执行文件是需要的 `#!` 来决定使用什么程序来执行脚本。
```shell
arirus@Arirus-Win:~/shell$ sh ex.sh  #当前目录下使用 shell 执行 ex.sh 脚本
arirus@Arirus-Win:~/shell$ chmod 777 ex.sh #添加权限 使得 ex.sh 可被执行
arirus@Arirus-Win:~/shell$ ./ex.sh  #以可执行的方式执行脚本
```
通常情况下我们可以使用`;`来分隔两个不同的命令。

#### echo 命令
最基本的打印命令
后跟字符串可以不带引号，带单引号，带双引号。都会打印出相应的字符串，不过各有不同

    echo $PATH #会打印出当前的系统路径
    echo '$PATH' #单引号内的内容，不会解析内容
    echo "\"It is a test\"" #显示转义字符

注意添加参数：-n 表示显示完不进行换行，-e 表示开启转义
```shell
arirus@Arirus-Win:~/shell$ echo "1\t2\t3"
1\t2\t3
arirus@Arirus-Win:~/shell$ echo -e "1\t2\t3"
1       2       3
```

#### 变量与环境变量
变量用于储存各种数据，不需要进行类型定义直接赋值即可，Bash 中每一个变量的值都是字符串。
变量赋值：var=value 注意=的左右都没有空格。
变量输出：$var 这样便会输出变量相应的值

环境变量是未在当前进程中定义，而从父进程中继承过来的。通过使用 export 导出变量，则这个变量，在当前进程执行的任何程序（子进程）都会继承这个变量。
```shell
export GOPATH=/mnt/c/Users/Arirus/go
export GOBIN=/mnt/c/Users/Arirus/go/bin

export GOROOT=/usr/lib/go
```
这里我导出了 go 的几个必要环境变量

```shell
# 获取字符串长度 echo ${#var} :
arirus@Arirus-Win:~$ echo ${#PATH}
1398
```

#### 重定向
所谓重定向就是将打印在终端的输出，输出到另外的地方。
```shell
arirus@Arirus-Win:~/shell$ echo "Arirus is me" > readme.txt
arirus@Arirus-Win:~/shell$ cat readme.txt
Arirus is me
```
这样便把一个语句直接输出到 readme.txt 文件中了。注意：这种调用 `>` 类似与 C++ 文件的写模式，如果文件不存在则把文件创建出来，否则把整个文件清空把输出写上去。
如果使用 `>>` 则把需要添加的语句附加到文件的末尾处。类似于 `Append` 模式。以上称谓标准输出重定向。

##### 标准错误重定向
对于非法的信息使用上述方法则不会传入文件中：
```shell
arirus@Arirus-Win:~/shell$ wq > dd
wq: command not found
arirus@Arirus-Win:~/shell$ cat dd

arirus@Arirus-Win:~/shell$ wq 2> dd
arirus@Arirus-Win:~/shell$ cat dd
wq: command not found
```
因为 wq 操作系统并不知道是什么操作因此会报错，标准输出没有内容，而标准错误有内容。这时候要在 `>` 前面加个 2，便可以将标准错误输出到另一个文件中。

##### 同时重定向
或者采用 `&` 将标准输出和标准错误重定向到同一个文件中：
```shell
arirus@Arirus-Win:~/shell$ wq &> dd
```
当然了我们也可以使用 `2>&1` 使标准错误重定向到标准输出中。
注意这里使用的是 2>&1 而非 2>1 ，因为后者表示将标准错误重定向到文件1，而非标准输入，因此 & 一定要加：
```shell
arirus@Arirus-Win:~/shell$ wq > dd 2>&1
```
这样便把 `wq` 的错误信息打印到 dd 文件当中了。

##### 重定向到"垃圾桶"
如果对于某类信息不感兴趣，可以通过定向到 `/dev/null` 当中直接抛弃掉：
```shell
arirus@Arirus-Win:/dev$ ls > /dev/null
```
这样 `ls` 便不会打印出任何东西。

##### 重定向和 tee 的联合使用
tee 指令会从标准输入设备读取数据，将其内容输出到标准输出设备，同时保存成文件。
输入 `tee outputFile` 并进行输入操作，同时输入会直接打印到屏幕上，并同时写到 `outputFile` 文件中。
```shell
arirus@Arirus-Win:~/shell$ ls | tee outputFile
outputFile
arirus@Arirus-Win:~/shell$ cat outputFile
outputFile
```
`cmd 2>&1 | tee outputFile` 便可以将信息同时打印到终端和 `outputFile`
##### 小对比
`cmd > outputFile 2>&1` `cmd 2>&1 > outputFile ` `cmd 2>&1 | tee outputFile`
第一种和第三种写法是最常见的，第二种用来和第一种进行对比。
第一种：把标准输出重定向到 `outputFile` 同时标准错误重定向等同于标准输出（`outputFile`）
第二种：把标准错误重定向到标准输出（终端），同时标准输出重定向到`outputFile`，因此标准输出相当于还是指向了终端，不会在 `outputFIle` 中输出错误信息。
第三张：把标准错误重定向到标准输出（终端），通过管道和 tee 将标准输出和标准错误输出到 `outputFile` ，这样终端和文件都能看到信息。

通常会有写入文件的需求，会用：
```shell
echo alias vps='ssh -p 1 root@1.1.1.1' >> .bash_profile # 别名写入到最后一行
```
这样便将别名写入到 `.bash_profile` 中了。此时使用 tee 则是不合适的，因为设置别名是没有标准输出和标准错误的。
```shell
arirus@Arirus-Win:~$ alias vps='ssh -p 1 root@1.1.1.1' | tee .bash_profile --append # 不管用别名不会写入到文件中
```

##### 标准输入重定向
类似于标准输出重定向，标准输入重定向：
```shell
cmd < file
```
这样便把 file 中的内容传入标准输入。

#### alias 设置别名
```shell
alias ll='ls -la' //ll 设置为 ls -la
```
注意设置别名的等号左右两侧不要有空格，同时我们会把别名写入 .bashrc 或者 .bash_profile 中，这样每次启动系统别名设置就会生效。

#### 日期
读取时期：
```shell
arirus@Arirus-Win:~$ date   #获取当前时间
Mon Aug  6 14:40:13 DST 2018

arirus@Arirus-Win:~$ date +%s  #获取当前时间戳 单位秒
1533537630

arirus@Arirus-Win:~$ date '+%d %b %Y' #格式化输出当前时间
06 Aug 2018
```

获取两个时间点差：
```shell
start=$(data +%s) //$() 命令替换（command substitution），这里是替换成秒
end=$(data +%s)

dif=$((end -start)) //$(()) 这里做的是数学运算

echo "时间差为 ${dif}"  //等价于 $dif ，不过前者更清晰一些。
```

#### 数学计算
bash 中，通常使用 let、$(())、$[] 来执行基本的算术操作。
```shell
arirus@Arirus-Win:~/shell$ m=4
arirus@Arirus-Win:~/shell$ n=5
arirus@Arirus-Win:~/shell$ result=m+n
arirus@Arirus-Win:~/shell$ echo $result
m+n
arirus@Arirus-Win:~/shell$ let result=m+n   //当使用 let 时，变量名钱不需要添加 $
arirus@Arirus-Win:~/shell$ echo $result
9
```
同理 $(()) $[] 用法。

##### 小对比

    `$()` 命令替换 如上：$(data +%s)
    `${}` 变量替换 如上：${dif}
    `$(())` `$[]` 用于数学计算，括号内的变量名不需要添加 $ 来获得其真实值。和之前使用 let 一样效果
    `[]` 测试（test）命令 [ $? -ne 0 ]，必须在左括号的右侧和右括号的左侧各加一个空格，否则会报错。
    `(())` `[[]]` 分别是[ ]的针对数学比较表达式和字符串表达式的加强版，其中`(())` 内可以使用 > < 无需使用 -gt -lt 而 `[[]]` 支持 && || 逻辑操作符

#### 将命令序列的输出读入变量（|）
```shell
$ cmd1 | cmd2 | cmd3
```
cmd1的输出传递给cmd2，而cmd2的输出传递给cmd3，最终的输出（来自cmd3）将会被打印或导入某个文件。

#### read 命令
```shell
read -n number_of_chars variable_name  #读取n个字符，设置到变量里
read -s var #无回显的方式读取密码
read -p "Enter input:" var  #显示提示信息
read -t timeout var  #限定时间内完成输入
read -d delim_char var  #用特定的定界符作为输入行的结束
```

#### 比较
```shell
if condition;
then
    commands;
fi

if condition; then
    commands;
else if condition; then
    commands;
else
    commands;
fi

[ condition ] && action; # 如果 condition 为真，则执行 action ；[ condition ] || action; # 如果 condition 为假，则执行 action 。注意逻辑与逻辑或不能用在单中括号中，只能在双中括号中。
```

##### 字符串比较

    [ -n STRING ] # the length of STRING is nonzero
    [ -z STRING ] # the length of STRING is zero
    [ STRING1 = STRING2 ] # the strings are equal
    [ STRING1 != STRING2 ]  # the strings are not equal
注意在 = 前后各有一个空格。如果忘记加空格，那就不是比较关系了，而变成了赋值语句。

##### 算术比较

    -gt -lt -ge -le ：大于 小于 大于等于 小于等于
```shell
[ $var1 -ne 0 -a $var2 -gt 2 ] #使用逻辑与-a
[ $var1 -ne 0 -o var2 -gt 2 ] #逻辑或 -o
```

##### 文件系统相关

    [ -f $file_var ] ：FILE exists and is a regular file. # 判断路径存在，文件非 devices, pipes, sockets 这些。
    [ -e $file_var ] ：FILE exists.

    # 判断文件权限
    [ -x $file_var ] ：FILE exists and execute (or search) permission is granted
    [ -w $file_var ] ：FILE exists and write permission is granted
    [ -r $file_var ] ：FILE exists and read permission is granted
    # 判断文件类型（文件夹还是文件）
    [ -d $file_var ] ：FILE exists and is a directory

    [ -s $file_var ] ：FILE exists and has a size greater than zero

### 小结
以上算是对于 Shell 的简单的介绍：脚本第一行注明使用的运行环境；echo 命令的3种用法；变量与环境变量；重定向的使用；别名的设置（注意赋值和判等的区别）；日期的常用方式；数学计算；`|` 的使用；常用的三种比较。
