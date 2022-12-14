- [从网卡如何接收数据说起](#从网卡如何接收数据说起)
- [CPU 如何知道接受了数据？](#cpu-如何知道接受了数据)
- [进程阻塞为什么不占用 CPU 资源？](#进程阻塞为什么不占用-cpu-资源)
  - [工作队列](#工作队列)
  - [等待队列](#等待队列)
  - [唤醒进程](#唤醒进程)
- [内核接收网络数据全过程](#内核接收网络数据全过程)
- [同时监视多个 socket 的方法](#同时监视多个-socket-的方法)
  - [select 的监听流程](#select-的监听流程)
  - [select 的唤醒流程](#select-的唤醒流程)
  - [select 的缺点](#select-的缺点)
- [epoll 的设计思路](#epoll-的设计思路)
- [epoll 的原理和流程](#epoll-的原理和流程)
- [epoll 的实现细节](#epoll-的实现细节)
- [结论](#结论)
- [参考资料](#参考资料)

### 从网卡如何接收数据说起

下图是一个典型的计算机结构图，计算机由 CPU、存储器(内存)、网络接口等部件组成。要想真正了解 epoll, 第一步就要从硬件的角度看计算机如何接收网络数据。

<div align="center"><img src=./picture/what-is-it-004-001.jpg /></div>

上图展示了网卡接收数据的过程：
- 阶段1: 网卡收到网线传来的数据。
- 阶段2: 硬件电路(网卡)的处理和传输。
- 阶段3: 将数据写入到内存的某个地址上。

> 上面的整个过程涉及到 **DMA传输**、**IO通路选择** 等硬件有关的知识，但我们只需要知道：**网卡会把接收到的数据写入内存**。


### CPU 如何知道接受了数据？

了解 epoll 的第二步，要从 CPU 的角度来看数据接收。要理解这个问题，就要先了解一个概念：**中断**。

计算机执行程序时，就会有优先级的需求。比如，当计算机收到中断信号时(**电容可以保存少许电量，供 CPU 运行很短的一小段时间**)，它应立即去保存数据，保存数据的程序具有较高的优先级。

一般而言，由硬件产生的中断信号需要 CPU 立刻做出回应(**不然数据就可能丢失**)，所以它的优先级很高。CPU 理应中断掉正在执行的程序，去做出响应；当 CPU 完成对硬件的响应后，再重新执行用户程序。中断的过程如下图 2 ，和函数调用差不多，只不过函数调用是事先定好位置，而中断的位置由**中断信号**决定。

<div align="center"><img src=./picture/what-is-it-004-002.svg /></div>

以键盘⌨️为例，当用户按下键盘某个按键时，键盘会给 CPU 的中断引脚发出一个高电平。CPU 能捕获这个信号，然后执行键盘中断程序。下图 3 展示了各种硬件通过中断与 CPU 进行交互。

<div align="center"><img src=./picture/what-is-it-004-003.svg /></div>

到现在可以回答本节提出的问题了：当网卡把数据写入到内存后，**网卡向 CPU 发出一个中断信号，操作系统便能得知有新数据到来**，再通过**网卡的中断程序**去处理这些数据。

### 进程阻塞为什么不占用 CPU 资源？

了解 epoll 的第三步，要从 **操作系统进程调度**的角度来看数据接收。阻塞是进程调度的关键一环，指的是进程在等待某事件(如接收网络数据)发生之前的等待状态，recv、select 和 epoll 都是阻塞方法。

为了简单器件，我们从普通的 recv 接收开始分析，先看看下面代码：

```c++
// 创建 socket
int s = socket(AF_INET, SOCK_STREAM, 0);

// 绑定
bind(s, ...)

// 监听
listen(s, ...)

// 接受客户端连接
int c = accept(s, ...)

// 接收客户端数据
recv(c, ...)

// 将数据打印
print(...)
```
这是一段最基础的网络编程代码，先新建 socket 对象，依次掉用 bind、listen、accept，最后调用 recv 接收数据。recv 是一个阻塞方法，当程序运行到 recv 时，它会一直等待，直到接收到数据才往下执行。

那么阻塞的底层原理是什么？？

#### 工作队列

操作系统为了支持**多任务**，实现了进程调度的功能，会把进程分为**运行态**和**等待态**等几种状态。运行状态是进程获得 CPU 使用权，正在执行代码的状态；等待状态是阻塞状态，比如上述程序运行到 recv 的时候，程序会从运行态转换为等待状态，接收到数据后又变回运行状态。操作系统会分时执行各个运行状态的进程，由于速度很快，看上去就像是同时执行多个任务。

下图 4 中的计算机中运行着 A、B、C 三个进程，其中进程 A 执行着上述基础网络程序，一开始，这 3 个进程都被操作系统的工作队列所引用，都处于运行状态，会分时执行。

<div align="center"><img src=./picture/what-is-it-004-004.svg /></div>

#### 等待队列

当进程 A 执行到创建 socket 的语句时，操作系统会创建一个由**文件系统**管理的 socket 对象(如下图 5)。这个 socket 对象包含了发送缓冲区、接收缓冲区、**等待队列**等成员。**等待队列是个非常重要的结构，它指向所有需要等待该 socket 事件的进程**。

<div align="center"><img src=./picture/what-is-it-004-005.svg /></div>

当程序执行到 recv 时，操作系统会将进程 A 从工作队列移动到该 socket 的等待队列中(如下图 6)。由于工作队列只剩下了进程 B 和进程 C，依据进程调度，CPU 会轮流调度执行这两个进程的程序，不会执行进程 A 的程序。**所以进程 A 被阻塞，不会往下执行代码，也不会占用 CPU 资源**。

<div align="center"><img src=./picture/what-is-it-004-006.svg /></div>

> 操作系统添加等待队列只是添加了对这个**等待中进程**的引用，以便在接收到数据时获取进程对象、将其唤醒，而非直接将进程管理纳入自己之下。上图为了方便说明，直接将进程挂到socket 的等待队列之下。

#### 唤醒进程

当 socket 接收到数据后，操作系统将该 socket 等待队列上的进程重新放回到工作队列，该进程变成运行状态，继续执行代码。也由于 socket 的接收缓冲区已经有了数据，recv 可以返回接收到的数据。

### 内核接收网络数据全过程

这一步，贯穿网卡、中断、进程调度的知识，叙述阻塞 recv 下，内核接收数据全过程。

如下图 7 所示，进程在 recv 阻塞期间，计算机收到了对端传送的数据，数据经网卡传送到内存，然后网卡通过中断信号通知 CPU 有数据到达，CPU 执行中断程序。**此处的中断程序主要有两项功能，先将网络数据写入到对应 socket 的接收缓冲区里面，再唤醒进程 A，重新将进程 A 放入工作队列中**。

<div align="center"><img src=./picture/what-is-it-004-007.svg /></div>

这里有两个问题需要思考一下：

- 操作系统如何知道网络数据对应于哪个 socket？
- 如何同时监视多个 socket 的数据？

第一个问题：因为一个 socket 对应着一个端口号，而网络数据包中包含了 IP 和 PORT 的信息，内核可以通过端口号找到对应的 socket。当然，为了提高处理速度，操作系统会维护端口号到 socket 的索引结构，以便快速读取。

第二个问题是**重中之重**，是后面**多路复用**的重点。

### 同时监视多个 socket 的方法

服务端需要管理多个客户端连接，而 recv 只能监视单个 socket，这种矛盾下，人们开始寻找监视多个 socket 的方法。**epoll 的要义就是高效的监视多个 socket**。从发展角度看，必然先出现一种不太高效的方法，后人再加以改进。只有先了解了不太高效的方法，才能够理解 epoll 的本质。

假如能够预先传入一个 socket 列表，**如果列表中的 socket 都没有数据，挂起进程，直到有一个 socket 收到数据，唤醒进程**。这种方式很直接，也是 select 方式的思想。

为方便理解，我们先复习 select 的用法。在如下的代码中，先准备一个数组（下面代码中的fds），让fds存放着所有需要监视的socket。然后调用select，如果fds中的所有socket都没有数据，select会阻塞，直到有一个socket接收到数据，select返回，唤醒进程。用户可以遍历fds，通过FD_ISSET判断具体哪个socket收到数据，然后做出处理。

```c++
// 创建 socket
int s = socket(AF_INET, SOCK_STREAM, 0);
// 绑定
bind(s, ...)
// 监听
listen(s, ...)

int fds[] =  存放需要监听的socket

while(1){
    int n = select(..., fds, ...)
    for(int i=0; i < fds.count; i++){
        if(FD_ISSET(fds[i], ...)){
            //fds[i]的数据处理
        }
    }
}
```

#### select 的监听流程

select的实现思路很直接。假如程序同时监视如下图的sock1、sock2和sock3三个socket，那么在调用select之后，操作系统把进程A分别加入这三个socket的等待队列中。

<div align="center"><img src=./picture/what-is-it-004-008.svg /></div>

#### select 的唤醒流程

所谓唤起进程，就是将进程从所有的等待队列中移除，加入到工作队列里面。如下图 9 所示。

<div align="center"><img src=./picture/what-is-it-004-009.svg /></div>

经由这些步骤，当进程 A 被唤醒后，它知道至少有一个 socket 接收了数据。程序只需遍历一遍 socket 列表，就可以得到就绪的 socket。

这种简单方式行之有效，在几乎所有操作系统都有对应的实现。

#### select 的缺点

**最简单的方法往往有缺点，主要是**：
- 每次调用 select 都需要将进程加入到所有监视 socket 的等待队列中，每次唤醒又需要从每个 socket 的等待队列中移除。这里涉及了两次遍历，而且每次都要将整个fds列表传递给内核，有一定的开销。正是因为遍历操作开销大，**出于效率的考量，才会规定 select 的最大监视数量，默认只能监视 1024 个 socket**。
- 进程被唤醒后，程序并不知道哪些socket收到数据，还需要遍历一次。

那么，有没有能减少遍历的方式？有没有保存就绪 socket 的方法？这两个问题便是 epoll 技术要解决的。

> 上面只解释了select的一种情形。当程序调用select时，内核会先遍历一遍socket，如果有一个以上的socket接收缓冲区有数据，那么select直接返回，不会阻塞。这也是为什么select的返回值有可能大于1的原因之一。如果没有socket有数据，进程才会阻塞。

### epoll 的设计思路

epoll 是在 select 出现 N 年之后才被发明的，是 select 和 poll 的增强版本。epoll 通过以下一些措施来改进效率。

**1. 功能分离**

select 低效的原因之一是将 **维护等待队列** 和 **阻塞进程**两个步骤合二为一。如下图 10 所示，每次调用 delect 都需要这两个步骤，然而大多数应用场景中，需要监视的 socket 相对固定，并不需要每次都修改。epoll 将这两个操作分开，先用 epoll_ctl 维护等待队列，再调用 epoll_wait 阻塞进程。显而易见的，效率就能得到提升。

<div align="center"><img src=./picture/what-is-it-004-010.svg /></div>

为了方便理解后续的内容，我们先复习下 epoll 的用法。如下的代码中，先用 epoll_create 创建一个 epoll 对象 epfd，再通过 epoll_ctl 将需要监视的 socket 添加到 epfd 中，最后调用 epoll_wait 等待数据。

```c++
int s = socket(AF_INET, SOCK_STREAM, 0);   
bind(s, ...)
listen(s, ...)

int epfd = epoll_create(...);
epoll_ctl(epfd, ...); //将所有需要监听的socket添加到epfd中

while(1){
    int n = epoll_wait(...)
    for(接收到数据的socket){
        //处理
    }
}
```

由于功能分离，使得 epoll 有了优化的可能。

**2. 就绪列表**

select 抵消了的另外一个原因在于程序不知道哪些 socket 收到数据，只能一个个遍历。如果内核维护一个 **就绪列表**，引用收到数据的 socket，就能避免遍历。如下图11所示，计算机共有三个 socket，收到数据的 socket2 和 socket3 被 **rdlist(就绪列表)** 所引用。当进程被唤醒后，只要获取 rdlist 的内容，就能够知道哪些 socket 收到数据。

<div align="center"><img src=./picture/what-is-it-004-011.svg /></div>

### epoll 的原理和流程

本节会以示例和图表来介绍 epoll 的原理和流程。

**1. 创建 epoll 对象**

如下图 12 所示，当某个进程调用 epoll_create 方法时，内核会创建一个 eventpoll 对象(也就是程序中的 epfd 对象)。eventpoll 对象也是文件系统中的一员，和 socket 一员，它也会有等待队列。

<div align="center"><img src=./picture/what-is-it-004-012.svg /></div>

创建一个代表该 epoll 的 eventpoll 对象是必须的，因为内核要维护就绪列表数据，**就绪列表可以作为 eventpoll 的成员**。

**2. 维护监视列表**

创建 epoll 对象后，可以用 epoll_ctl 添加或删除所要监听的 socket。下面以添加 socket 为例，如下图 13 所示，如果通过 epoll_ctl 添加 socket1， socket2， socket3 的监视，内核会将 eventpoll 添加到这三个 socket 的等待队列中。

<div align="center"><img src=./picture/what-is-it-004-013.svg /></div>

当 socket 收到数据后，中断程序会操作 eventpoll 对象，而不是直接操作进程。

**3. 接收数据**

当 socket 接收到数据后，中断程序会给 eventpoll 的就绪列表添加 socket 引用。如下图 14 所示，socket2 和 socket3 接收到数据后，中断程序让 rdlist 引用这两个 socket。

<div align="center"><img src=./picture/what-is-it-004-014.svg /></div>

eventpoll 对象相当于是 socket 和进程之间的中介，socket 的数据接收并不直接影响进程。而是通过改变 eventpoll 的就绪列表来改变进程状态。

当程序执行到 epoll_wait 时，如果 rdlist 已经引用了 socket，那么 epoll_wait 直接返回，如果 rdlist 为空，阻塞进程。

**4. 阻塞和唤醒进程**

假设计算机正在运行 进程 A 和进程 B，在某时刻进程 A运行到了 epoll_wait 语句。如下图 15 所示，内核将会将进程 A 放入 eventpoll 的等待队列中，阻塞进程。

<div align="center"><img src=./picture/what-is-it-004-015.svg /></div>

当 socket 接收到数据，中断程序一方面修改 rdlist，另一方面唤醒 eventpoll 等待队列中的进程，进程 A 再次进入运行状态。也因为有 rdlist 的存在，进程 A 可以知道哪些 socket 发生了变化。

<div align="center"><img src=./picture/what-is-it-004-016.svg /></div>

### epoll 的实现细节

至此，我们对 epoll 的本质已经有了一定的了解。但我们还有一个问题，**eventpoll 的数据结构是怎样的？**

我们可以先思考两个问题：
- **就绪队列** 应该用什么数据结构？
- **eventpoll** 应该用什么数据结构？

<div align="center"><img src=./picture/what-is-it-004-017.svg /></div>

**1. 就绪列表的数据结构**

就绪列表引用着就绪的socket，所以它应能够快速的插入数据。

程序可能随时调用epoll_ctl添加监视socket，也可能随时删除。当删除时，若该socket已经存放在就绪列表中，它也应该被移除。

所以就绪列表应是一种能够快速插入和删除的数据结构。双向链表就是这样一种数据结构，epoll使用**双向链表**来实现就绪队列(对应上图的rdllist, *这里我觉得单链表也可以*)。

**2. 索引结构**

既然epoll将维护监视队列和进程阻塞分离，也意味着需要有个数据结构来保存所有监视的socket。至少要方便的添加和移除，还要便于搜索，以避免重复添加。红黑树是一种自平衡二叉查找树，搜索、插入和删除时间复杂度都是O(log(N))，效率较好。epoll使用了**红黑树**作为索引结构（对应上图的rbr）

> ps：因为操作系统要兼顾多种功能，以及由更多需要保存的数据，rdlist并非直接引用socket，而是通过epitem间接引用，红黑树的节点也是epitem对象。同样，文件系统也并非直接引用着socket。为方便理解，本文中省略了一些间接结构。

### 结论

epoll 在 select 和 poll（poll 和 select 基本一样，有少量改进）的基础**引入了eventpoll 作为中间层，使用了先进的数据结构**，是一种高效的多路复用技术。

### 参考资料
1. [罗培羽知乎专栏](https://zhuanlan.zhihu.com/p/63179839)
2. 《Linux高性能服务器编程》