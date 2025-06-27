# raft-网络

## 网络逻辑

### message

```c++
    //memcpy 函数直接按内存中的二进制表示逐字节复制数据
    //memcpy(dest, src, n) 会从 src 指向的内存地址开始，逐字节复制 n 个字节到 dest
```

#### LogEntry

message.h中，定义 struct LogEntry

```cpp
struct LogEntry
    {
        int term;         // 条目的任期
        std::string data; // 条目的数据

        // 序列化方法：string-》二进制 或 二进制-》string
        std::string serialize() const;
        static LogEntry deserialize(const char *data, size_t size);
        static LogEntry deserialize(const std::string &data);
        size_t getSerializedSize() const;
    };
```

\[term(4字节)（int)]\[数据长度(4字节)（unit32-t）][数据内容]

**string  LogEntry::serialize（）：**将logentry转为二进制形式，方法：char指针配合memcpy函数。

**LogEntry  LogEntry::deserialize：**从二进制logentry中读取logentry（任期，数据长度，数据)。提供const char* data, size_t size；const std::string& data两种传参方式

**size-t  getSerializedSizs：**返回logentry的size

#### 消息

定义消息类型➕消息头

```c++
    enum class MessageType : uint32_t
    {
        REQUESTVOTE_REQUEST = 1,
        REQUESTVOTE_RESPONSE = 2,
        APPENDENTRIES_REQUEST = 3,
        APPENDENTRIES_RESPONSE = 4
    };
```

```c++
		struct MessageHeader
    {
        MessageType type;      // 消息类型
        uint32_t payload_size; // 负载大小（不包括消息头）
    };
```

定义class Message：

**成员函数：**

**createNetworkMessage：**创建网络消息（二进制）：消息头➕消息=Type+payload.size+payload；payload=serialize（）成员变量term，data等。

**extraPayload（const char *data, size_t size）：**从网络消息（二进制）中提取消息体

还包括**子类**：

请求投票消息RequestVoteRequest，投票响应消息RequestVoteResponse，附加日志请求消息AppendEntriesRequest，附加日志响应消息AppendEntriesResponse。共四种消息子类。

每个子类内实现**虚函数**：

MessageType GetType（），

bool serialize（），

bool deserialize(const char *data, size_t size) 

#### 投票消息

**请求：class RequestVoteRequest：public message**

public：

```cpp
    class RequestVoteRequest : public Message
    {
    public:
        int term;           // 候选者的任期
        int candidate_id;   // 候选者ID
        int last_log_index; // 候选者的最后日志索引
        int last_log_term;  // 候选者最后日志的任期

        MessageType getType() const override
        {
            return MessageType::REQUESTVOTE_REQUEST;
        }

        std::string serialize() const override;
        bool deserialize(const char *data, size_t size) override;
    };
```

1.string serialize： memcpy+ptr:\[term(4字节)]\[candidate_id(4字节)]\[last_log_index(4字节)][last_log_term(4字节)]转为二进制，返回该二进制消息

2.bool deserialize：将二进制的消息提取出term，candidate-id，last-log-index，last-log-term字段，返回成功或失败（size < 4 * sizeof(int)）

**响应：class RequestVoteResponse : public Message**

[term(4字节)]\[vote_granted(1字节)]

同上

#### 追加消息

**请求：class AppendEntriesRequest : public Message**

```c++
        int term;                      // 领导者的任期
        int leader_id;                 // 领导者ID
        int prev_log_index;            // 前一个日志的索引
        int prev_log_term;             // 前一个日志的任期
        int leader_commit;             // 领导者的提交索引
        std::vector<LogEntry> entries; // 要附加的日志条目
        int seq;                       // 序列号，用于标识请求
```

**1.serialize（leader调用**

total_size = 6 * sizeof(int) + sizeof(uint32_t) + entries_size（entries_size += entry.getSerializedSize();）

六个字段：memcpy+ptr和前面写入方法相同

条目数字段：entry_count = static_cast<uint32_t>(entries.size());

