# 网络基础 TCP UDP

## TCP
### 头部格式

![](data:image/png;base64,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)

头部大小最小为：32×5 位  20 字节

序列号（32位）：在建立连接时由计算机生成的随机数作为其初始值，通过 SYN 包传给接收端主机，每发送一次数据，就「累加」一次该「数据字节数」的大小。用来解决网络包乱序问题。

确认应答号（32位）：指下一次「期望」收到的数据的序列号，发送端收到这个确认应答以后可以认为在这个序号以前的数据都已经被正常接收。用来解决不丢包的问题。

控制位：

- ACK：该位为 1 时，「确认应答」的字段变为有效，TCP 规定除了最初建立连接时的 SYN 包之外该位必须设置为 1 。

- RST：该位为 1 时，表示 TCP 连接中出现异常必须强制断开连接。

- SYC：该位为 1 时，表示希望建立连，并在其「序列号」的字段进行序列号初始值的设定。

- FIN：该位为 1 时，表示今后不会再有数据发送，希望断开连接。当通信结束希望断开连接时，通信双方的主机之间就可以相互交换 FIN 位置为 1 的 TCP 段。


### 基本特性
TCP 是面向连接的、可靠的、基于字节流的传输层通信协议。

- 面向连接：一定是「一对一」才能连接，不能像 UDP 协议 可以一个主机同时向多个主机发送消息，也就是一对多是无法做到的；

- 可靠的：无论的网络链路中出现了怎样的链路变化，TCP 都可以保证一个报文一定能够到达接收端；

- 字节流：消息是「没有边界」的，所以无论我们消息有多大都可以进行传输。并且消息是「有序的」，当「前一个」消息没有收到的时候，即使它先收到了后面的字节已经收到，那么也不能扔给应用层去处理，同时对「重复」的报文会自动丢弃。

TCP 四元组可以唯一的确定一个连接：
**源地址**和**目的地址**的字段（32位）是在 IP 头部中，作用是通过 IP 协议发送报文给对方主机。
**源端口**和**目的端口**的字段（16位）是在 TCP 头部中，作用是告诉 TCP 协议应该把报文发给哪个进程。

### TCP 三次握手

#### 具体流程
- 一开始，客户端和服务端都处于 CLOSED 状态。先是服务端主动监听某个端口，处于 LISTEN 状态
- 客户端会随机初始化序号（client_isn），将此序号置于 TCP 首部的「序号」字段中，同时把 SYN 标志位置为 1 ，表示 SYN 报文。接着把第一个 SYN 报文发送给服务端，表示向服务端发起连接，该报文不包含应用层数据，之后客户端处于 SYN-SENT 状态。
- 服务端收到客户端的 SYN 报文后，首先服务端也随机初始化自己的序号（server_isn），将此序号填入 TCP 首部的「序号」字段中，其次把 TCP 首部的「确认应答号」字段填入 client_isn + 1, 接着把 SYN 和 ACK 标志位置为 1。最后把该报文发给客户端，该报文也不包含应用层数据，之后服务端处于 SYN-RCVD 状态。
- 客户端收到服务端报文后，还要向服务端回应最后一个应答报文，首先该应答报文 TCP 首部 ACK 标志位置为 1 ，其次「确认应答号」字段填入 server_isn + 1 ，最后把报文发送给服务端，**这次报文可以携带客户到服务器的数据**，之后客户端处于 ESTABLISHED 状态。
- 服务器收到客户端的应答报文后，也进入 ESTABLISHED 状态。

#### 原因
- 三次握手才可以阻止历史重复连接的初始化（主要原因）

- 三次握手才可以同步双方的初始序列号

- 三次握手才可以避免资源浪费

核心在于两次握手只能单方面同步，客户端的状态和知情状态（客户端知道服务端已经准备好了，也知道服务端已经知道客户端准备好了），但是不能同步服务端的知情状态（服务端知道客户端已经准备好了，不知道客户端知不知道服务端知道客户端准备好了）。这样的话客户端可能不会发消息，那如果这样，服务器的初始化本身就是多余的操作。如果是历史链接的话，客户端无法通知服务端关闭初始化。


客户端和服务端生成的 ISN 是同一个方法，但是由于 网络延迟，超时重传等原因

### MTU（Maximum Transfer Unit） 和 MSS（Maximum Segment Size）
IP 传输单元是 数据报（Datagram），超过一个IP长度则会分片。
TCP  传输单元是 报文（message），超过一个TCP长度则会分段（segment）

最大传输单元MTU，是链路层对数据帧的长度都有一个限制。如果IP层有一个数据报要传，而且数据的长度比链路层的MTU还大，那么IP层就需要进行分片（fragmentation），把数据报分成若干片，这样每一片都小于MTU。分片传输的IP数据报不一定按序到达，但IP首部中的信息能让这些数据报片按序组装。IP数据报的分片与重组是在网络层进完成的。以太网的MTU为1500字节

MSS其标识TCP能够承载的最大的应用数据段长度。在以太网环境下，MSS值一般就是1500-20-20=1460字节（MTU-IP HEADER-TCP HEADER）。TCP报文段的分段与重组是在运输层完成的。

TCP分段的原因是MSS，IP分片的原因是MTU，由于一直有MSS<=MTU，很明显，分段后的每一段TCP报文段再加上IP首部后的长度不可能超过MTU，因此也就不需要在网络层进行IP分片了。因此TCP报文段很少会发生IP分片的情况。反之UDP报文段会有很多IP分片的情况。

