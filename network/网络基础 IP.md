# 网络基础 IP

网络层的主要作用是：实现主机与主机之间的通信，也叫点对点（end to end）通信。

IP（网络层） 和 MAC （数据链路层）之间的区别和关系：MAC 的作用则是实现「直连」的两个设备之间通信，而 IP 则负责在「没有直连」的两个网络之间进行通信传输。在网络中数据包传输中也是如此，源IP地址和目标IP地址在传输过程中是不会变化的，只有源 MAC 地址和目标 MAC 一直在变化。

## IP 地址

IP 地址（IPv4 地址）由 32 位正整数来表示，IP 地址在计算机是以二进制的方式处理的，采用点分十进制的标记方式。

因此 IP 最大地址为：`255.255.255.255`，总量在 2的32 次幂约为 43亿。

### 分类
![](https://mmbiz.qpic.cn/mmbiz_png/J0g14CUwaZdpI5tWwB8wMfZQ8ichJW1yu3u9AgedCb0tFU7W1fZlqG0ImZVnIoAan7z2vX71WYeKZz1waagqoXw/640?wx_fmt=png&tp=webp)

