# 一、进程中如何通信

**重要的进程间通信（不同进程之间传播或交换信息）方式分为六种**

管道、信号、消息队列、共享内存、信号量、socket。其中前五种主要用于一台主机之中的各个进程之间的通信，socket套接字通信主要用于**网络之中不同主机之间的通信**。

## 1.1 管道

管道（Pipe）其本质是**由内核维护的一段内存缓存区**。一个进程向该缓存写入数据，另一个进程从中读取数据，形成单向数据流。管道传输的数据是**无格式的字节流**，且受内核缓冲区大小的限制。

### 1.1.1 核心特性

1. **单向通信**
   匿名管道仅支持单向数据传输（一端写入，另一端读取），若需双向通信，必须建立两条独立的管道。这种单向性体现了其**半双工通信**的特性。
2. **亲缘关系依赖**
   - **匿名管道**：通常用于父子进程或兄弟进程等有亲缘关系的进程间通信。子进程通过继承父进程的文件描述符访问管道。
   - **命名管道（FIFO）**：通过文件系统中的路径标识，允许无亲缘关系的进程通过打开同一路径进行通信，突破了匿名管道的亲缘限制。
3. **阻塞与非阻塞模式**
   - 默认情况下，读进程在管道无数据时会阻塞等待；写进程在缓冲区满时也会阻塞，直到有空间释放。
   - 可通过`fcntl`函数设为非阻塞模式：读空管道时直接返回`EAGAIN`错误，写满时丢弃数据或部分写入。
4. **生命周期管理**
   - 匿名管道随进程终止自动销毁。
   - 命名管道需手动删除其文件路径（如`unlink`），否则会持久存在于文件系统中。
5. **容量限制**
   内核缓冲区大小固定（通常为4KB~64KB）。若写入速度远超读取速度，写进程可能被长时间阻塞，需设计合理的读写协同逻辑。

### 1.1.2 缺点

1. **半双工通信的天然限制**
   匿名管道仅支持单向数据传输，双向通信需额外建立一条管道，增加了资源管理和协调的复杂度。
2. **读写阻塞的强依赖性**
   若管道内的数据未被读进程及时消费，写进程会因缓冲区满而阻塞，直到读进程取走数据。这种强同步机制可能导致进程间**死锁**（如双方同时等待对方读写）。

### 1.1.3 匿名管道与命名管道的对比

| **特性**     | **匿名管道**                   | **命名管道（FIFO）**         |
| :----------- | :----------------------------- | :--------------------------- |
| **创建方式** | `pipe()`系统调用               | `mkfifo()`命令或函数         |
| **通信方向** | 半双工（单向，需双向则建两条） | 半双工，但支持多进程读写     |
| **进程关系** | 仅限亲缘进程                   | 任意进程（通过文件路径访问） |
| **持久性**   | 随进程结束销毁                 | 需手动删除文件路径           |

> **注意事项**
>
> - **数据原子性**：若单次写入数据量小于`PIPE_BUF`（通常512B~4KB），内核保证写入的原子性；反之可能被拆分。
> - **同步问题**：共享内存需配合信号量，而管道自身通过阻塞机制隐式同步，但仍需注意读写端协调。
> - **性能瓶颈**：高频大数据传输时，管道可能因拷贝开销和容量限制成为瓶颈，此时可改用共享内存。

## 1.2 信号

信号（Signal）是轻量级异步通知机制，由内核或进程向目标进程发送特定事件的通知。其本质是**预定义的事件编号**（如`SIGINT`对应终端中断），用于触发进程的默认行为或自定义处理逻辑。

信号是进程间通信中最简单、最直接的异步通知机制，适用于事件驱动、进程控制等场景。但其设计初衷是“通知”而非“数据传输”，因此复杂交互需结合其他IPC机制（如管道、共享内存）。

### 1.2.1 核心特性

1. **异步通知**
   信号在任意时刻可中断进程当前操作，直接跳转到信号处理函数执行，与进程的执行流无关。
