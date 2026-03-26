# 面试准备

## 网络编程
> TCP/UDP网络协议及相关编程

### 1 三次握手
client端ISN（Initial Sequence Number） = x, server端ISN = y

#### 具体流程
1. client发送SYN(x)
2. server发送SYN(y), ACK(x+1)
3. client发送ACK(y+1)
4. SYN重传

#### 为什么不是2次/4次
如果只进行两次握手的话，即在client端收到ACK后并没有继续进行后续确认。那么server端将在第一次发出ACK包之后，就已经进入了Established状态。如果这个时候ACK包并没有被Client接收的话，对方会重新传ACK，就会造成资源的浪费。

但第三次握手的 ACK 已经足以让服务器确认客户端收到了 SYN-ACK，并且双方已经同步了序列号。再增加一次往返只会多一次报文，不增加任何可靠性或安全性，反而增加延迟。

#### SYN 洪水攻击
客户端发送 SYN 报文，服务端为该连接分配一个 传输控制块（TCB），包含连接状态、缓冲区、计时器等，并返回 SYN+ACK。攻击者伪造大量不同源 IP 的 SYN 报文发送给服务端。耗尽服务端的CPU和内存资源。

解决方案：在第一次收到SYN后不为其分配资源，而是通过hash function获得一个哈希值做为cookie返回给客户端。在接受第三次ACK时，将cookie进行比对

### 2 TCP四次挥手
1. 主动方发送FIN -> 进入FIN_WAIT_1
2. 被动方发送ACK -> 进入CLOSE_WAIT
3. 被动方发送FIN -> 进入LAST_ACK
4. 主动方收到ACK -> 进入FIN_WAIT_2
5. 主动发发送ACK -> 进入TIME_WAIT
6. FIN重传

#### 为什么需要有TIME_WAIT
存在发送的ACK没有被被动方接受情况，这时被动发会重发FIN，需要接受重传来的FIN

#### 过长的TIME_WAIT 带来的影响
内核内存占用每个 TIME_WAIT 连接仍在内核中保留少量内存（如连接控制块），过多的 TIME_WAIT 会消耗系统内存。影响高并发性能在高并发的服务端如果作为主动关闭方（如一些代理服务器），大量的 TIME_WAIT 会导致文件描述符和端口紧张，降低系统吞吐量。

### 3 滑动窗口
Allow more TCP frames on flight.