条目本身：遍历entries，对每条entry执行LogEntry：：serialize，并memcpy➕ptr移动，循环执行

**2.deserialize：（follower调用**

提取出六个字段+条目数后，执行`entries.clear()`（清除之前的entries）

不断调用LogEntry::deserialize提取entry，`entries.push_back(entry)`;然后移动ptr继续提取和pushback

返回成功或失败（entries.size() != entry_count）

**响应：class AppendEntriesResponse**

```c
        int term;            // 当前任期号
        int follower_id;     // 跟随者ID
        int log_index;       // 跟随者的最新日志索引
        bool success;        // 是否成功添加日志
        int follower_commit; // 跟随者的提交索引
        int ack;             // 确认的序列号
```

 \[term(4)]\[follower_id(4)]\[log_index(4)]\[success(1)]\[follower_commit(4)][ack(4)]

同上

#### 解析消息

**parseMessage**

调用Message::extractPayload，得到header->type, payload，

调用auto message = createMessage(type);，得到一个消息子类实例，并调用该实例的deserialize(payload)，解析提取当前二进制消息中的信息

**createMessage(MessageType type)**

根据type，调用对应的子类构造函数，动态分配一块内存，在内存内构造一个该子类





##### **1. 代码主要运行逻辑**

这段代码实现了一个 **Raft 分布式共识算法的网络层模块（`NetworkManager`）**，负责节点间的通信（Raft 协议消息）和客户端请求处理。其核心运行流程如下：

---

**(1) 初始化阶段**

- **解析配置文件**：  
  - 读取节点配置（如 `follower_info`），确定本节点的 `client_port` 和 `raft_port`（默认 `raft_port = client_port - 1000`）。  
  - 存储其他节点的 IP 和端口信息到 `peers_` 列表。  
- **启动线程池**：  
  - 分为 **客户端请求处理线程池** 和 **Raft 消息处理线程池**，避免阻塞网络事件循环。  
- **初始化网络**：  
  - 创建 `epoll` 实例，监听客户端和 Raft 端口（非阻塞模式）。  
  - 忽略 `SIGPIPE` 信号（防止写关闭的 socket 导致进程退出）。

---

**(2) 主事件循环（`networkLoop`）**

- **监听事件**：  
  - 使用 `epoll_wait` 监听 socket 事件（新连接、数据到达、断开等）。  
- **处理事件**：  
  - **新连接**：调用 `handleNewConnection`，将 socket 加入 `epoll` 并记录连接类型（`CLIENT` 或 `RAFT`）。  
  - **数据到达**：调用 `processSocketData`，根据连接类型分发到不同处理逻辑：  
    - **客户端请求**：读取数据后提交到线程池异步处理（`asyncProcessClientRequest`）。  
    - **Raft 消息**：解析消息后提交到 Raft 线程池（`asyncProcessRaftMessage`），并更新节点 ID 与 socket 的映射关系。  
  - **连接断开**：关闭 socket 并清理相关资源（`closeConnection`）。

---

**(3) 节点间通信**

- **主动连接其他节点**：  
  - `connectToPeer` 尝试连接到配置中的其他节点（非阻塞 + `select` 超时检测）。  
- **断线重连**：  
  - 独立的重连线程定期检查未连接的节点并尝试重连。  
- **消息发送**：  
  - `sendMessage` 通过 socket 发送 Raft 消息（如 `AppendEntries`），失败时触发重连。

---

**(4) 回调机制**

- **客户端请求回调**：  
  - 由 `client_request_callback_` 处理业务逻辑（如键值存储操作），返回响应后通过 `sendClientResponse` 发送。  
- **Raft 消息回调**：  
  - 由 `message_callback_` 处理 Raft 协议逻辑（如选举、日志复制），返回响应消息（如 `AppendEntriesResponse`）。

---

##### **2. 主要运用到的技术**

**(1) 网络编程**