2. **预定义类型**
   系统定义了约30种标准信号，编号范围通常为1~31。
3. **处理方式灵活性**
   - **默认行为**：终止进程、忽略信号、暂停进程。
   - **自定义处理**：通过`signal()`或`sigaction()`注册用户函数（如处理`SIGINT`实现优雅退出）。
4. **生命周期**
   - **生成**：由内核、其他进程或终端触发。
   - **传递**：内核将信号加入目标进程的信号队列，等待进程调度处理。
   - **处理**：进程从内核态返回用户态时，检查并执行信号处理函数。

### 1.2.2 缺点

1. **信息传递能力弱**
   信号仅能传递事件编号，无法携带额外数据（实时信号如`SIGRTMIN`可携带少量信息，但需复杂处理）。
2. **信号丢失与覆盖**
   - 同类非实时信号多次到达时，可能被合并为一次。
   - 处理函数执行期间，新到达的同类型信号可能被阻塞。
3. **处理函数的安全限制**
   信号处理函数需为**可重入函数**，避免使用非线程安全操作。
4. **实时性受限**
   非实时信号无优先级，内核可能延迟传递，无法保证严格时序。

### 1.2.3 信号分类对比

| **类型**     | **非实时信号（标准信号）** | **实时信号（`SIGRTMIN~SIGRTMAX`）** |
| :----------- | :------------------------- | :---------------------------------- |
| **编号范围** | 1~31                       | 34~64（依系统不同）                 |
| **队列机制** | 不排队，多次发送可能合并   | 支持排队，按顺序处理                |
| **数据携带** | 不支持                     | 可通过`sigqueue()`附加数据          |
| **优先级**   | 无                         | 支持信号优先级                      |

> **注意事项**
>
> 1. **避免处理函数阻塞**
>    信号处理函数应快速完成，复杂逻辑可通过标记位在主线程序处理。
> 2. **信号屏蔽与竞态条件**
>    - 使用`sigprocmask`屏蔽关键代码段的信号，防止处理函数中断敏感操作。
>    - 处理共享资源时需考虑信号引发的竞态问题。
> 3. **系统调用中断**
>    信号可能中断阻塞的系统调用，需检查错误码`EINTR`并重试。
> 4. **信号与多线程**
>    多线程程序中，信号可能由任意线程处理，建议统一由主线程接管。

## 1.3 消息队列

消息队列（Message Queue）其本质是**由内核维护的链表结构**，允许进程以消息块（结构化数据）的形式异步通信。相比管道，消息队列支持更灵活的格式和随机读取，适用于频繁或结构化的数据交换场景。

消息队列弥补了管道在结构化数据和异步通信上的不足，适用于中等频率、结构化消息交换的场景。但其性能瓶颈（上下文切换与拷贝开销）使其难以应对超高频需求，此类场景可优先考虑共享内存或Unix域套接字。

### 1.3.1 核心特性

1. **异步非阻塞通信**
   - 发送进程将消息写入队列后立即返回，无需等待接收进程响应。
   - 接收进程可主动拉取消息，若队列为空可选择阻塞或非阻塞模式。
2. **结构化消息**
   - 消息包含**类型标识**和**数据体**，双方需约定格式。
   - 支持按消息类型读取，而非严格FIFO顺序。
3. **内核持久性**
   - 消息队列独立于进程存在，进程终止后消息仍保留在内核中。
   - 可通过权限控制限制其他进程访问。
4. **原子性保证**
   - 单次写入的消息若小于`MSGMAX`，内核保证原子性）。
5. **多进程共享**
   任意进程（需权限）均可通过队列标识符访问同一队列，支持多对多通信。

### 1.3.2 缺点

1. **消息大小限制**
   单条消息长度受内核参数`MSGMAX`限制（默认约8KB），超出需分片处理。
2. **性能开销**
   - **CPU上下文切换**：每次读写需通过系统调用进入内核态，频繁操作时开销显著。
   - **数据拷贝**：消息从用户空间拷贝到内核队列，再拷贝到接收方用户空间，高频场景效率低。
