---
date : 2018-08-30T18:02:24+08:00
tags : ["玩儿", "笔记", "Linux"]
title : "玩儿 Linux网络工具"

---

上一篇中我们介绍了 Vim 这个 “编辑器之神”，其各个模式的相互切换、结合确实是在当前的 IDE 中属于比较罕见的，
也正是因为这样才可以在各个模式才可以高效的运行。本篇我们来说说 Liunx 最优于 Windows 的网络工具部分。本篇部分参考了网站：[Linux Tools Quick Tutorial](https://linuxtools-rst.readthedocs.io/zh_CN/latest/) 和阮一峰老师的 [SSH 原理与运用](http://www.ruanyifeng.com/blog/2011/12/ssh_remote_login.html)，在此表示感谢。

### 网络设置
既然这一篇我们主要是讲 Linux 网络部分，那么我们先说说最基本的系统网络信息配置。
#### ifconfig
ifconfig 命令用于配置及显示网络接口、子网掩码等详细信息。它通常位于/sbin/ifconfig中。（主要不要和 Windows 下的 ipconfig 混淆）
```Shell
$ ifconfig
lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Global
          ...

wifi0     Link encap:UNSPEC  HWaddr XX-XX-XX-XX-XX-XX-00-00-00-00-00-00-00-00-00-00
          inet addr:192.168.94.56  Bcast:192.168.94.255  Mask:255.255.255.0
          inet6 addr: fe80::5de3:a947:b773:66a1/64 Scope:Global
          ...
```
信息差不多都给出了，这里不多说了。

#### 域名服务
我们平时登陆一个网站，大部分情况下都是使用网址 `t66y.com` ,很少会直接输入 `104.25.31.112` 这样的 ip 地址，那么这之间肯定是要有一个转换或者对应关系。这种利用符号名对IP地址进行抽象的技术就被称为域名服务（DNS）。Linux 中的 DNS 服务器都是配置在 `/etc/resolv.conf` 当中。我们既可以添加新的 DNS 服务器也可以修改现有的 DNS 配置。
```shell
$ cat resolv.conf
nameserver 223.5.5.5 #这里我便是把自动配置的 DNS 服务器，换成了阿里的 DNS 服务器
nameserver 223.6.6.6
# nameserver 192.168.3.248
# nameserver 114.114.114.114
```

除了使用 DNS 服务，我们还有别的办法来进行域名解析（例如当 DNS 污染时，Github 是登不上去），就是使用修改 `hosts` 的方法。`host` 中文名是主机名查询静态表，负责IP地址与域名快速解析的文件。在没有域名服务器的情况下，系统上的所有网络程序都通过查询该文件来解析对应于某个主机名的IP地址，否则就需要使用DNS服务程序来解决。通常可以 将常用的域名和IP地址映射加入到hosts文件中，实现快速方便的访问。
```Shell
$ echo "192.30.253.113  github.com" >> /etc/hosts #这样将需要静态解析的ip地址传入到 hosts 文件中，
```
查看 `/etc/host.conf` 可以看到其有 `order hosts,bind` 意思就是先查找 hosts 文件的对应关系，再进行 DNS 解析。

#### ping
绝大多数操作系统上都包含了该命令。 ping 是一个验证网络上两台主机连通性的诊断工具，能够找出网络上的活动主机。
使用方法 `$ ping [Address]` 。
```shell
$ ping github.com
PING github.com (192.30.253.113) 56(84) bytes of data.
64 bytes from github.com (192.30.253.113): icmp_seq=1 ttl=47 time=250 ms
64 bytes from github.com (192.30.253.113): icmp_seq=2 ttl=47 time=260 ms
64 bytes from github.com (192.30.253.113): icmp_seq=3 ttl=47 time=264 ms
^C
--- github.com ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 3004ms
```
这就是说明 Github 我们可以 Ping 的通，并且没有丢包，所以，我们可以正常链接。当然我们也可以使用 `-c N` 表示发送多少数据包进行请求。

### wget
wget 是一个用于文件下载的命令行工具，支持 HTTP HTTPS FTP 协议，也支持 HTTP 代理。其命令格式为：`wget [参数] [URL地址]`
```shell
$ wget https://www.baidu.com
# 这样便把百度的首页下载到当前的位置，一个html文件
```
这就是一个最基本的 wget 的用法，下载一个文件。
我们再给出几个wget的常用参数用法：
```shell
#-O 下载并以不同的文件名保存。由于某些链接属于动态下载链接，无法定位到真实文件名字，-O 参数可以重新命名来解决这个问题。
$wget -O wordpress.zip http://www.minjieren.com/download.aspx?id=1080

-–limit-rate 限速下载。wget 默认下载会占满带宽，限速下载对于大文件下载比较有用
$wget --limit-rate=300k http://www.minjieren.com/wordpress-3.1-zh_CN.zip

-c断点续传。wget -c重新启动下载中断的文件，默认情况下则是会重新下载一个新的文件。
$wget -c http://www.minjieren.com/wordpress-3.1-zh_CN.zip

-r 递归下载 -l 递归层数，默认是5 -A .pdf，.jpg 需要下载的内容 -R .html 需要规避的内容
$wget -r -A pdf url
```
从上面最后一个，我们也能看出来使用 wget 来进行爬虫也是可以的选择。其实我也试过了，这种太基础了，大多数商业网站都是可以反爬虫的，静态网站倒是可以，那也没啥意思，还得用专业工具搞爬虫。

### cURL
说完 wget 怎么着也要提下 cURL 啊。二者是非常的像，都是用在网络请求上面，这里先引用另一位同学对于二者区别的看法：

    wget是个专职的下载利器，简单，专一，极致；而curl可以下载，但是长项不在于下载，而在于模拟提交web数据，POST/GET请求，调试网页，等等。
wiki 上面也有对于 wget 缺点的描述这里不再搬运了。其命令的主要格式为：`curl [参数] [URL地址]` 没错，和 wget 是一样的。

由于 curl 默认是进行 GET 请求，因此我们可以之间使用 curl 来下载文件：
```Shell
$ curl -O [URL地址] #将下载的文件重命名为，远程文件的文件名，如果使用 -o 需要自己定义保存的名称
$ curl -o fileName [URL地址] #手动指定下载文件的名字
$ curl -C - [URL地址] #自动判断位置进行断点续传
```
大部分使用 `wget` 的下载使用 `curl` 也是可以实现的，这里我们主要说说其进行 API 请求的功能。
```Shell
$ curl -X METHOD [URL地址] #使用不同的方法可以进行不同的网络 API 请求
$ curl -X POST --user-agent  http://httpbin.org/post --data 'parameter1=first&parameter2=second'
{
  "args": {},
  "data": "",
  "files": {},
  "form": {
    "parameter1": "first",
    "parameter2": "second"
  },
  "headers": {
    "Accept": "*/*",
    "Connection": "close",
    "Content-Length": "17",
    "Content-Type": "application/x-www-form-urlencoded",
    "Host": "httpbin.org",
    "User-Agent": "CHrome"
  },
  "json": null,
  "origin": "XXX.XXX.XXX.XXX",
  "url": "http://httpbin.org/post"
}
```
本例中，使用了 http://httpbin.org 网址，这是一个开源的 HTTP 请求测试网址，会把你请求的数据原封不动的返回给你。
这里我们使用了 post 方法，将参数进行 urlencode 请求，以键值对的的形式将两对参数传过去。
得到的返回数据如上。

同理，我们也可以进行 request body 的传递：
```Shell
$  curl -X POST  http://httpbin.org/post --data '{"dsds":33,"wewew":["Arirus","whd","me"]}' \
> -H "Content-Type:application/json"
{
  "args": {},
   "data": "{\"dsds\":33,\"wewew\":[\"Arirus\",\"whd\",\"me\"]}",
   "files": {},
   "form": {},
   "headers": {
     "Accept": "*/*",
     "Connection": "close",
     "Content-Length": "41",
     "Content-Type": "application/json",
     "Host": "httpbin.org",
     "User-Agent": "curl/7.47.0"
   },
   "json": {
     "dsds": 33,
     "wewew": [
       "Arirus",
       "whd",
       "me"
     ]
   },
   "origin": "XXX.XXX.XXX.XXX",
   "url": "http://httpbin.org/post"
}
```
因为我们是传送的 json 数据，因此要加上 `-H` 来声明传输的数据类型，同时需要将传输的数据放到一个 json 格式中。由于传输的是非表单数据，因此 `form` 字段返回一个空的数据结构，同时 `data` 字段会返回所有请求数据。除了 `json` 也可以传输 `text/plain` 等格式，可根据需要调整。

当然 curl 也是支持上传相应的文件的。
```shell
$ curl X POST -F "fileName=@/usr/local/bin/README.md" -F "parm1=43" -F "parm2=ARIRUS"  httpbin.org/post
{
  "args": {},
  "data": "",
  "files": {
    "fileName": "Hello cURL" #README.md 的内容
  },
  "form": {
    "parm1": "43",
    "parm2": "ARIRUS"
  },
  "headers": {
    "Accept": "*/*",
    "Connection": "close",
    "Content-Length": "7072",
    "Content-Type": "multipart/form-data; boundary=------------------------3b427c901e208bfc",
    "Expect": "100-continue",
    "Host": "httpbin.org",
    "User-Agent": "curl/7.47.0"
  },
  "json": null,
  "origin": "124.160.26.190",
  "url": "http://httpbin.org/post"
}
```
这里我们上传了一个文件，上传名为 `fileName`其在系统中对应的位置在 `/usr/local/bin/README.md` 注意要用 `@` 作为开头。
同时还上传了两对参数：`parm1` 与 `parm2`，curl 同时会将 `Content-Type` 设置为 `multipart`。

当然了，cURL 还有许多参数可供配置，我们就不一一来讲，看看其 manual 会有更多发现。

### SSH
SSH 本质上是一种网络协议，就像 HTTP TCP 这种一样，这次我们不是深究协议本身而是，从他的一个实现：OpenSSH 来解释介绍他的基本用法。

#### 基本操作
SSH 主要用于远程登录。假定你要以用户名user，登录远程主机host，只要一条简单命令就可以了。

    $ ssh user@host
SSH的默认端口是22，也就是说，你的登录请求会送进远程主机的22端口。使用p参数，可以修改这个端口。

    $ ssh -p 2222 user@host

SSH 登陆时，显示用户端与服务器链接，服务器返回公钥给用户端，用户端使用其加密自己登陆账号和密码，服务器收到数据使用私钥来进行解密，如果密码正确则同意用户登陆。
SSH 可以避免中间人攻击（man-in-the-middle attack），所谓中间人攻击就是攻击者冒充远程服务器，将伪造公钥发给登陆用户。
所以 SSH 第一次登陆时，用户会记录下来远程服务器的公钥指纹信息，之后每次登陆都会做校验，如果核对不上的话则任务遇上了中间人攻击。
在第一次登陆时，会进行指纹信息登记，将其记录到 known_hosts 文件中，之后每次登陆进行校验就好了。

有时我们只想在远程服务器上执行一个命令，使用 SSH 可以保证数据传输过程的安全性，我们会进行如下操作：
```shell
$ ssh user@host 'COMMANDS'  # 注意命令的内容需要使用 “” 或者‘’引用起来
$ ssh root@XXX.XXX.XXX.XXX "whereis go" # 打印出 go 的位置
go: /usr/bin/go
```
注意，传输传递的过程中并不涉及 shell 的登陆，因此自定义的变量并不会生效。

#### 免密登陆
SSH 提供了免密登陆机制，可以配合脚本实现自动登陆。
设置 SSH 自动化认证需要两步：

    (1) 创建 SSH 密钥，这用于登录远程主机；
    (2) 将生成的公钥传给远程主机，并将其加入文件 ~/.ssh/authorized_keys中。
创建密钥对的话使用 `ssh-keygen` 来生成。生成后，密钥对位于 ~/.ssh 文件夹下面，其中 `id_rsa.pub` 是公钥放置的地方，是可以分享给别的用户的，而 `id_rsa` 是私钥放置的地方，要保存好。  

生成密码之后，要将公钥添加到服务器的 `~/.ssh/authorized_keys` 里，我们可以使用

    $ ssh-copy-id root@hostname  
将`.pub` 文件的公钥添加到 `authorized_keys` 内。这样以后再登陆远程服务器就不需要每次都输入登陆密码了。
我们也可以

    $ ssh root@hostname 'mkdir -p -m 700 .ssh && cat >> ./ssh/authorized_keys && chmod 600 ./ssh/authorized_keys' < ~/.ssh/id_rsa.pub #这里就是使用了 SSH 来传递命令，使远程服务器来执行。
    # 创建 .ssh 文件夹，并且将其权限移除 group 和 others 的相关权限
    # 将本地 `.pub` 内的内容写到 `authorized_keys` 上
    # 最后将 `authorized_keys` 权限改为 600
    # 权限相关的内容是参考了 [I copied my public key to authorized_keys but public-key authentication still doesn't work.](https://web.archive.org/web/20140327182105/http://www.openssh.org/faq.html#3.14)

#### 端口转发

##### 本地端口转发
所谓本地端口转发，就是将**发送到本地端口的请求，转发到目标端口**。这样，就可以通过访问本地端口，来访问目标端口的服务。
```shell
# -L <local port>:<target host>:<target port> <SSH hostname>
$ ssh -L 1080:localhost:1926 root@XXX.XXX.XXX.XXX
# 这样便把本地 1080 端口和远程的 1926 端口相连接了起来，可以接受或者发送数据
```
这里我们已 `Golang` 的中文教程为例，简单说明一下。[Go 语言官方教程中文版](https://github.com/Go-zh/tour)，源项目中，服务是部属在本地：
```shell
var (
	httpListen  = flag.String("http", "127.0.0.1:3999", "host:port to listen on")
	openBrowser = flag.Bool("openbrowser", true, "open browser automatically")
) # 定义再 tour/local.go 文件内
```
我们简单说一下 `127.0.0.1` 与 `0.0.0.0` 两个地址的区别。服务器上使用 `0.0.0.0` 指向本机的所有 IP 地址，假如服务部署在了此上面，那么无论调用内外 IP 还是公网 IP 都能访问到该服务。
但是如果使用 `127.0.0.1` ，其属于本地回环地址，所谓回环地址在其他计算机上不能进行访问，只能本机进行访问。所以如果服务部署到 `127.0.0.1` 上的话外部是不能访问的，只能本机调用。

如果我在 VPS 上，将其部署到了 `127.0.0.1` ，我是用外网无法进行访问的。只允许 VPS 主机上通过 3999 端口进行访问。如果我们要临时通过外网访问就可以使用如下操作：
```shell
$ ssh -L 8080:localhost:3999 root@XXX.XXX.XXX.XXX
# 这样便通过本地端口转发将远程的 3999 端口，绑定到了本地 8080 端口，这时候就可以在本地通过 localhost：8080 来访问中文教程服务了。
```

##### 远程端口转发
类似于本地端口转发，远程端口转发会将**将发送到远程端口的请求，转发到目标端口**。
```shell
# -R <remote port>:<target host>:<target port> <SSH hostname>
```
个人理解的两者的不同：

    1.本地端口转发，将本地服务器 <port> 的所有数据请求转到 <target host>:<target port>
    这个默认的 <target> 就是远程服务器
    2.远程端口转发，将远程服务器 <port> 的所有数据请求转到 <target host>:<target port>
    这个默认的 <target> 就是本地服务器
第一种像是正常的内网机请求外网机的服务，第二种则是外网机请求内网机的服务，由于外网机无法直接请求和内网机链接，所以在外网机上是无法直接使用本地端口转发链接内网机的。这时候在内网机上使用远程端口转发是最合适不过的，这样远端调用时直接会访问本地端口。
```Shell
$ ssh -R 8100:localhost:8000 root@XXX.XXX.XXX.XXX #这样便把远端的8100端口和本地的8000端口联系了起来，在远端可以直接访问本地数据。
```

##### scp
scp是secure copy的简写，用于在Linux下进行远程拷贝文件的命令，通过其命令我们也大致能猜出来，scp 是使用了 ssh 来实现了 copy 命令。
```shell
scp [参数] [原路径] [目标路径]
```
参数的话，没什么好说的看 manual 好了，有一点需要强调下，如果ssh是非默认端口22，则要使用 `-P` 来进行端口设定
```shell
$ scp -r /opt/soft/test root@10.6.159.147:/opt/soft/scptest # 上传本地目录到远程目录。
```

### lsof 与 netstat
netstat 网络服务分析的命令，可以使用其列出服务与端口号：
```Shell
$ netstat -anpt
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 127.0.0.1:25            0.0.0.0:*               LISTEN      1656/master
tcp        0      0 0.0.0.0:1926            0.0.0.0:*               LISTEN      26380/python
tcp        0      0 0.0.0.0:22            0.0.0.0:*               LISTEN      11457/sshd
tcp        0      0 XXX.XXX.XXX.XXX:22       YYY.YYY.YYY.YYY:12465     ESTABLISHED 12988/sshd: root@pt
tcp6       0      0 ::1:25                  :::*                    LISTEN      1656/master
tcp6       0      0 :::3999                 :::*                    LISTEN      12752/./gotour
tcp6       0      0 :::22                 :::*                    LISTEN      11457/sshd
tcp6       0      0 :::80                   :::*                    LISTEN      5404/hugo
```
当前 VPS 上所有的TCP端口使用情况。80 端口对应了hugo服务 22 端口对应了ssh服务 3999 端口对应了Golang的教程服务等等。
参数方面 -n 将端口的“昵称”直接转换为相应的数字，-p 显示相应的PID，-a 显示所有socket链接情况包括`LISTEN`，- t 则表示进行显示 tcp 链接。

类似的 lsof 则适用性更广，因为其本质是查询当前系统文件的工具。因为Linux下，任何任何事物都是以文件的形式存在的，包括 tcp 和 udp 的链接。
因此我们可以使用如下方式来显当前的 tcp 链接：
```shell
$ lsof -i tcp
COMMAND    PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
master    1656 root   13u  IPv4  15866      0t0  TCP localhost:smtp (LISTEN)
master    1656 root   14u  IPv6  15867      0t0  TCP localhost:smtp (LISTEN)
hugo      5404 root    8u  IPv6 571530      0t0  TCP *:http (LISTEN)
sshd     11457 root    3u  IPv4  27513      0t0  TCP *:stun-p2 (LISTEN)
sshd     11457 root    4u  IPv6  27515      0t0  TCP *:stun-p2 (LISTEN)
gotour   12752 root    3u  IPv6 659439      0t0  TCP *:nvcnet (LISTEN)
sshd     12988 root    3u  IPv4 662969      0t0  TCP ArirusVps:stun-p2->115.238.89.35:12465 (ESTABLISHED)
ssserver 26380 root    4u  IPv4  85781      0t0  TCP *:egs (LISTEN)
```
这样我们便显示出所有的 tcp 链接。这里如果想向 netstat 一样显示具体的ip地址和端口号，需要配上 `-nP` 参数。

`-i` 参数可以列出符合条件的进程：
```shell
-i i   select by IPv[46] address: [46][proto][@host|addr][:svc_list|port_list]
# 就是说其支持 4，6 @ip tcp|udp :port
$ lsof  -i udp
COMMAND    PID   USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
dhclient   846   root    6u  IPv4  14032      0t0  UDP *:bootpc
dhclient   846   root   20u  IPv4  14008      0t0  UDP *:57717
dhclient   846   root   21u  IPv6  14009      0t0  UDP *:58851
chronyd  11684 chrony    1u  IPv4  28681      0t0  UDP localhost:323
chronyd  11684 chrony    2u  IPv6  28682      0t0  UDP localhost:323
ssserver 26380   root    5u  IPv4  85782      0t0  UDP *:egs
ssserver 26380   root    8u  IPv4  85783      0t0  UDP *:52667

$ lsof  -i :1991,1926
COMMAND    PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
sshd     11457 root    3u  IPv4  27513      0t0  TCP *:stun-p2 (LISTEN)
sshd     11457 root    4u  IPv6  27515      0t0  TCP *:stun-p2 (LISTEN)
sshd     12988 root    3u  IPv4 662969      0t0  TCP ArirusVps:stun-p2->XXX.XXX.XXX.XXX:12465 (ESTABLISHED)
ssserver 26380 root    4u  IPv4  85781      0t0  TCP *:egs (LISTEN)
ssserver 26380 root    5u  IPv4  85782      0t0  UDP *:egs
```

### 小结
本篇中，我们从网络设置查看入手，接触了 wget 和 curl 两个常用的网络工具。再到 ssh 的相关概念和使用方法，还介绍了基于 ssh 的 scp 安全拷贝。最后介绍了 lsof 和 netstat 两个查看当前的网络链接。大致上，如果正常使用 Linux 系统，这4篇文章应该可以涵盖大部分的操作需要知道知识。这个系列就先告一段落了，有缘再见。