- **Socket API**：`socket`/`bind`/`listen`/`accept`/`connect`。  
- **I/O 多路复用**：`epoll` 实现高并发事件驱动模型。  
- **非阻塞 I/O**：`fcntl(fd, O_NONBLOCK)` 避免线程阻塞。  
- **TCP 优化**：`SO_REUSEADDR` 避免端口占用问题。

**(2) 多线程与同步**

- **线程池**：  
  - 使用 `ThreadPool` 异步处理客户端请求和 Raft 消息，避免阻塞网络线程。  
- **互斥锁**：  
  - `std::mutex` 保护共享数据（如 `connections_mutex_` 保护 `node_id_to_fd_` 等映射表）。  
- **生产者-消费者模型**：  
  - 网络线程生产任务（如接收消息），线程池消费任务（如处理请求）。

**(3) Raft 协议相关**

- **消息序列化/反序列化**：  
  - `MessageHandler` 类处理 Raft 消息的编解码（如 `AppendEntriesRequest`）。  
- **节点状态管理**：  
  - 维护 `node_id_to_fd_` 和 `fd_to_node_id_` 映射，快速定位节点连接。  
- **故障恢复**：  
  - 重连线程确保节点断开后自动恢复连接。

**(4) C++ 特性**

- **RAII**：  
  - 在析构函数中自动释放资源（如关闭 socket、停止线程）。  
- **智能指针**：  
  - `std::unique_ptr` 管理线程池和消息对象生命周期。  
- **Lambda 表达式**：  
  - 用于异步任务提交（如 `thread_pool_->enqueue([...]{...})`）。

---

##### **3. 面试常见问题与回答思路**

**Q1: 如何保证网络层的线程安全？**

- **答**：  
  - 使用 `std::mutex` 保护共享数据结构（如 `connections_mutex_` 保护 `fd_types_` 和 `node_id_to_fd_`）。  
  - 网络线程负责接收事件，耗时操作（如业务逻辑）交给线程池，避免阻塞。  

**Q2: 如何处理节点断连和重连？**

- **答**：  
  - 检测到 `EPOLLHUP/EPOLLERR` 事件时关闭连接并清理映射。  
  - 独立的重连线程定期检查未连接的节点，调用 `connectToPeer` 重试。  

**Q3: 为什么选择 `epoll` 而不是 `select`/`poll`？**

- **答**：  
  - `epoll` 时间复杂度 O(1)，适合高并发场景；`select`/`poll` 是 O(n)。  
  - `epoll` 支持边缘触发（ET）模式，减少无效事件通知。  

**Q4: Raft 消息如何保证有序到达？**

- **答**：  
  - TCP 本身保证数据有序，Raft 通过 `prev_log_index` 和 `prev_log_term` 检测日志连续性。  

**Q5: 如何优化大量小数据包的传输？**

- **答**：  
  - 启用 TCP `Nagle` 算法合并小包（但可能增加延迟）。  
  - 设计消息批处理机制（如合并多个 `AppendEntries`）。  

---

##### **总结**

- **核心逻辑**：事件驱动网络层 + 线程池异步处理 + Raft 消息路由。  
- **关键技术**：`epoll`、非阻塞 I/O、线程池、RAII、TCP 协议。  
- **面试亮点**：强调线程安全设计、故障恢复机制、性能优化（如避免锁竞争）。  

建议结合 Raft 论文和实际项目经验（如 etcd 源码）加深理解。

## TCP

TCP（**Transmission Control Protocol**，传输控制协议）是互联网核心协议之一，属于 **传输层（Transport Layer）** 的协议，用于在不可靠的 IP 网络（如互联网）上提供 **可靠的、面向连接的、基于字节流的** 数据传输服务。（详见计网笔记）

<img src="/Users/racy/Library/Application Support/typora-user-images/截屏2025-06-18 20.04.02.png" alt="截屏2025-06-18 20.04.02" style="zoom:50%;" />

## redis/**RESP**