3. **队列容量限制**
   队列总大小受内核参数`MSGMNB`限制（默认约16KB~64KB），写满后发送进程默认阻塞。
4. **复杂性**
   需自行处理消息类型匹配、分片重组、队列满/空等问题，开发复杂度较高。

> **注意事项**
>
> 1. **消息类型设计**
>    - 类型值应明确区分用途（如正数用于请求，负数用于响应）。
>    - 避免类型冲突，建议使用枚举或宏定义。
> 2. **队列泄露防护**
>    - 确保进程退出前释放队列。
>    - 通过`ipcs -q`和`ipcrm`命令管理残留队列。
> 3. **超长消息处理**
>    - 若消息长度超过`MSGMAX`，需在应用层分片发送，接收端重组。
> 4. **信号量同步（可选）**
>    - 多进程竞争读写时，可结合信号量实现互斥锁，避免消息覆盖。

## 1.4 共享内存

共享内存（Shared Memory）是进程间通信（IPC）中**速度最快**的机制，其本质是**由内核分配的一段物理内存区域**，被多个进程映射到各自的虚拟地址空间中。进程通过直接读写该内存区域实现数据交互，无需内核中转或数据拷贝，从而极大提升通信效率。

共享内存是进程间通信的性能天花板，尤其适合对吞吐量和延迟敏感的场景。但其“直接访问”的特性如同一把双刃剑，在提供极致速度的同时，也要求开发者严格管理同步与数据一致性。结合信号量、互斥锁等机制，可构建高效且稳定的多进程协作系统。

### 1.4.1 核心特性

1. **零拷贝高效性**
   数据直接在共享内存区域读写，避免了管道、消息队列等机制中用户态与内核态间的数据拷贝开销。
2. **虚拟地址映射**
   - 每个进程通过页表将共享内存映射到自身虚拟地址空间的不同位置。
   - 进程通过虚拟地址访问共享内存，由MMU（内存管理单元）完成虚实地址转换。
3. **多进程并发访问**
   多个进程可同时映射同一共享内存区域，实现高速数据共享，但需配合**同步机制**（如信号量、互斥锁）避免竞争。
4. **内核持久性**
   - 共享内存独立于进程存在，进程退出后仍保留（除非显式删除）。
   - 通过`shmctl(IPC_RMID)`销毁或系统重启后清除。

### 1.4.2 缺点

1. **同步复杂度高**

   需额外机制（如信号量）协调读写

2. **安全隐患**

   恶意进程可能改写数据

3. **生命周期管理**

   需显示删除避免内存泄漏

> **注意事项**
>
> 1. **内存对齐与访问**
>    - 确保数据结构对齐，避免不同进程因编译差异导致的内存解释错误。
> 2. **缓存一致性**
>    - 多核CPU中，共享内存可能引发缓存一致性问题，需通过内存屏障或原子操作保证可见性。
> 3. **安全与权限控制**
>    - 设置严格的IPC权限（如`0666`仅允许同组用户访问），防止未授权进程篡改数据。
> 4. **资源泄漏防护**
>    - 确保进程退出前调用`shmdt()`和`shmctl()`，避免内存段永久占用。

## 1.5 信号量

信号量（Semaphore）是进程间或线程间**同步与互斥**的核心工具，其本质是**由内核维护的整型计数器**，用于协调多个执行单元对共享资源的访问。信号量的核心思想是通过`P`（等待）和`V`（释放）操作，实现资源的原子性分配与释放，避免竞态条件（Race Condition）。

信号量是解决并发编程中同步与资源分配问题的基石，其灵活性使其适用于从简单互斥到复杂资源管理的广泛场景。然而，信号量的低级特性也要求开发者对并发逻辑有深刻理解，避免死锁、饥饿等典型问题。在实际开发中，可优先使用高层抽象（如线程池、无锁队列），但在需要精细控制时，信号量仍是不可替代的工具。