对IP分片的数据报来说，即使只丢失一片数据也要重新传整个数据报（既然有重传，说明运输层使用的是具有重传功能的协议，如TCP协议）。这是因为IP层本身没有超时重传机制------由更高层（比如TCP）来负责超时和重传。当来自TCP报文段的某一段（在IP数据报的某一片中）丢失后，TCP在超时后会重发整个TCP报文段，该报文段对应于一份IP数据报（可能有多个IP分片），没有办法只重传数据报中的一个数据分片。

### TCP 四次挥手

#### 具体流程
- 客户端打算关闭连接，此时会发送一个 TCP 首部 FIN 标志位被置为 1 的报文，也即 FIN 报文，之后客户端进入 FIN_WAIT_1 状态。

- 服务端收到该报文后，就向客户端发送 ACK 应答报文，接着服务端进入 CLOSED_WAIT 状态。

- 客户端收到服务端的 ACK 应答报文后，之后进入 FIN_WAIT_2 状态。

- 等待服务端处理完数据后，也向客户端发送 FIN 报文，之后服务端进入 LAST_ACK 状态。

- 客户端收到服务端的 FIN 报文后，回一个 ACK 应答报文，之后进入 TIME_WAIT 状态

- 服务器收到了 ACK 应答报文后，就进入了 CLOSE 状态，至此服务端已经完成连接的关闭。

- 客户端在经过 2MSL 一段时间后，自动进入 CLOSE 状态，至此客户端也完成连接的关闭。

2MSL是因为：网络中可能存在来自发送方的数据包，当这些发送方的数据包被接收方处理后又会向对方发送响应，所以一来一回需要等待 2 倍的时间。

#### 原因

- 关闭连接时，客户端向服务端发送 FIN 时，仅仅表示客户端不再发送数据了但是还能接收数据。

- 服务器收到客户端的 FIN 报文时，先回一个 ACK 应答报文，而服务端可能还有数据需要处理和发送，等服务端不再发送数据时，才发送 FIN 报文给客户端来表示同意现在关闭连接。


#### TIME-WAIT
仅主动关闭连接的，才有 TIME_WAIT 状态。

这个状态出现的原因：
- 防止具有相同「四元组」的「旧」数据包被收到，避免新的TCP链接时，旧的消息抵达并被接受；（时间长到可以丢弃链路上所有的报文）

- 保证「被动关闭连接」的一方能被正确的关闭，即保证最后的 ACK 能让被动关闭方接收，从而帮助其正常关闭；（就是上面原因的第二点）


### 重传机制

#### 超时重传

- 数据包丢失
- 确认应答丢失

超时重传时间 RTO 的值应该略大于报文往返  RTT 的值。较大会使得网络空隙时间较大，降低网络的传输效率，较小会导致不必要的重传，导致网络负荷增加。

当数据等待时间超过 RTO 时，就触发超时重传。

#### 快速重传

快速重传的工作方式是当收到三个相同的 ACK 报文时，会在定时器过期之前，重传丢失的报文段。这种适用于数据包丢失，而非确认应答丢失


### 滑动窗口

开辟一个缓存区，里面有要传输的数据序列，有这个缓存区，无需等待一个个返回确认，就可以直接发送数据，提高传输效率。这个缓存区，被称为滑动窗口，窗口大小就是指无需等待确认应答，而可以继续发送数据的最大值。

整个滑动窗口将待发送数据分为4端：
- 已发送并收到 ACK确认的数据
- 已发送但未收到 ACK确认的数据
- 未发送但总大小在接收方处理范围内（接收方还有空间）
- 未发送但总大小超过接收方处理范围（接收方没有空间）

其中 2，3 是滑动窗口的总大小，3是可用滑动窗口的大小，为0时就说明没有可用空间进行传输数据。

当滑动窗口向右移动距离之后，就又有空余位置发送数据了。

#### 流量控制
当接受端的处理速度跟不上的时候，返回给发送端的window为一个较小的值，这样发送端就会调整发送数据的大小，这样就改变传输速率，进行了流量控制。

### TCP 长连接
TCP 长连接是一种保持 TCP 连接的机制。当一个 TCP 连接建立之后，启用 TCP Keep Alive 的一端便会启动一个计时器，当这个计时器到达 0 之后，一个 TCP 探测包便会被发出。这个 TCP 探测包是一个纯 ACK 包，但是其 Seq 与上一个包是重复的。

### 拥塞控制

前面的流量控制是避免「发送方」的数据填满「接收方」的缓存，而阻塞控制则是避免「发送方」的数据填满网络。
因此定义了一个**阻塞窗口**，类似滑动窗口来进行处理。


拥塞窗口 cwnd 变化的规则：

- 只要网络中没有出现拥塞，cwnd 就会增大；
- 但网络中出现了拥塞，cwnd 就减少；

其实只要「发送方」没有在规定时间内接收到 ACK 应答报文，也就是发生了**超时重传**，就会认为网络出现了用拥塞。

#### 慢启动

TCP 在刚建立连接完成后，首先是有个慢启动的过程，这个慢启动的意思就是一点一点的提高发送数据包的数量，避免一下向网络中传输大量数据（温水煮青蛙）。其方式在于：当发送方每收到一个 ACK，就拥塞窗口 cwnd 的大小就会加 1。

初始 cwnd 的大小为1，那么增长的过程应该是 1->2->4->8->16 呈指数倍的增长。

当到达一个阈值时，使用**拥塞避免算法**

#### 拥塞避免算法

那么进入拥塞避免算法后，它的规则是：每当收到一个 ACK 时，cwnd 增加 1/cwnd。那么其增长过程变为 n->n+1->n+2->n+3->n+4 ，由原先的指数增长降级为线性增长。

然后一直增长到发送滑动窗口后就不再增长了。



