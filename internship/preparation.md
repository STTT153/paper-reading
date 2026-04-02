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

### 4 Congestion Control

### 5 TCP v.s. UDP
TCP（传输控制协议）和 UDP（用户数据报协议）是传输层的两大核心协议，它们的核心区别在于：TCP 追求可靠、有序、但牺牲了部分速度和灵活性；UDP 追求速度与实时性，但把可靠性保障交给了上层应用。

TCP：面向连接，用三次握手，四次挥手，流量控制，拥塞控制，SEQ，ACK等方式保证稳定连接。适合HTTP协议，数据传输，数据库连接

UDP：无连接，发数据包，追求时效性，适合实时音视频、DNS、在线游戏。

### 6 Socket编程
Server: socket()->setsocket()->bind()->listen()->mainloop
Clinet: socket()->connect()->read()->close()

### 7 IO多路复用
I/O 多路复用是一种单线程/单进程同时监控多个 I/O 事件的机制，允许程序在等待多个文件描述符（如 socket、管道）时，当其中任意一个就绪（可读、可写、异常）时，立即得到通知并处理，而无需为每个连接创建一个线程或进程。

#### select():
使用fd_set存放监控的fd, 每次调用select()时, fd_set会被复制进内核, 内核遍历所有的fd检查是否有事件发生。最多支持1024，每次调用需要重新初始化fd_set

#### poll():
使用 pollfd 数组，每个元素包含描述符和所关注的事件。调用 poll 时，将数组拷贝到内核，内核遍历数组检查事件。没有固定的描述符数量上限（受系统资源限制）。接口更简洁，支持的事件类型更丰富。

#### epoll():
- epoll_create：创建 epoll 实例，返回一个文件描述符。
- epoll_ctl：向 epoll 实例添加、修改、删除需要监控的描述符及事件。
- epoll_wait：等待事件发生，返回就绪的描述符列表。

操作系统用红黑树实现管理所有的fd，通过回调机制将触发事件的fd储存到链表中。

## c/cpp 语言特性
### 内存管理
#### 栈和堆的区别？new/malloc 有什么不同？
栈由编译器自动管理，分配速度快，空间小（一般 8MB），函数返回自动释放。堆由程序员手动管理，空间大，需显式释放。

``` c
// malloc：C风格，只分配内存，不调用构造函数
int* p = (int*)malloc(sizeof(int));
free(p);

// new：C++风格，分配内存 + 调用构造函数
MyObj* o = new MyObj();
delete o;  // 调用析构函数 + 释放内存
```

#### 内存泄漏怎么产生的？如何避免？
new 了没 delete、异常路径跳过 delete、循环引用（shared_ptr 互指）都会导致泄漏。
```c 
// 危险写法
void bad() {
    int* p = new int[100];
    if (error) return;  // 泄漏！
    delete[] p;
}

// 正确：RAII + 智能指针
void good() {
    auto p = std::make_unique<int[]>(100);
    if (error) return;  // 自动释放
}
```

#### 智能指针 unique_ptr / shared_ptr / weak_ptr 各自用途？
``` c
// unique_ptr：独占所有权，零开销，不可拷贝只能move
auto u = std::make_unique<MyObj>();
auto u2 = std::move(u);  // u 变 nullptr

// shared_ptr：引用计数，多个指针共享同一对象
auto s1 = std::make_shared<MyObj>();
auto s2 = s1;  // 引用计数 +1，= 2

// weak_ptr：弱引用，不增加计数，解决循环引用
std::weak_ptr<MyObj> w = s1;
if (auto locked = w.lock()) { /* 使用 */ }
```

#### 什么是 RAII？在存储系统开发中有何意义？
Resource Acquisition Is Initialization：资源在构造时获取，在析构时释放。利用 C++ 对象生命周期自动管理资源，保证异常安全。
``` c
class FileHandle {
    FILE* fp;
public:
    FileHandle(const char* path) : fp(fopen(path,"r")) {}
    ~FileHandle() { if(fp) fclose(fp); }
};
// 离开作用域自动关闭文件，异常也不会泄漏
```

### 面向对象
#### 虚函数和纯虚函数的区别？虚表（vtable）是什么？
虚函数: 在基类中用virtual修饰，允许在子类中override。
纯虚函数: 在基类中没有实现，子类必须实现

虚函数作用: 允许运行时多态。联想模拟器项目，用户可以在命令行启动模拟器时调整不同的Cache读写规则(write through, write back)， 这个时候就要Cache的基类，将读写相关的函数用虚函数实现。

虚表是虚函数的实现原理: 每一个虚类都会有一个vtable, 每一个instance有一个vptr指向vtable, 在调用函数的时候这个vptr用来寻址。

代价是每次调用都多一次间接寻址，并且无法内联

#### 深拷贝和浅拷贝
浅拷贝：只复制指针，两个对象共享同一块堆内存，析构时 double free。深拷贝：复制指针指向的数据本身。

#### 虚继承解决什么问题？
菱形继承问题：D 继承 B 和 C，B 和 C 都继承 A，则 D 中有两份 A 的成员，产生二义性。虚继承保证只有一份 A。
``` c
class A { int x; };
class B : virtual public A {};
class C : virtual public A {};
class D : public B, public C {};
```

### STL容器
#### unordered_map vs map，底层实现和复杂度对比？
``` c
// map：红黑树，有序，所有操作 O(log n)
std::map<string,int> m;
m["key"] = 1;  // 按key有序存储

// unordered_map：哈希表，无序，平均 O(1)，最坏 O(n)
std::unordered_map<string,int> um;
um["key"] = 1;  // 更快但无序
```

#### priority_queue 底层是什么？
底层：二叉堆（数组存储），父节点 i 的子节点是 2i+1 和 2i+2。插入和删除 O(log n)，堆顶访问 O(1)。