### 1.5.1 核心特性

1. **计数器抽象**
   - 信号量值表示当前可用资源数量：
     - **正值**：剩余可用资源数。
     - **零值**：资源已被完全占用，请求者需等待。
     - **负值**：绝对值表示等待该资源的进程/线程数。
2. **原子操作**
   - `P操作`（Proberen，尝试获取）：
     若信号量值 > 0，则减1并继续；否则阻塞等待。
   - `V操作`（Verhogen，释放资源）：
     信号量值加1，并唤醒一个等待进程。
3. **分类**
   - **二进制信号量**：值范围为0或1，等同于互斥锁（Mutex）。
   - **计数信号量**：值范围≥0，表示资源池容量（如连接池限制）。
4. **内核与用户态实现**
   - **System V信号量**：内核维护，支持跨进程同步（如`semget()`）。
   - **POSIX信号量**：可位于共享内存中，支持进程或线程级同步（如`sem_init()`）。

### 1.5.2 缺点

1. **死锁风险**

   错误使用可能导致进程永久阻塞

2. **优先级反转**

   低优先级进程占用资源，高优先级进程饥饿

3. **复杂性**

   需手动管理信号量创建、初始化和销毁

> **注意事项**
>
> 1. **死锁预防**
>    - **顺序一致性**：所有进程以相同顺序获取信号量。
>    - **超时机制**：使用`sem_timedwait()`避免无限阻塞。
> 2. **信号量泄漏**
>    - System V信号量需显式调用`semctl(IPC_RMID)`删除。
>    - POSIX命名信号量需`sem_unlink()`防止残留。
> 3. **原子性与错误处理**
>    - 确保`P/V`操作的原子性（如`SEM_UNDO`标志应对进程崩溃）。
>    - 检查`sem_wait()`返回值，处理`EINTR`（信号中断）等错误。
> 4. **性能优化**
>    - 避免过度使用信号量，高频场景可结合自旋锁或无锁数据结构。

# 二、Socket

## 2.1 Socket原理

### 2.1.1 什么是Socket

在计算机通信领域，socket被翻译为套接字，他是计算机之间进行通信的一种约定或一种方式。通过socket这种约定，一台计算机可以接收其他计算机的数据，也可以向其他计算机发送数据。

socket起源于Unix，而Unix/Linux基本哲学之一就是”一切皆文件“，都可以用”打开open -→读写write/read–> 关闭close”模式来操作。

我的理解就是Socket就是该模式的一个实现：即socket是一种特殊的文件，一些sokcet函数就是对其他进行的操作（读写IO、打开、关闭）。

Socket()函数返回一个整型的Socket描述符，随后的连接建立、数据传输等操作都是通过该Socket实现的。

### 2.1.2 网络进程如何通信

我们要理解网络中进程如何通信，得解决两个问题：
　　ａ、我们要如何标识一台主机，即怎样确定我们将要通信的进程是在那一台主机上运行。
　　ｂ、我们要如何标识唯一进程，本地通过pid标识，网络中应该怎样标识？
解决办法：
　　ａ、TCP/IP协议族已经帮我们解决了这个问题，网络层的“ip地址”可以唯一标识网络中的主机
　　ｂ、传输层的“协议+端口”可以唯一标识主机中的应用程序（进程），因此，我们利用三元组（ip地址，协议，端口）就可以标识网络的进程了，网络中的进程通信就可以利用这个标志与其它进程进行交互

### 2.1.3 Sokcet如何通信

现在，我们知道了网络中进程如何进行通信，即利用三元组d[ip地址，协议，端口]可以进行网络间通信了，那我们应该怎么实现？因此，我们sokcet应运而生，他就是利用三元组解决网络通信的一个中间件工具，就目前而言，几乎所有应用程序都是采用socket。

socket通信的数据传输方式常用的有两种：