本实验使用一种**简化版 RESP 协议（Redis 序列化协议）**来编码客户端发送给服务器的请求（request）和服务器返回给客户端的响应（response）。

##### 3.2.1.1 客户端请求消息

客户端使用 RESP 数组格式发送命令。格式如下：

- 以 `*` 开头，后跟数组元素数量（十进制），再跟回车换行（CRLF）；
- 接着是若干个字符串（bulk string），每个字符串格式为：
  - 以 `$` 开头，后跟字符串长度（十进制），再跟 CRLF；
  - 字符串内容；
  - 再跟一个 CRLF。

例子：字符串 `CS06142` 被编码为：

`$7\r\nCS06142\r\n`

一个字符串数组被编码为 `foo bar`：

`*2\r\n$3\r\nfoo\r\n$3\r\nbar\r\n`

##### 3.2.1.2 服务器响应消息

1. **成功消息**：

格式为：以 `+` 开头，后跟字符串（不含换行符），以 CRLF 结尾。例如：

`+OK\r\n`

1. **错误消息**：

格式为：以 `-` 开头，后跟错误字符串，以 CRLF 结尾。例如：

`-ERROR\r\n` 

1. **RESP 数组消息**：

执行 `GET` 命令成功时，服务器返回 RESP 数组格式。例如：

```
*2\r\n$5\r\nCloud\r\n$9\r\nComputing\r\n
```

1. **整数消息**：

某些命令（如 `DEL`）需返回整数值。格式为以 `:` 开头的整数，结尾为 CRLF。例如：

```
:1\r\n
```

**数据库消息**

`SET key value`  `GET key` `DEL key1 key2 ...`

##### redis

“我并没有实现完整的 Redis 数据库（**内存键值存储数据库**），而是 **基于 TCP 实现了 RESP（REdis Serialization Protocol）协议解析器**，用于客户端与服务端的通信。
比如：

- 将 `SET key value` 编码成 RESP 格式的二进制数据：`*3\r\n$3\r\nSET\r\n$5\r\nkey\r\n$5\r\nvalue\r\n`
- 解析服务端返回的 `+OK\r\n` 或 `-ERROR\r\n`
  这类似于 Redis 的 **网络通信模块**，但仅聚焦于协议编解码，不涉及存储、命令处理等核心功能。

“RESP 是 Redis 官方规定的通信协议，所有 Redis 客户端/服务端都依赖它交互数据。
我的工作相当于 **实现了一个轻量版 Redis 客户端网络层**（类似 `hiredis` 库的核心部分），但未涉及 Redis 的存储、集群等复杂功能。
例如：

- 基于 Socket + RESP 实现了一个能发送 `GET/SET` 命令并解析响应的 demo。
- 如果要扩展成完整 Redis，还需实现命令处理、持久化等模块。”

## epoll

###### 基础知识（tcp socket，recv阻塞，select）

**tcp socket连接过程**

```
监听 Socket: 0.0.0.0:80  
通信 Socket 1: 0.0.0.0:80 ←→ 192.168.1.2:54321  
通信 Socket 2: 0.0.0.0:80 ←→ 192.168.1.3:12345  
```

| 函数              | 作用                                                         |
| :---------------- | :----------------------------------------------------------- |
| `socket()`        | 创建 Socket，返回文件描述符（`AF_INET` 表示 IPv4，`SOCK_STREAM` 表示 TCP）。 |
| `bind()`          | 绑定 IP 和端口（服务器端必需）。                             |
| `listen()`        | 启动监听，设置等待连接队列的最大长度（`SOMAXCONN` 表示系统最大值）。 |
| `accept()`        | 接受客户端连接，返回新的 Socket 文件描述符（阻塞直到有连接到达）。 |
| `connect()`       | 客户端主动连接服务器（需指定服务器 IP 和端口）。             |
| `send()`/`recv()` | 发送/接收数据（实际是写入/读取内核缓冲区）。                 |
| `close()`         | 关闭 Socket，触发四次挥手。                                  |

