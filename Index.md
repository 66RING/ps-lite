# 源码分析

## 基本使用

> 知道怎么用后才有overview来看源码

### 原理

- server node
    * 维护和更新weight
- worker node
    * 通过pull和push和server通信: push计算好的梯度, pull模型
- scheduler
    * 监听结点存活情况, 发送控制信号和收集进度信息

> SGD: 随机梯度下降

```c
// Server
t = 0;
while (Received(&grad)) {
  // 更新权重
  // eta(t): t时刻的学习率
  w_k -= eta(t) * grad;
  t++;
}


// Worker
Read(&X, &Y);  // read a minibatch X and Y
Pull(&w);      // pull the recent weight from the servers
ComputeGrad(X, Y, w, &grad);  // compute the gradient
Push(grad);    // push the gradients to the servers
```

- 异步SGD: weight信息可能不是最新的
- 同步SGD: weight的新的

同步SGD通过scheduler同一控制

```c
// Scheduler
for (t = 0, t < num_iteration; ++t) {
  for (i = 0; i < num_worker; ++i) {
    // 通知worker计算梯度
    IssueComputeGrad(i, t);
  }
  for (i = 0; i < num_server; ++i) {
    // 通知Server更新权重
    IssueUpdateWeight(i, t);
  }
  WaitAllFinished();
}

// Server
ExecUpdateWeight(i, t) {
   // 接收所有worker信息再更新
   for (j = 0; j < num_workers; ++j) {
      Receive(&grad);
      aggregated_grad += grad;
   }
   w_i -= eta(t) * aggregated_grad;
}
```

### Usage

- kv的形式, Push, Pull, Wait
- Flow
    * TODO: 搞清楚基础设施怎么建立的
    * `Start(customer_id)`


## 实现

- Push -> ZPush -> Send

- ZPush
    * NewRequest
    * AddCallback
    * Send(kv)
- Send
    * `slicer_(kvs, range, sliced)`拆成小块?? TODO:
    * 构建一个Message结构:
        + 元数据: Server所在rank, 命令类型等
        + AddData添加kv数据, 保存到一个`vector<SArray<char>>`中
    * 通过`Postoffice::Get()->van()->Send(msg)`发送
- Van::Send: 以使用zmq通信为例
    * SendMsg
        + 找到Server对应的socket
        + Van::PackMeta: convert into protobuf
        + send meta
        + send data
- Pull(key, &value)
    * AddPullCB
        + NewRequest
        + AddCallback: 用于附值&value输出参数
            + 接收结果保存在类成员`recv_kvs_`中
    * Send: 同上
- Wait(ts) -> WaitRequest(ts): 使用条件变量等待
    * tracker_cond_.wait()
- `tracker_`的功能
    1. 是一个`vector<group_size, recv_num>`, 所以timestamp是是一个全局时钟(idx)
        - NewRequest: ts = `tracker_.push_back`后返回`size() - 1`
    2. 条件变量等待直到`group_size == recv_num`, 即`tracker_[timestamp].first == tracker_[timestamp].second`

- Client启动
    * TODO: Start -> Postoffice::Get()->Start()
- Client主循环
    * 入口: KVWorker对象构造后自动创建独立recv线程, 通过Customer包裹
- Server主循环
    * 入口: KVServer对象构造后自动创建独立recv线程, 通过Customer包裹
- Scheduler主循环

- Start & Finalize
    * Start
        1. 读取环境变量
            - 结点类型信息在读取环境变量时附值到`is_worker_`等成员
        2. 分组编号保存到map中
        3. van.start
            a. TODO
            b. 启动接收线程, 处理基础分布式的系统(心跳, 控制, 一般msg等)
    * Finalize

- Receiving和AddResponse的区别
    * AddResponse可以手动微调`tracker_[timestamp].second`
    * Receiving是单独的接收线程
        + `Customer::Receiving()`
        + `Van::Receiving()`

- TODO:
    * Customer抽象
        + 每个server和worker都有一个Customer类型的`_obj`成员用于通信
        + 可以自定义传入一个`recv_handle`, 然后使用独立线程做`Receiving()`, `Receiving()`中会调用`recv_handle()`
        + **Receiving**
            + van模块收到底层数据包后保存到`recv_queue_`中
            + customer根据条件变量通知获取`recv_queue_`中的数据(Message)
            + 使用用户定义的handler处理Message(Server, Worker各不同)
            + 对于response就通知tracker更新接收到的情况
    * Server抽象: KVServer
        + KVServer继承自SimpleApp
        + 绑定`KVServer<Val>::Process`
        + 绑定处理函数`KVServerDefaultHandle`
        + 
    * Client抽象: KVWorker
    * SArray
    * 再次搞清楚Send细节
    * 搞清楚分片等操作的细节
    * van和Customer的区别
        + van是真正负责底层通信的模块, 把网络包构建成Message对象装入`recv_quque_`中
        + Customer是于app对接的通信模块, 使用条件变量通知来访问`recv_quque_`


- 使用timestamp作为uuid??
- Customer
    * NewRequest
        + `tracker_`: 

- SimpleApp

- TODO
    * sarray