- SOCK_STREAM：表示面向连接的数据传输方式。数据可以准确无误的达到另一台计算机，如果损坏或丢失，可以重新发送，但效率相对较慢。常见的http协议就使用了SOCK_STREAM数据传输，因为要确保数据的正确性，否则网页不能正常解析。
- OCK_DGRAM：表示无连接的数据传输方式。计算机只管传输数据，不作数据校验，如果数据在传输中损坏，或者没有到达另一台计算机，是没有办法补救的。也就是说，数据错了就错了，无法重传。因为 SOCK_DGRAM 所做的校验工作少，所以效率比 SOCK_STREAM 高。

## 2.2 TCP/IP协议

### 2.2.1 概念

TCP/IP提供点对点的连接机制，将数据应该如何封装、定址、传输、路由以及在目的地如何接收，都加以标准化。它将软件通信过程抽象化为四个抽象层，采取协议堆栈的方式分别实现出不同通信协议。协议族下的各种协议，依其功能不同，被分别归属到四个层次结构中，常被视为是简化的七层OSI模型。

- **四层结构（由下至上）**：
  1. **网络接口层（链路层）**：负责物理介质的数据帧传输（如以太网协议）。
  2. **网络层（IP层）**：通过IP地址实现主机间的逻辑寻址和路由（如IP协议）。
  3. **传输层**：提供端到端的数据传输服务（如TCP、UDP协议）。
  4. **应用层**：面向用户提供具体服务（如HTTP、FTP协议）。

- TCP（传输控制协议）是面向连接的、可靠的、基于字节流的传输层协议。其**核心特性**包括：

  - **三次握手建立连接**：确保双方通信能力及初始序列号同步。

  - **四次挥手释放连接**：保证数据完整性并优雅关闭双工通道。

  - **超时重传、流量控制、拥塞控制**：保障数据传输的可靠性。

### 2.2.2 TCP数据报结构

![TCP协议及数据结构_tcp数据包结构-CSDN博客](https://img-blog.csdnimg.cn/ccd05bf87233433b8aacaf81d04dd026.png)

TCP报文头部固定20字节（不含选项字段），关键字段如下：

1. **源端口与目的端口（各16位）**：标识发送方和接收方的应用进程。
2. **序号（Seq，32位）**：本报文段发送数据的第一个字节的编号。
3. **确认号（Ack，32位）**：期望接收的下一个字节的编号，**Ack = 收到的Seq + 数据长度 + 1**（若数据长度为0，如SYN/FIN标志位，视为占1个序号）。
4. **数据偏移（4位）**：TCP首部长度（以4字节为单位）。
5. **标志位（6位）**：
   - **URG**：紧急指针有效（需配合紧急指针字段使用）。
   - **ACK**：确认号有效（建立连接后所有报文必须置1）。
   - **PSH**：接收方应立即将数据提交应用层。
   - **RST**：强制断开连接（异常终止）。
   - **SYN**：发起连接请求（同步序列号）。
   - **FIN**：请求终止连接。
6. **窗口大小（16位）**：接收方当前可接受的数据量（流量控制）。
7. **校验和（16位）**：确保数据完整性。

## 2.3 连接建立（三次握手）

### 2.3.1 建立过程

![TCP 的三次握手和四次挥手-CSDN博客](https://i-blog.csdnimg.cn/direct/08cacabbbb604580a192c4d01cec8ec1.png)

客户端调用 socket() 函数创建套接字后，因为没有建立连接，所以套接字处于CLOSED状态；服务器端调用 listen() 函数后，套接字进入LISTEN状态，开始监听客户端请求
这时客户端发起请求：

  　　1) 当客户端调用 connect() 函数后，TCP协议会组建一个数据包，并设置 SYN 标志位，表示该数据包是用来建立同步连接的。同时生成一个随机数字 1000，填充“序号（Seq）”字段，表示该数据包的序号。完成这些工作，开始向服务器端发送数据包，客户端就进入了SYN-SEND状态。
    　　2) 服务器端收到数据包，检测到已经设置了 SYN 标志位，就知道这是客户端发来的建立连接的“请求包”。服务器端也会组建一个数据包，并设置 SYN 和 ACK 标志位，SYN 表示该数据包用来建立连接，ACK 用来确认收到了刚才客户端发送的数据包
       服务器生成一个随机数 2000，填充“序号（Seq）”字段。2000 和客户端数据包没有关系。
       服务器将客户端数据包序号（1000）加1，得到1001，并用这个数字填充“确认号（Ack）”字段。
       服务器将数据包发出，进入SYN-RECV状态
      　　3) 客户端收到数据包，检测到已经设置了 SYN 和 ACK 标志位，就知道这是服务器发来的“确认包”。客户端会检测“确认号（Ack）”字段，看它的值是否为 1000+1，如果是就说明连接建立成功。
       接下来，客户端会继续组建数据包，并设置 ACK 标志位，表示客户端正确接收了服务器发来的“确认包”。同时，将刚才服务器发来的数据包序号（2000）加1，得到 2001，并用这个数字来填充“确认号（Ack）”字段。
       客户端将数据包发出，进入ESTABLISED状态，表示连接已经成功建立。
        　　4) 服务器端收到数据包，检测到已经设置了 ACK 标志位，就知道这是客户端发来的“确认包”。服务器会检测“确认号（Ack）”字段，看它的值是否为 2000+1，如果是就说明连接建立成功，服务器进入ESTABLISED状态。
       至此，客户端和服务器都进入了ESTABLISED状态，连接建立成功，接下来就可以收发数据了。

