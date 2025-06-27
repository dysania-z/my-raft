# raft-核心

## 1.解决了什么问题？或者说，为什么raft解决，其他技术解决呢？

###### **分布式一致性问题**

https://zhuanlan.zhihu.com/p/130332285

ACID中的C (Consistency): 指数据库事务的一致性，确保事务将数据库从一个有效状态转变为另一个有效状态，遵守所有定义的规则（如约束、级联等）。
CAP中的**C (Consistency):** 指线性一致性(Linearizability)，即所有节点在同一时间看到相同的数据，所有操作都有全局顺序。即用户在不同的系统节点访问数据的时候应该是同样的结果，不能出现1号节点是结果1， 2号节点是结果2这种不一致的情况。

**A availability 可用性**
分布式系统为用户提供服务，需要保证能够在一些节点异常的情况下仍然支持为用户提供服务。

**P partition tolerance 分区容错性**
分布式系统的部署可能跨省，甚至跨国。不同省份或者国家之间的服务器节点是通过网络进行连接，此时如果两个地域之间的网络连接断开，整个分布式系统的体现就是分区容错性了。如果为了在出现网络分区之后提供仍然能够提供可用性，那么一致性必然无法满足。**所以CAP理论一定是无法全部满足三者，只能满足其中的两者。**

分布式事务一致性 → 协调者单点（2PC） → 可用性问题 （单点故障）→ 需要共识 → 共识≡线性存储≡全序广播 → Paxos/Raft实现全序广播

**线性一致性（Linearizability）**要求系统表现得像所有操作按某种全局顺序原子执行，且顺序与实际时间一致（强一致性）。

- 每次"写"操作（提案值）需要通过共识协议达成一致。
- 每次"读"操作必须返回最新达成共识的值。

**全序广播（Total Order Broadcast）**要求所有节点以**完全相同的顺序**接收并处理所有消息。

- 每条消息作为一个提案，通过共识协议确定其全局顺序（如Raft的日志索引）。
- 所有节点按共识决定的顺序交付消息。

强一致性：paxos，raft（etcd分布式kv数据库），zab

弱一致性（最终一致性）：DNS系统，gossip协议

###### paxos：

Proposal提案，即分布式系统的修改请求，可以表示为**[提案编号N，提案内容value]**

Client用户，类似社会民众，负责提出建议

Propser议员，类似基层人大代表，负责帮Client上交提案

Acceptor投票者，类似全国人大代表，负责为提案投票，**不同意比自己以前接收过的提案编号要小的提案，其他提案都同意**，例如A以前给N号提案表决过，那么再收到小于等于N号的提案时就直接拒绝了

Learner提案接受者，类似记录被通过提案的记录员，负责记录提案

- Basic Paxos算法步骤

1. Propser准备一个N号提案
2. Propser询问Acceptor中的多数派是否接收过N号的提案，如果都没有进入下一步，否则本提案不被考虑
3. Acceptor开始表决，Acceptor**无条件同意**从未接收过的N号提案，达到多数派同意后，进入下一步
4. Learner记录提案

