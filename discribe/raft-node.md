# raft-node

#### Raft-node.cpp

1. raftnode：根据配置文件，获取本节点id（port%10）

2. run/stop：启动网络通信networkmanager-start；启动日志应用线程logaplyloop；启动raftcore-start；

3. init component：启动networkmanager，logstore，kvstore，raftcore，设置三个回调

4. handleClientRequest：RedisProtocol::parseCommand（）将resp协议格式解析为commad格式，并进入下一个函数 `*2\r\n$3\r\nfoo\r\n$3\r\nbar\r\n`--'foo bar'

5. handleRespCommand:处理commad格式的指令.

   只有leader真正对commad进行处理：

   首先，将该条指令连同当前任期加入logstore中。

   对于del指令，等待前面的指令都commit，并统计删除次数，

   等待所有指令被提交

   发回相应

6. applycommand：get，不改变；set，kv_store_->set(key, value);del，kv_store_->del(parsed[i]);

7. logapplierloop：对于last-applied到commit-index之间的logstore，调用applycommand，并更新last-applied

**解耦设计**：Raft 不关心消息如何发送，由用户通过回调注入网络逻辑。

假设我们有以下调用代码：

```cpp
void sendData(int data, std::function<void(int)> processResponse) {
    int response = networkSend(data); // 1. 发送数据到网络
    processResponse(response);       // 2. 调用回调函数处理响应
}
```



```cpp
int main() {
    // 调用 sendData，并定义回调函数
    sendData(42, [](int result) {
        std::cout << "收到响应: " << result << std::endl;
    });

    // 后续代码...
    return 0;
}
```

**执行流程**

1. **`main` 函数** 调用 `sendData(42, callback)`，其中：
   - `data = 42`。
   - 回调是一个Lambda：`[](int result) { std::cout << "收到响应: " << result; }`。
2. **`sendData` 内部**：
   - 调用 `networkSend(42)` 发送数据，假设返回响应 `200`。
   - 调用 `processResponse(200)`，即执行Lambda，输出：
     `收到响应: 200`。
3. **回到 `main` 函数**，继续执行后续代码。