### 2.3.2 关键问题

#### 为什么是三次握手，而不是两次四次？

- 三次握手才可以阻止**重复历史连接的初始化**（主要原因）
- 三次握手才可以**双方同步的初始序列号**
- 三次握手才可以避免浪费资源

1. **阻止重复历史连接的初始化**（主要原因）

   - 在两次握手的情况下，服务端没有中间状态给客户端来阻止历史连接，导致服务端可能建立一个历史连接，造成资源浪费。

   - 三次握手已满足上述所有需求，额外增加握手次数（如四次）会引入不必要的延迟，且无法进一步解决核心问题。三次是理论上的最小安全交互次数。

2. **双方同步的初始序列号**

   当客户端发送携带「初始序列号」的 `SYN` 报文的时候，需要服务端回一个 `ACK` 应答报文，表示客户端的 SYN 报文已被服务端成功接收，那当服务端发送「初始序列号」给客户端的时候，依然也要得到客户端的应答回应，**这样一来一回，才能确保双方的初始序列号能被可靠的同步。**

3. 避免资源浪费

   如果只有「两次握手」，当客户端的 SYN 请求连接在网络中阻塞，客户端没有接收到 ACK 报文，就会重新发送 SYN ，由于没有第三次握手，服务器不清楚客户端是否收到了自己发送的建立连接的 ACK 确认信号，所以每收到一个 SYN 就只能先主动建立一个连接，这会造成什么情况呢？

   如果客户端的 SYN 阻塞了，重复发送多次 SYN 报文，那么服务器在收到请求后就会建立多个冗余的无效链接，造成不必要的资源浪费。


## 2.4 断开连接（四次挥手）

### 2.4.1 断连过程

TCP连接的释放需要**四次挥手**，其本质是双向通信的全双工特性决定的：每个方向必须独立关闭。以下为详细过程（以客户端主动关闭为例）：

1. **第一次挥手（FIN）**
   客户端调用`close()`后，发送**FIN报文**（FIN=1），进入`FIN_WAIT_1`状态，表示客户端不再发送数据，但仍可接收数据。
2. **第二次挥手（ACK）**
   服务端收到FIN后，立即回复**ACK报文**，进入`CLOSE_WAIT`状态。此时服务端可能仍有未发送完的数据，客户端收到ACK后进入`FIN_WAIT_2`状态。
3. **第三次挥手（FIN）**
   当服务端数据发送完毕，准备好关闭连接时，发送**FIN报文**，进入`LAST_ACK`状态，表示服务端不再发送数据。