![在这里插入图片描述](https://i-blog.csdnimg.cn/blog_migrate/55cc316aa20b766cdde57a0b7637de1e.png#pic_center)

客户端发起request请求到Proposer，然后Proposer发出提案并携带提案的编号1，分别发给每一个Acceptor；每一个Acceptor根据天都编号是否满足大于1，将投票结果通过Propose阶段反馈给Proposer；Proposer收到Acceptor的结果判断是否达到了多数派，达到了多数派则向Acceptor发出Accept请求，并携带提案的具体内容V；Acceptor收到提案内容后向Proposer反馈Accepted表示收到，并将提案内容交给Learner进行备份。

Learner写入提案内容之后向Client发送完成请求的响应。

- 节点故障 
  - 若Proposer故障，没关系，再从集群中选出Proposer即可
  - 若Acceptor故障，表决时能达到多数派也没问题
- 潜在问题-**活锁**
  - 假设系统有多个Proposer，他们不断向Acceptor发出提案，还没等到上一个提案达到多数派下一个提案又来了，就会导致Acceptor放弃当前提案转向处理下一个提案，于是所有提案都别想通过了。

![截屏2025-06-12 21.48.28](/Users/racy/Library/Application Support/typora-user-images/截屏2025-06-12 21.48.28.png)解决：proposer1的上线之后 重新提交法案使用随机时间机制，即随机生成一个时间戳，在这段时间内不向Acceptor发送消息；这样proposer2的提案能够被处理完成，这个时候proposer1再次提交新的提案。

- **难实现**
  有太多的角色需要实现
- **效率低(2轮 rpc)**
  每一个请求需要proposer和acceptor 至少交互两次才能完成请求的处理

###### raft

Paxos算法不容易实现，Raft算法是对Paxos算法的简化和改进

Multi Paxos算法

- 根据Basic Paxos的改进：整个系统**只有一个**Proposer，称之为Leader。
- Multi Paxos角色过多，对于计算机集群而言，可以将Proposer、Acceptor和Learner三者身份**集中在一个节点上**，此时只需要从集群中选出Proposer，其他节点都是Acceptor和Learner，这就是接下来要讨论的Raft算法

<img src="/Users/racy/Library/Application Support/typora-user-images/image-20250612212259845.png" alt="image-20250612212259845" style="zoom:40%;" />

**同步主从**：接受请求，复制到所有log，返回所有成功信息，应用到状态机

**同步多数**：返回多数成功信息即可应用到状态机----》并发请求导致顺序不一致：在满足多数派的情况下可能会出现顺序的问题，不同的节点因为请求同步的时间问题可能出现请求的顺序不一致。

###### ZAB

1. 对于Leader的任期，raft叫做term，而ZAB叫做epoch
2. 在状态复制的过程中，raft的心跳从Leader向Follower发送，而ZAB则相反。

###### 更强大的复制机

实现行存储和列存储

## 2.raft原理

#### **leader选举**

###### become candidate

```cpp
void RaftCore::followerLoop()
    {
        while (running_ && (state_ == NodeState::FOLLOWER))
        {
            // 获取随机选举超时时间
            int timeout = FOLLOWER_TIMEOUT_MS;
            // 等待超时时间
            std::this_thread::sleep_for(std::chrono::milliseconds(timeout));
            // 检查是否收到心跳
            if (!received_heartbeat_)
            {
                // 未收到心跳，转换为候选者状态
                becomeCandidate();
            }
            else
            {
                // 收到了心跳，重置标志
                received_heartbeat_ = false;
            }
        }
    }
```



###### Candidateloop：

超时未收到-》任期加一-〉成为candidate-》给自己投票-》给其他节点发送投票请求-〉未收到回复：等待超时变回follower

```cpp
void RaftCore::candidateLoop()
    {
        // 创建随机数生成器，生成选举超时时间
        std::random_device rd;
        std::mt19937 gen(rd());
        // 定义均匀整数分布，生成区间[a, b]内的随机数。
        std::uniform_int_distribution<> dis(ELECTION_TIMEOUT_MIN_MS, ELECTION_TIMEOUT_MAX_MS);

        while (running_ && state_ == NodeState::CANDIDATE)
        {
            // 设置已投票标志
            voted_ = true; // 已经给自己投票
            // 开始新的选举
            current_term_++; // 任期+1
            vote_count_ = 1; // 先给自己投一票
            // 向其他节点发送投票请求
            std::vector<int> peer_ids = getPeerNodeIds(); // 得到含且仅含peersID的一个vector数组
            for (int peer_id : peer_ids)
            {
                std::cout << "[RaftCore:] " << "向节点 " << peer_id << " 发送投票请求" << std::endl;
                sendRequestVote(peer_id);
            }
            // 等待随机的选举超时时间
            int timeout = dis(gen);
            std::this_thread::sleep_for(std::chrono::milliseconds(timeout));
            // 检查状态，还没变成leader的话：
            if (state_ == NodeState::CANDIDATE)
            {
                // 选举超时，回到Follower状态
                becomeFollower(current_term_);
                std::cout << "[RaftCore:] " << id_ << " 未获得多数票，变回follower" << std::endl;
            }
            // 重置投票标志
            voted_ = false;
        }
    }
```



###### candidate发出请求投票消息

消息中包含，当前任期，节点id，最后一条日志的索引和任期

```cpp
   void RaftCore::sendRequestVote(int target_id)
    {
        if (target_id == id_)
        {
            return; // 不向自己发送
        }
        // 创建RequestVote请求
        auto request = std::make_unique<RequestVoteRequest>();
        request->term = current_term_;
        request->candidate_id = id_;

        // 获取最后一条日志的索引和任期
        int last_log_index = log_store_->latest_index();
        int last_log_term = log_store_->latest_term();

        request->last_log_index = last_log_index;
        request->last_log_term = last_log_term;

        // 发送请求 RPC
        sendMessage(target_id, *request);
    }
```



###### 其他节点给出投票（处理投票请求）：

任何类型节点都会接受投票请求，先比较任期，若当前任期小于请求任期，则放弃自己当前身份初始化成为follower（voted=false），并更新任期

判断是否需要更新log：

1. **`request.last_log_term > log_store_->latest_term()`**
   - **Candidate 的最后一条日志的任期 > Follower 的最后一条日志的任期**
   - 说明 Candidate 的日志比 Follower 更新（因为 Raft 保证高 Term 的日志一定包含所有低 Term 的已提交日志）。
2. **`request.last_log_term == log_store_->latest_term() && request.last_log_index >= log_store_->latest_index()`**
   - **Candidate 的最后一条日志的任期 == Follower 的最后一条日志的任期**
     并且 **Candidate 的最后一条日志的索引 ≥ Follower 的最后一条日志的索引**
   - 说明 Candidate 的日志至少和 Follower 一样新（可能更长）。

只有同时满足当前未投票且需要更新日志时才会给对方投票

```cpp
response->term = current_term_;
        response->vote_granted = false;

        if (request.term < current_term_) {
            return response; // 任期太旧，不投票
        }

        if (request.term > current_term_) {
            current_term_ = request.term;
            becomeFollower(current_term_);
            voted_ = false;
        }

        bool log_up_to_date = 
            request.last_log_term > log_store_->latest_term() || 
            (request.last_log_term == log_store_->latest_term() && 
            request.last_log_index >= log_store_->latest_index());

        if (!voted_ && log_up_to_date) {
            voted_ = true;
            response->vote_granted = true;
            std::cout << "[RaftCore:] " << id_ << " 投票给了节点 " << request.candidate_id << std::endl;
        }
    
        return response;
```



###### candidate收到投票结果回复：handle RequestVote Response：

回复中任期更大：回退为同任期的follower，否则统计票数，超过半数时成为leader

```cpp
 void RaftCore::handleRequestVoteResponse(int from_node_id, const RequestVoteResponse &response)
    {
        // from_node_id参数未使用，但保留接口一致性
        //(void)from_node_id;
        std::cout << "[RaftCore:] " << id_ << " 收到来自节点 " << from_node_id << " 的投票响应" << std::endl;
        // 只有在candidate状态才处理投票响应
        if (state_ != NodeState::CANDIDATE)
        {
            return;
        }
        // Job2:收到来自其他节点的投票回应，补全代码，做出对应的反应
        if (response.term > current_term_) {
        becomeFollower(response.term);
        return;
        }
				//
        if (response.vote_granted) {
            vote_count_++;
            std::cout << "[RaftCore:] " << id_ << " 获得来自节点 " << from_node_id << " 的选票，总票数：" << vote_count_ << std::endl;

            if (vote_count_ > cluster_size_ / 2) {
            becomeLeader();
            }
        }
    }
```

#### 日志复制

###### leaderloop

按时发送心跳，多次没收到回复之后就由网络连接错误回退到follower状态

```cpp
void RaftCore::leaderLoop()
    {
        while (running_ && state_ == NodeState::LEADER)
        {
            // 发送心跳
            std::vector<int> peer_ids = getPeerNodeIds();
            seq_ = seq_ == 10 ? 0 : seq_ + 1; // seq_是心跳序列号，每10次心跳后重置为0，每次while循环后加一
            for (int peer_id : peer_ids)
            {
                sendAppendEntries(peer_id, true); // sendAppendEntries(peer_id, bool isheartbeats);
            }

            // 等待心跳间隔
            std::this_thread::sleep_for(std::chrono::milliseconds(HEARTBEAT_INTERVAL_MS));

            // 检查Leader的存活计数
            // 初始值心跳周期内预期的最大存活响应数（如 cluster_size / 2 + 1）。
            // 收到回复会加一，不超过初始值
            // 每次心跳等待间隔会减一
            live_count_--;
            if (live_count_ < 0)
            {
                // 长时间未收到大多数节点的响应，怀疑网络分区，退回到Follower状态
                becomeFollower(current_term_);
                std::cout << "[RaftCore:] " << id_ << " 出现网络链接问题，退回到follower" << std::endl;
                break;
            }
        }
    }
```

###### leader发出send append entries（心跳or附带信息）

创建AppendEntries请求：任期➕leaderID➕心跳seq➕prev-log-index（follower）➕prev-log-term（leader）➕leader-commit➕entries

调用sendMessage(target_id, *request)发出心跳



```cpp
void RaftCore::sendAppendEntries(int target_id, bool is_heartbeat)
    {
        if (target_id == id_)
        {
            return; // 不向自己发送
        }

        // 创建AppendEntries请求：任期➕leaderID➕心跳seq➕prev-log-index（follower）➕prev-log-term（leader）➕leader-commit➕entries
  
        auto request = std::make_unique<AppendEntriesRequest>();
        request->term = current_term_;
        request->leader_id = id_;
        request->seq = seq_;

 
        // 获取leader认为的，目前目标节点的最高日志索引（维护在match-index中）
        int prev_log_index = 0;
        int prev_log_term = 0;
        {
            std::lock_guard<std::mutex> lock(match_mutex_);

            int idx = nodeIdToIndex(target_id);
            if (idx >= 0 && idx < static_cast<int>(match_index_.size()))
            {
                prev_log_index = match_index_[idx];
            }
            else
            {
                std::cout << "Sending message to node " << target_id << std::endl;
                std::cerr << "Unknown target node ID: " << target_id << std::endl;
                return;
            }
        }

        // 获取leader在此处（prev-log-index）的term
        if (prev_log_index > 0)
        {
            prev_log_term = log_store_->term_at(prev_log_index);
        }
  
  
        request->prev_log_index = prev_log_index;
        request->prev_log_term = prev_log_term;
  
  
        // 类似一个全局变量，每次在处理follower回复附加消息时，leader根据多数节点的`match_index`计算
        request->leader_commit = commit_index_;

        // 如果是心跳，则不附加日志条目，每次帮忙追赶进度
        if (is_heartbeat)
        {
            // 添加目标节点日志条目
            int next_index = prev_log_index + 1;
            // leader的最后index
            int last_index = log_store_->latest_index();

            for (int i = next_index; i <= last_index && i <= next_index + BATCH_SIZE; ++i)
            {
                // message.h 中定义的struct
                LogEntry entry;
              //附加上目前落后的日志data及其对应的索引
                entry.term = log_store_->term_at(i);
                entry.data = log_store_->entry_at(i);
                request->entries.push_back(entry);
            }
        }

        // 发送请求
        sendMessage(target_id, *request);
    }
```

###### 其他节点收到来自leader节点的日志同步请求（心跳）

**handleAppendEntries**：term+success(bool)

仍是先比较任期，若leader任期小，直接返回response(带有当前term和失败：

```cpp
 response->term = current_term_;
 response->success = false;
```

若leader任期大，先把不是follower的节点退回follower（request.term），对于每一个节点，设置收到心跳，确定leaderid，response-succcess，但还不返回response。继续进行日志匹配检查

```cpp
if (request.prev_log_index > 0) {
            if (log_store_->latest_index() < request.prev_log_index ||
                log_store_->term_at(request.prev_log_index) != request.prev_log_term) {
                response->success = false;
                return response;
            }
        }
```

1. **`log_store_->latest_index() < request.prev_log_index`**
   - **Follower 的日志比 Leader 认为的 `prev_log_index` 短**，说明 Follower 缺失部分日志。
   - 例如：
     - Leader 认为 Follower 的最后一条日志是 `index=5`，但 Follower 只有 `index=3`。
     - 此时 Follower 无法接受新日志，因为 `index=4` 和 `5` 的日志缺失。
2. **`log_store_->term_at(request.prev_log_index) != request.prev_log_term`**
   - **Follower 在 `prev_log_index` 处的日志任期与 Leader 认为的不一致**，说明日志冲突。
   - 例如：
     - Leader 认为 `index=3` 的日志 `Term=2`，但 Follower 的 `index=3` 的日志 `Term=3`。
     - 说明 Follower 的日志和 Leader 不一致（可能是之前网络分区导致的日志分歧）。

如果以上任意条件成立，则 **`success = false`**，表示 **日志不匹配，拒绝追加请求**。

```cpp
// 若匹配，且存在，追加新日志（如果有）
        if (!request.entries.empty()) {
            for (const auto& entry : request.entries) {
                log_store_->append(entry.data, entry.term);
            }
        }

        // 更新提交索引
        if (request.leader_commit > commit_index_) {
            commit_index_ = std::min(request.leader_commit, log_store_->latest_index());
        }
        response->success = true;
```

所以整个的流程就是：

任期小直接返回response-false---信息全当作心跳处理-----检查日志匹配情况，不匹配则返回false----存在且匹配，追加-----更新commit值-----返回true

```cpp
    // Job3:收到来自leader节点的日志同步请求（心跳），补全代码，构造正确的回应消息
        response->term = current_term_;
        response->success = false;

        if (request.term < current_term_) {
            return response; // leader任期太小，拒绝
        }

        if (request.term >= current_term_) {
            if (request.term > current_term_ || state_ != NodeState::FOLLOWER) {
                current_term_ = request.term;
                becomeFollower(current_term_);
            }
            received_heartbeat_ = true;
            leader_id_ = request.leader_id;
            response->success = true;
        }
        // 检查前一条日志是否匹配（日志的lastindex就是leader assume 的那条，并且任期也相同
        if (request.prev_log_index > 0) {//要追加的不是第一条日志
            if (log_store_->latest_index() < request.prev_log_index ||
                log_store_->term_at(request.prev_log_index) != request.prev_log_term) {
                response->success = false;
                return response;
            }
        }

        // 若匹配，且存在，追加新日志（如果有）
        if (!request.entries.empty()) {
            for (const auto& entry : request.entries) {
                log_store_->append(entry.data, entry.term);
            }
        }

        // 更新提交索引
        if (request.leader_commit > commit_index_) {
            commit_index_ = std::min(request.leader_commit, log_store_->latest_index());
        }
        response->success = true;
        return response;
```



###### leader收到对日志同步的回复



leader收到来自follower节点的日志同步回应：**handle  AppendEntries  Response**

```cpp
//不是leader，return；任期小于回复任期，变为follower，return
if (state_ != NodeState::LEADER) {
            return;
        }
if (response.term > current_term_) {
     becomeFollower(response.term);
     return;
        }
```

具有处理回复资格时：

获得match-mutex锁，对match-index和match-term进行修改：

若response.success，（此时follower已经进行了一次日志同步，从next index到min（leader last index，next index ➕ batchsize））

则将leader维护的match-index更新至log_store_->latest_index()，term更新为当前term

-  Raft 的实现通常 **乐观更新** `match_index`，因为：
  1. **下一次 RPC 会检查 `prev_log_index`**，如果 follower 缺少某些日志，它会拒绝，并 leader 会回退 `next_index`。
  2. 即使 `match_index` 被乐观更新，在第三阶段有界限判断，所以不会提交未复制的日志。

**日志提交（Commit）**

日志被提交是指该日志已被**持久化到多数节点**的日志存储（`LogStore`）

- **触发条件**：
  - Leader 收到多数节点（包括自己）对某条日志的复制确认（即 `match_index` 达到多数）。
  - Leader 确保该日志的任期（`term`）是当前任期（避免提交旧任期的日志，防止图8问题，见Raft论文5.4.2）。

**日志应用（Apply）**

将已提交的日志**应用到状态机**（如KV存储），执行日志中的命令（如 `put x=1`），修改实际数据。

- **触发条件**：
  - 本地节点的 `commit_index` 更新后，且存在 `last_applied < commit_index`（有未应用的日志）。
    - 通常由**单独的应用线程**或主循环异步执行（避免阻塞核心Raft逻辑）。

```cpp
        //维护两个数组，match_index是每个节点最高的日志索引
        //match_term:每个节点最高的日志任期
        std::lock_guard<std::mutex> lock(match_mutex_);
        int idx = nodeIdToIndex(from_node_id);
        if (idx < 0) return;

        if (response.success) {
            match_index_[idx] = log_store_->latest_index();
            match_term_[idx] = current_term_;
            live_count_ = LEADER_RESILIENCE_COUNT;

            // 尝试更新 自身（leader）的commit_index_（更新大了也没事，在第三阶段有界限判断
            std::vector<int> match_indexes = match_index_;
            match_indexes.push_back(log_store_->latest_index()); // 加上 leader 自己
            std::sort(match_indexes.begin(), match_indexes.end());
            int majority_match = match_indexes[cluster_size_ / 2];

            if (majority_match > commit_index_ && 
                log_store_->term_at(majority_match) == current_term_) {
                commit_index_ = majority_match;
            }
        } 
```

如果response.success=false（日志不匹配

将自己认为的，follower的最高索引回退一格

```cpp
else {
            // follower 同步失败，回退 match_index_
            if (match_index_[idx] > 0) {
                match_index_[idx]--;
            }
        }
```

###### 日志提交

**Raft-node：handleRespCommand**

只有leader真正对commad进行处理：

首先，将该条指令连同当前任期加入logstore中。

对于del指令，等待前面的指令都commit，并统计删除次数，

> [!NOTE]
>
> 其他几种指令呢？为什么无需这么做？

等待所有指令被提交

发回响应（commit就响应，而非apply才响应

```cpp
std::string cmd_type = command[0];//SET GET DELETE
std::transform(cmd_type.begin(), cmd_type.end(), cmd_type.begin(), ::toupper);

// 对于所有操作，添加到日志
int current_term = raft_core_->getCurrentTerm();
int log_index = raft_core_->appendLogEntry(original_request, current_term);
int commit_index = raft_core_->getCommitIndex();

//对于del执行前检查：当执行 DEL 命令时，必须确保该命令依赖的 前一条日志（log_index - 1）已经被提交（committed），否则等待直到满足条件。
if (cmd_type == "DEL")
{

    while (log_index - 1 > commit_index)
    {
        // std::cout<<"[RaftNode:] " <<current_term << " "<< commit_index << std::endl;
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
        commit_index = raft_core_->getCommitIndex();
    }
}
int del_count = 0; // 统计需要删除的数目：不能等del应用到状态机再在统计，因为del删除完后再去统计，会导致del_count为0
if (cmd_type == "DEL")
{
    if (command.size() >= 2)
    {
        for (size_t i = 1; i < command.size(); ++i)
        {
            if (!kv_store_->get(command[i]).empty())
            {
                del_count++;
            }
        }
    }
}
// 等待日志被提交--》调用log Applier loop---》调用Applycommand进行执行
while (log_index > commit_index)
{
    // std::cout<<"[RaftNode:] " <<current_term << " "<< commit_index << std::endl;
    std::this_thread::sleep_for(std::chrono::milliseconds(10));
    commit_index = raft_core_->getCommitIndex();
}



// 根据命令类型生成响应//get key，return value or nil or error
if (cmd_type == "GET")
{
    if (command.size() >= 2)
    {
        std::string value = kv_store_->get(command[1]);
        if (value.empty())
        {
            // 返回nil值
            return "*1\r\n$3\r\nnil\r\n";
        }
        else
        {
            // 使用专门的GET响应编码方法
            // td::cout<<"[RaftNode:] " << value << std::endl;
            return RedisProtocol::encodeGetResponse(value);
        }
    }
    return RedisProtocol::encodeError("Wrong number of arguments for GET command");
}
else if (cmd_type == "SET")//return ok or error
{
    if (command.size() >= 3)
    {
        return RedisProtocol::encodeStatus("OK");
    }
    return RedisProtocol::encodeError("Wrong number of arguments for SET command");
}
else if (cmd_type == "DEL")//return del-count or error 
{
    if (command.size() >= 2)
    {
        return RedisProtocol::encodeInteger(del_count);
    }
    return RedisProtocol::encodeError("Wrong number of arguments for DEL command");
}
else
{
    return RedisProtocol::encodeError("Unknown command: " + command[0]);
}
```

等待日志被提交--》调用log Applier loop---》调用Applycommand进行执行

#### leader重定向

收到客户端请求时**RaftNode::handleRespCommand**

当有效的follower节点收到请求时，回复MOVED leader-id进行重定向

```cpp
 if (state == NodeState::CANDIDATE)
{
    // 候选者状态，拒绝客户端请求
    return "+TRYAGAIN\r\n";
}
else if (state == NodeState::FOLLOWER)
{
    // 跟随者状态，重定向到Leader
    if (leader_id != 0)
    {
        return "+MOVED " + std::to_string(leader_id) + "\r\n";
    }
    else
    {
        return "+TRYAGAIN\r\n";
    }
}
else if (state == NodeState::LEADER)
```

用户会得到 leader-id，再次发起请求

```shell
if [[ $resp =~ $moved_regex ]]; then
            leader_id=${BASH_REMATCH[1]}
            leader_idx=${leader_map[$leader_id]}
            echo -e "${RED}【tester】：【重定向】收到MOVED响应，切换到leader ${leader_id} (${follower_ip[$leader_idx]}:${follower_port[$leader_idx]})${RESET}"
            LEADER_IP=${follower_ip[$leader_idx]}
            LEADER_PORT=${follower_port[$leader_idx]}
            ((try_count++))
            sleep $retry_interval
            continue
        fi
```

#### 成员管理

我实现的版本目前聚焦于**基础故障处理**（如 Leader 选举、日志复制和节点崩溃恢复）。动态成员变更（如运行时增删节点）是 Raft 的进阶功能，需要处理配置切换的复杂性（如 `Joint Consensus` 或单步变更）。
我对其设计原理有深入研究（可举例说明），但出于项目阶段考虑，优先保证了核心功能的稳定性。

**脑裂问题：**出现两个leader。

**联合一致状态：**阶段一：oldnew，rpc在新旧配置中都需要达到大多数；阶段二：cnew，在新配置中达到大多数。总是使用最新的日志，不论是否提交

#### 强一致性

###### 共识算法：

所以 Raft 的核心就是 leader 发出日志同步请求，follower 接收并同步日志，最终保证整个集群的日志一致性。按照相同顺序应用logentries，结束状态一定一致（相同的初始状态+相同的输入=相同的结束状态）

- **分布式 ≠ 所有节点都能随意读写**，而是通过多副本协作提供单机无法实现的 **容错性、扩展性和持久性**。
- **强一致读需走 Leader** 是一种设计权衡，但分布式仍意义重大：

###### 任期

每段任期从一次选举开始，一个任期内最多有一个leader，也可能没有leader（均为达到半数），此时会马上开始下一个任期的选举

每个entry都有任期号。如果不同 Leader 在同一索引位置提交了不同的日志（如脑裂后恢复），任期号用于确定哪条日志有效（选择任期号更大的日志）。

###### 日志号

标识entry的先后顺序。Raft 要求日志必须按顺序提交和应用，索引号确保所有节点对日志顺序达成一致。

###### 日志复制

对于没有回应的follower，不放弃不断重发appendentry，保证没有空洞；崩溃后恢复的follower，根据previndex和prevterm的匹配，不断向前定位直至找到正确的可恢复的entry，并按顺序恢复（失败不经常发生，没有必要优化一个个往前找的日志；leader故障，新leader不具备老leader少量未提交日志，认定老leader日志提交失败，还是使用一致性检查找到最后一个一样的日志并对后面的日志进行覆盖。这样leader只要不断appendentris就可以使日志趋向一致

###### 安全性（几种边界状态

**选举限制**

落后若干条目（但未落后一个任期）的节点可能当选leader，导致缺失的条目永远无法补上-----选举出来的leader一定要包含所有已提交条目-----不投给日志比自己老的节点

**新leader日志提交**

每次收到append entry回复时，尝试更新自己的commit-index，并在发送下一个entry时，作为leader-index发送，也就是说，节点在收到i+1的条目时或心跳时，commit-index最多就是i。实际上leadercommit后直接返回客户端ok了，但更严谨的做法是集群commit后再返回ok。

新leader一致性检查不断使日志趋向一致的过程中，会有老任期内的日志由不到半数到超过半数，但是新leader不能提交这些日志，为了防止提交后的覆盖产生不一致（如现在是任期4，提交了任期2的，然后故障了，任期4还没进行任何提交，然后拥有任期3日志的节点凭借最大任期变成新leader，用任期3覆盖了任期4）。只有自己任期内的日志超过半数时才认为可以提交，此时会把之前的老日志也一并提交。相当于用任期4的日志保护了任期2的日志。因为此时任期3的节点不会再当选leader了

**candidate和follower的故障**

不断重发appendentry RPC进行一个一致性检查即可。raft的rpc是**幂等的**。一个操作如果多次执行的效果与一次执行的效果相同，则称该操作是幂等的。

###### 时间

不依赖于客观时间，后发的rpc先到也不会影响正确性

**广播时间（得到选票时间）〈选举超时时间（10ms～500ms）〈平均故障时间**



## 3.kv存储&log存储（现有的轮子，自己的实现，可优化？）

###### **kv存储**

互斥锁mutex保护操作，利用unordered-map（哈希表➕链式表）实现kv存储，实现get，set，del

**(1) 哈希桶（Buckets）**

- **动态数组**：存储一组桶（每个桶是一个链表的头指针或迭代器）。
- **哈希函数**：将键（Key）映射到桶的索引（如 `hash(key) % bucket_count`）。

**(2) 解决冲突：链地址法（Separate Chaining）**

- 每个桶指向一个链表（或类似结构），存储哈希冲突的键值对。

- **示例**：

  ```
  // 简化的伪代码结构
  vector<forward_list<pair<Key, T>>> buckets;
  ```

  - 插入 `(key, value)` 时：
    1. 计算 `hash(key)` 得到桶索引。
    2. 在对应链表中查找是否已存在该键（若存在则更新，否则插入）。

###### log存储

 内存实现的日志存储，锁保护