**Q1: TCP 三次握手和四次挥手的细节？**

- **握手**：`SYN` → `SYN+ACK` → `ACK`（确认双方收发能力）。
- **挥手**：`FIN` → `ACK` → `FIN` → `ACK`（全双工连接需独立关闭）。

**Q2: 如果 `recv()` 返回 0 是什么意思？**

- 表示对方已关闭连接（收到 `FIN` 包）。

**Q3: 如何优化 TCP 传输速度？**

- 调整窗口大小、禁用 Nagle 算法、启用 TCP Fast Open（TFO）。

**Q4: `select`/`poll`/`epoll` 的区别？**

- `select`：跨平台，但效率低（O(n)）。
- `epoll`：Linux 专属，高性能（O(1)），支持边缘触发。

当线程调用阻塞式系统调用（如 `accept()` 或 `recv()`）时，如果所需条件未满足（如没有新连接或数据未到达），**内核会将线程状态标记为“睡眠”（TASK_INTERRUPTIBLE）**，并将其移出 CPU 调度队列。

线程不再占用 CPU 资源，直到内核检测到条件满足（如新连接到达或数据到达缓冲区），再将其重新加入调度队列。

**内核管理 Socket 的核心资源**

- **连接队列**：`accept()` 依赖内核维护的“未完成连接队列”（半连接队列）和“已完成连接队列”。

- **数据缓冲区**：`recv()`/`send()` 操作的内核缓冲区，由内核负责与网卡交互。

- 线程通过系统调用（如 `accept()`、`recv()`）进入内核态，内核决定其运行或阻塞。

- 当 Socket 事件就绪（如新连接到达），内核唤醒阻塞的线程，将其重新加入调度队列。

      +-------------------+       +-------------------+
      |    用户线程        |       |      内核         |
      +-------------------+       +-------------------+
               |                           |
               | 调用 accept()             |
               |-------------------------->|
               |                           | 检查“未完成连接队列”
               |                           |───若无连接，线程加入等待队列
               |                           |   线程进入睡眠状态
               |                           |
               |                           |───新连接到达（完成三次握手）
               |                           |   唤醒线程
               |<--------------------------|
               | 返回新连接的 Socket FD     |
               |                           |