4. **第四次挥手（ACK）**
   客户端收到FIN后，回复**ACK报文**，进入`TIME_WAIT`状态，等待**2MSL**（Maximum Segment Lifetime，报文最大生存时间）后关闭连接。服务端收到ACK后立即进入`CLOSED`状态。

![简述TCP的三次握手和四次挥手_三次握手四次挥手简述-CSDN博客](https://img-blog.csdnimg.cn/20210323204828555.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQ0NjQ3ODA5,size_16,color_FFFFFF,t_70)

### 2.4.2 关键问题

#### 为什么是四次挥手，不能是三次挥手

- **全双工通信的特性**
  TCP连接是全双工的，双方需独立关闭自己的数据通道。客户端发送FIN仅表示其不再发送数据（但可接收），服务端的ACK仅确认收到FIN。服务端的FIN需等待其数据发送完毕后再发送，因此ACK和FIN不能合并为一次。
- **数据完整性保障**
  若服务端收到FIN后立即合并ACK与FIN（变为三次挥手），可能丢失未传输完的数据。分开发送确保服务端有足够时间处理剩余数据。
- **可靠性设计**
  客户端最后的`TIME_WAIT`状态（等待2MSL）有两个作用：
  - 确保服务端收到最后的ACK。若ACK丢失，服务端会重传FIN，客户端可再次响应。
  - 防止旧连接的延迟报文干扰新连接。

#### 为什么不能是两次挥手

- 服务端未确认自身数据是否已发送完毕。
- 客户端无法确认服务端是否收到最终ACK，可能造成服务端持续等待。

# 三、TS通过Socket实现聊天室基础功能

## 3.1 服务端实现原理

1. **核心结构**

- 使用Node.js的`net`模块创建TCP服务器

- 定义了`Room`类型管理聊天室信息：

  ```typescript
  type Room = {
    roomName: string;   // 房间名称
    port: number;       // 监听端口
    users: [string, net.Socket][]; // 用户列表（[客户端地址, Socket对象]）
  };
  ```

2. **启动流程**

![image-20250429183123624](https://gitee.com/gomugenius/nei-fruit-palor_-picture/raw/master/img/deepseek_mermaid_20250429_fc7867.png)

3. **关键功能实现**

   - **客户端连接管理**：使用serverConnectEvent处理新连接

     - 记录客户端地址（client.remoteAddress:client.remotePort）

     - 存储Socket对象到用户列表

   - **消息广播机制**：

     ```typescript
     private broadcast(content: string) {
       for (const [_, userClient] of this.room.users) {
         if (userClient.writable) {
           userClient.write(content); // 向所有客户端发送消息
         }
       }
     }
     ```
   
4. **服务端实现示例**

   ```typescript
   // src/server/server.ts
   import * as net from 'net';
   import * as readline from 'readline';
   
   // 定义房间类型
   type Room = {
       roomName: string;
       port: number;
       users: [string, net.Socket][];
   };
   
   // 定义服务器类
   class MyTCPServer {
       private server: net.Server;
       private room: Room;
   
       constructor(port: number = 8080, roomName: string = '大厅') {
           this.room = {
               roomName,
               port,
               users: []
           };
           this.server = net.createServer(this.serverConnectEvent.bind(this));
           this.initServer();
           this.listenForShutdown()
       }
   
       private initServer() {
           this.server.listen(this.room.port, () => {
               console.log(`服务器已启动，监听端口 ${this.room.port}`);
           });
           this.server.on('close', () => {
               console.log('服务器已关闭');
           });
       }
   
       private serverConnectEvent(client: net.Socket) {
           console.log(`客户端已连接: ${client.remoteAddress}:${client.remotePort}`);
           //设计用户ID为标识
           const clientId = `${client.remoteAddress}:${client.remotePort}`;
           // 添加客户端到用户列表
           this.room.users.push([`${client.remoteAddress}:${client.remotePort}`, client]);
   
           client.on('data', (chunk) => {
               const content = chunk.toString();
               if( content === 'kick') {
                   this.disconnectClient(client)
               } else {
                   this.broadcast( `${clientId}: ${content}`, client)
               }
           });
           client.on('end', () => {
               console.log(`客户端已断开连接: ${client.remoteAddress}:${client.remotePort}`);
               this.removeClient(client);
           });
           client.on('error', (err) => {
               console.error(`客户端发生错误: ${err.message}`);
               this.removeClient(client);
           });
       }
   
       private broadcast(content: string, sender: net.Socket) {
           for (const [_, userClient] of this.room.users) {
               if (userClient.writable) {
                   userClient.write(content);
               }
           }
       }
   
       private removeClient(client: net.Socket) {
           const index = this.room.users.findIndex(([_, userClient]) => userClient === client);
           if (index !== -1) {
               this.room.users.splice(index, 1);
           }
       }
   
       //断开客户端连接
       private disconnectClient(client: net.Socket) {
           const clientInfo = `${client.remoteAddress}:${client.remotePort}`;
           console.log(`正在断开客户端连接: ${clientInfo}`);
           client.end();
           this.removeClient(client);
       }
   
       //关闭服务器
       private listenForShutdown() {
           const rl = readline.createInterface({
               input: process.stdin,
               output: process.stdout
           });
   
           rl.question('输入 "shutdown" 关闭服务器: ', (input: string) => {
               if (input === 'shutdown') {
                    // 关闭所有客户端连接
                   for (const [_, userClient] of this.room.users) {
                       userClient.destroy(); // 强制断开客户端
                   }
                   //关闭服务器
                   this.server.close(() => {
                       console.log('服务器已关闭');
                   });
                   rl.close();
               } else {
                   rl.close();
                   this.listenForShutdown();
               }
           });
       }
   }
   
   // 启动服务器
   new MyTCPServer();
   ```

## 3.2 客户端实现原理

1. **核心结构**

   - 使用net.Socket连接服务器
   - 通过readline模块实现控制台的输入
   - 事件驱动架构

2. **工作流程**

   ![](https://gitee.com/gomugenius/nei-fruit-palor_-picture/raw/master/img/deepseek_mermaid_20250429_80e053.png)

3. **关键功能实现**

   - **输入处理**：

     ```typescript
     private readInput() {
       this.rl.question('请输入消息: ', (input) => {
         this.client.write(input);
         this.readInput(); // 递归调用实现持续输入
       });
     }
     ```

   - **消息接收**:

     ```typescript
     this.client.on('data', (chunk) => {
       const content = chunk.toString();
       console.log(content); // 直接打印原始消息
     });
     ```

4. **客户端实现示例**

   ```typescript
   // src/client/client.ts
   import * as net from 'net';
   import * as readline from 'readline';
   
   // 定义客户端类
   class MyTCPClient {
       private client: net.Socket;
       private rl: readline.Interface;
   
       constructor() {
           this.client = new net.Socket();
           this.rl = readline.createInterface({
               input: process.stdin,
               output: process.stdout
           });
           this.connectToServer();
       }
   
       private connectToServer() {
           this.client.connect(8080, () => {
               console.log('已连接到服务器');
               this.readInput();
           });
           this.client.on('data', (chunk) => {
               const content = chunk.toString();
               console.log('你接收到了一条消息\n' + content);
           });
           this.client.on('end', () => {
               console.log('与服务器的连接已断开');
               this.rl.close();
           });
           this.client.on('error', (err) => {
               console.error(`与服务器的连接发生错误: ${err.message}`);
               this.rl.close();
           });
       }
   
       private readInput() {
           this.rl.question('请输入消息(输入"exit"断开连接):\n ', (input) => {
               if( input === 'exit') {
                   this.client.end();
                   this.rl.close();
               } else {
                   this.client.write(input);
                   this.readInput();
               }
           });
       }
   }
   
   // 启动客户端
   new MyTCPClient();
   ```