## 数据结构
### 红黑树
1. 每个节点是红色或黑色
2. 根节点是黑色
3. 所有叶子节点（NIL/null）是 黑色
4. 红色节点不能有红色子节点（不能连续红）
5. 从任意节点到其所有叶子节点的路径上，黑色节点数量相同（黑高一致）

最长路径 ≤ 最短路径的 2 倍

### B树 B+树
B+ 树特点：① 所有数据只在叶节点 ② 叶节点通过链表串联（支持范围查询）③ 内节点只存 key 做索引，能塞入更多 key ④ 高度矮（3-4层可覆盖百亿数据）。为什么适合磁盘：每个节点大小 = 一个磁盘页（4KB/16KB），单次 IO 读一个节点，减少磁盘访问次数。红黑树节点小、树高大，IO 次数多。


## Linux 指令
### 如何查看系统资源使用情况：
top: -m 按照内存使用情况 -p 按照CPU使用情况

ps aux: 查看所有的进程

pstree: 查看进程数

### 如何使用Linux Pipe

| 将左侧的输出作为右侧的输入

ps aux | grep nginx: 查看进程，只显示包含 nginx 的行

cat /var/log/syslog | grep "error": 查看日志，过滤错误信息

ps aux | wc -l: 统计进程数

命令 | awk '条件 { 动作 }'

>>: 将内容输出到文件中，追加

>: 将内容输出到文件中，覆盖



## 项目1 操作系统
什么是CFS(Completely Fair Schedule): 而是用“虚拟运行时间”（vruntime）来衡量一个进程已经占用的 CPU 时间，每次调度时选择 vruntime 最小的进程运行。这样，长时间未获得 CPU 的进程的 vruntime 相对较小，从而更容易被选中，实现了自然的公平。系统维护一个红黑树，做pid的查找。


## 项目2 并行运算

### MPI
MPI_init -> DO work -> MPI_finalize

MPI_size/ MPI_rank: process number/ process id.

MPI_recv/ MPI_send: 发送，接受数据。默认Blocking，当进程A发送时，如果进程B没有执行到recv，返回ACK，那么A进程会被挂起。

其余的通信方式

Broadcast, Scatter, Gather, Ruduce.

MPI_Isend/ MPI_Irecv: non-blocking, 发送，接收后不管有没有成功都继续执行。

### OpenMP
omp parallel: 多线程fork的起点

omp_get_thread_num(): 获得当前的thread id

omp parallel for: 它将 parallel（创建线程组）和 for（将循环迭代分配给线程）结合在一起，实现循环的自动并行化。

omp critical

omp task

omp single

False sharing v.s. Race condition
当多个线程并发访问共享数据，且至少有一个线程执行写操作，却没有使用正确的同步机制（如锁、原子操作）时，程序的最终结果取决于线程执行的“运气”（时序交错），这就是竞态条件。CPU 缓存以缓存行（cache line）为单位（通常 64 字节）管理数据。当两个不同线程分别修改同一缓存行中不同变量时，虽然这两个变量在逻辑上无关，但硬件为了保证缓存一致性，会导致两个 CPU 核心的缓存行反复失效，引发大量缓存同步流量。

### pthread
pthread_create(routine):创建一个线程，执行routine

pthread_join(thread_t): 阻塞直到某一线程终止

pthread_mutex_t

### CUDA

### Project1 Image Processing
### Project2 GMM

#### Base line
1. i-j-k order 单核CPU matmul

#### 使用了哪些手段优化多核矩阵乘法
1. 针对编译器的优化：为result matrix指针添加 __restrict__ 修饰或者为最内层的循环添加一个tmp变量用做计算，使编译器知道这个result matrix 只可以被当前指针使用，于是可以生成FMA汇编代码，除此之外我在封装Matrix类的时候，重写() operator的时候实现了一个用const修饰的只读版本，这样编译器就可以做更激进的内存访问上的优化。

2. 针对Cache的优化：i-j-k顺序访问的话，在访问第二个matrix的时候实际上是strided reference patern，每次访问会有一行的空洞，每一个元素都会映射到不同的cache line上。有两种解决方案：更改访问顺序i-j-k，和对第二个矩阵进行转置。我选用了前者。除此之外，当矩阵的规模非常大的时候2048， 每个元素是double precision 那么需要2048*8=16384B的内存，存一行就需要1/4 L1D cache，会造成大量的cache misses。所以我选用了tiling的方式，将一个巨大的矩阵分解为多个sub-block，那么我们可以把每个block当作element，并实现block-wise矩阵乘法，这样就大大的增加了data reuse。

3. 并行化处理：我选用了两种等级的并行化：每个线程负责计算一行block -> 线程之间embarassingly parallel。除此之外，我在block_mat_mul行数中，使用ivdep，来提示编译器使用现代simd ISA将内层loop向量化。

#### 如何设计GPU矩阵乘法
1. 一个TB负责计算一个结果块，TB中的每个线程负责计算结果中的一个元素。
2. 主要计算流程：用一个warp的视角，我在K上循环，每次从HBMloadA,B,C的一个block到shared memory中，我的内部的每一个thread就只用load三个元素。调用__synchronize()__同步，然后每个thread计算C_sublock的一个元素累加到结果中，疯狂复用shared memory中的数据。再__synchronize()__同步然后转移到下一个tile上。
3. 我的优化：使用shared memory，并且调了tile_size的大小。每个element会被复用tile_size次，当block_size变大时，HBM的内存压力就越小。但是每个TB中的线程数会变大，每个线程可用的Memory就会变少，cuda runtime会减少每个SM上驻留的warp，那么Occupency就会下降。

#### Flash Attention

## 项目3 LLVM-MCA 测试

## 项目4 Android与架构