![img](https://pic2.zhimg.com/v2-696b131cae434f2a0b5ab4d6353864af_r.jpg)

服务端需要管理多个客户端连接，而recv只能监视单个socket，这种矛盾下，人们开始寻找监视多个socket的方法。epoll的要义是**高效**的监视多个socket。

**select**

```cpp
int s = socket(AF_INET, SOCK_STREAM, 0);  
bind(s, ...)
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

假如能够预先传入一个socket列表，**如果列表中的socket都没有数据，挂起进程，直到有一个socket收到数据，唤醒进程**。这种方法很直接，也是select的设计思想。

为方便理解，我们先复习select的用法。在如下的代码中，先准备一个数组（下面代码中的fds），让fds存放着所有需要监视的socket。然后调用select，如果fds中的所有socket都没有数据，select会阻塞，直到有一个socket接收到数据，select返回，唤醒进程。用户可以遍历fds，通过FD_ISSET判断具体哪个socket收到数据，然后做出处理。

![img](https://pica.zhimg.com/v2-85dba5430f3c439e4647ea4d97ba54fc_r.jpg)‘

当任何一个socket收到数据后，中断程序将唤起进程。下图展示了sock2接收到了数据的处理流程。

**缺点**：

其一，每次调用select都需要将进程加入到所有监视socket的等待队列，每次唤醒都需要从每个队列中移除。这里涉及了两次遍历，而且每次都要将整个fds列表传递给内核，有一定的开销。正是因为遍历操作开销大，出于效率的考量，才会规定select的最大监视数量，默认只能监视1024个socket。

其二，进程被唤醒后，程序并不知道哪些socket收到数据，还需要遍历一次。

###### epoll

先用[epoll_ctl](https://zhida.zhihu.com/search?content_id=102382364&content_type=Article&match_order=1&q=epoll_ctl&zhida_source=entity)维护等待队列，再调用[epoll_wait](https://zhida.zhihu.com/search?content_id=102382364&content_type=Article&match_order=1&q=epoll_wait&zhida_source=entity)阻塞进程。

```cpp
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

eventpoll对象相当于是socket和进程之间的中介，socket的数据接收并不直接影响进程，而是通过改变eventpoll的就绪列表来改变进程状态。

当程序执行到epoll_wait时，如果rdlist已经引用了socket，那么epoll_wait直接返回，如果rdlist为空，阻塞进程。当socket接收到数据，中断程序一方面修改rdlist，另一方面唤醒eventpoll等待队列中的进程，进程A再次进入运行状态（如下图）。也因为rdlist的存在，进程A可以知道哪些socket发生了变化。

![img](https://pic2.zhimg.com/v2-40bd5825e27cf49b7fd9a59dfcbe4d6f_r.jpg)

`epoll` 的工作流程分为三部分：**红黑树**、**就绪链表**、**用户通知**。以下是具体步骤：

**(1) 红黑树：存储所有待监听的 Socket**

- **作用**：高效管理需要监视的 Socket（文件描述符 fd）。
- **操作**：
  - 当调用 `epoll_ctl(EPOLL_CTL_ADD)` 添加一个 Socket 时，内核将其插入红黑树。
  - 当调用 `epoll_ctl(EPOLL_CTL_DEL)` 删除时，从树中移除。
- **为什么用红黑树？**
  插入、删除、查找的时间复杂度均为 **O(log n)**，适合动态增删大量 Socket。

**(2) 就绪链表：存储有事件发生的 Socket**

- **作用**：临时存放已触发事件的 Socket（如数据到达、连接请求）。
- **触发机制**：
  当网卡收到数据或连接建立时，内核将对应的 Socket 从红黑树移到就绪链表。

**(3) 用户通知：返回就绪事件**

- 当用户调用 `epoll_wait()` 时，内核直接返回就绪链表中的 Socket 列表，无需遍历所有监听的 Socket。
- **优势**：时间复杂度 **O(1)**（仅检查链表是否非空）。

## 什么是RPC

raftnode-inicomponents

```cpp
// 消息处理回调
// 网络板块的消息处理实际在raftnode板块完成
network_manager_->setMessageCallback([this](int from_node_id, const Message &message) -> std::unique_ptr<Message>
                                     { return this->handleMessage(from_node_id, message); });
// 网络板块的客户信息处理实际在node板块完成
network_manager_->setClientRequestCallback([this](int client_fd, const std::string &request) -> std::string
                                           { return this->handleClientRequest(client_fd, request); });
// raft的消息发生实际在网络板块完成
//                             回调函数类型         传入的回调函数
/*void setSendMessageCallback(SendMessageCallback callback)
{
    raftcore.cpp中sendmessage函数实际调用的就是send_message_callback，
    即设置的回调函数，即网络模块的network_manager_->sendMessage(target_id, message)
    send_message_callback_ = callback;
}*/
// 设置Raft核心的发送消息回调
raft_core_->setSendMessageCallback([this](int target_id, const Message &message) -> bool
                                   { return network_manager_->sendMessage(target_id, message); });
```

-  网络模块的消息处理-》回调node-》调用raftcore处理消息（处理投票请求，处理投票回复，处理同步请求，处理同步回复等
-  **requestVoteRPC+AppendEntriesRPC：** Raft核心的发送消息回调--》调用networkmanager-sendmessager（如发送投票请求消息，发出同步请求）
-  网络板块客户端信息处理--》回调node中handleClientRequest函数处理

RPC就是要像调用本地的函数一样去调远程函数。在研究RPC前，我们先看看本地调用是怎么调的。假设我们要调用函数Multiply来计算lvalue * rvalue的结果:

在远程调用时，我们需要执行的函数体是在远程的机器上的，也就是说，Multiply是在另一个进程中执行的。这就带来了几个新问题：

1. **[Call ID映射](https://zhida.zhihu.com/search?content_id=70367603&content_type=Answer&match_order=1&q=Call+ID映射&zhida_source=entity)**。我们怎么告诉远程机器我们要调用Multiply，而不是Add或者FooBar呢？在本地调用中，函数体是直接通过函数指针来指定的，我们调用Multiply，编译器就自动帮我们调用它相应的函数指针。但是在远程调用中，函数指针是不行的，因为两个进程的地址空间是完全不一样的。所以，在RPC中，所有的函数都必须有自己的一个ID。这个ID在所有进程中都是唯一确定的。客户端在做远程过程调用时，必须附上这个ID。然后我们还需要在客户端和服务端分别维护一个 {函数 <--> Call ID} 的对应表。两者的表不一定需要完全相同，但相同的函数对应的Call ID必须相同。当客户端需要进行远程调用时，它就查一下这个表，找出相应的Call ID，然后把它传给服务端，服务端也通过查表，来确定客户端需要调用的函数，然后执行相应函数的代码。
2. **[序列化和反序列化](https://zhida.zhihu.com/search?content_id=70367603&content_type=Answer&match_order=1&q=序列化和反序列化&zhida_source=entity)**。客户端怎么把参数值传给远程的函数呢？在本地调用中，我们只需要把参数压到栈里，然后让函数自己去栈里读就行。但是在远程过程调用时，客户端跟服务端是不同的进程，不能通过内存来传递参数。甚至有时候客户端和服务端使用的都不是同一种语言（比如服务端用C++，客户端用Java或者Python）。这时候就需要客户端把参数先转成一个字节流，传给服务端后，再把字节流转成自己能读取的格式。这个过程叫序列化和反序列化。同理，从服务端返回的值也需要序列化反序列化的过程。
3. **网络传输**。远程调用往往用在网络上，客户端和服务端是通过网络连接的。所有的数据都需要通过网络传输，因此就需要有一个网络传输层。网络传输层需要把Call ID和序列化后的参数字节流传给服务端，然后再把序列化后的调用结果传回客户端。只要能完成这两者的，都可以作为传输层使用。因此，它所使用的协议其实是不限的，能完成传输就行。尽管大部分[RPC框架](https://zhida.zhihu.com/search?content_id=70367603&content_type=Answer&match_order=1&q=RPC框架&zhida_source=entity)都使用TCP协议，但其实UDP也可以，而[gRPC](https://zhida.zhihu.com/search?content_id=70367603&content_type=Answer&match_order=1&q=gRPC&zhida_source=entity)干脆就用了HTTP2。Java的[Netty](https://zhida.zhihu.com/search?content_id=70367603&content_type=Answer&match_order=1&q=Netty&zhida_source=entity)也属于这层的东西。

#### 解释

直接用Socket/TCP实现消息收发，并通过回调函数处理消息，这本质上是**手写了一个简易RPC**

我的实现中，网络层通过回调函数机制解耦了Raft核心逻辑和消息传输。虽然没有用gRPC这类标准RPC框架，但底层仍然通过Socket或自定义协议实现了节点间的请求-响应通信，本质上遵循了RPC的模式。这种设计更轻量，适合特定场景。

Raft的核心通信模式是固定的（如`AppendEntries`、`RequestVote`），因此我直接用Socket封装了消息收发，并通过回调函数将网络事件传递给Raft状态机。这样减少了对外部库的依赖，更适合我们的性能需求。

Dubbo：一个基于Java的高性能RPC框架，支持多种协议和注册中心，提供服务治理和监控等功能。gRPC：一个基于HTTP/2和protobuf的跨语言RPC框架，支持多种编程语言和平台，提供双向流、拦截器、认证等功能。Thrift：一个由Facebook开源的跨语言RPC框架，支持多种协议和传输方式，提供IDL生成器和编译器等工具。其他还有ICE、Hessian、RMI等。