# 第二次作业

## 一、Mesos组成结构、在源码中具体位置与工作流程

### 1、Mesos组成结构
![image](https://github.com/draelcba/test/raw/master/2/Mesos框架图.png)

Mesos由以下几部分组成：

* master：接收Mesos slave和Framework scheduler的注册，分配资源，master通过resource offer在框架间进行资源分配和资源共享。
* Zookeeper：选举出Mesos master，解决master可能出现的问题。
* Framework：包括Scheduler和Executor两部分。scheduler启动后注册到Master，决定是否接收Master发送来的offer；executor执行framework的Task。master决定配给framework多少资源，其scheduler选择用哪一个。framework接受offer后，将Task描述发送给Mesos，在agent上启动。整个Mesos系统是一个双层调度的框架：第一层由master将资源分配给框架；第二层由框架自己的调度器将资源分配给自己内部的任务。
* Mesos slave：接收Mesos master发来的Task，调度Framework executor去执行。
* Task：Task由Slave调度Exexutor执行。

### 2、在源码中的具体位置

* Framework：mesos-1.1.0/src/examples/test_framework.cpp的main函数中。指定Executor的uri，配置Executor的信息，创建Scheduler。
* Scheduler：mesos-1.1.0/src/scheduler文件夹。运行MesosSchedulerDriver的代码在mesos-1.1.0/src/sched/sched.cpp中。检测Leader，创建一个线程，注册消息处理函数，调用了Test Framework的resourceOffers函数，根据得到的offers，创建一系列Tasks，调用driver的launchTasks函数，向Leader发送launchTasks的消息。
* Executor：mesos-1.1.0/src/examples/test_executor.cpp。
* Zookeeper：mesos-1.1.0/src/zookeeper文件夹。
* Master：位于mesos-1.1.0/src/master文件夹。main.cpp是入口程序。
* Slave：位于mesos-1.1.0/src/slave文件夹。main.cpp是入口程序。


### 3、工作流程

* 集群中的所有Slave节点会和Master定期进行通信，将自己的资源信息同步到Master，Master由此获知到整个集群的资源状况。
* Master会和已注册、受信任的Framework进行交互，定期将最新的资源情况发送给Framework，当Framework前端有工作需求时，将选择接收资源，否则拒绝。
* 前端用户提交了一个工作需求给Framework。
* Framework接收Master发过来的资源信息。
* Framework依据资源信息向Slave发起任务启动命令，开始调度工作。



## 二、框架在Mesos上的运行过程，与传统操作系统的对比

### 1、运行过程

* Slave1向Master汇报其空闲资源。
* Master收到Slave发来的消息后，调用分配模块，发送一个描述Slave当前空闲资源的resource offer给Framework。
* Framework的调度器回复Master，需要运行的task，及需要的资源。
* Master把任务需求资源发送给Slave，Slave分配适当的资源给Framework的Executor，然后Executor开始执行任务。

### 2、与传统操作系统的对比

二者的差异性主要体现在资源分配方式上。Framework在Mesos上运行时，Master向Framework报告可用的资源，至于是否接收由Framework自己决定；而程序运行在传统操作系统上时，进程向内核申请资源，申请一般都会被满足。

## 三、master和slave的初始化过程

### 1、master

mesos-1.1.0/src/master/master.cpp对Master进行了初始化，主要是通过<code>initialize()</code>初始化函数。初始化之前，相关命令行解析等工作前文已经提到，下面仅分析初始化函数。

在<code>master::initialize()</code>初始化函数中：

* 在进行一系列权限认证、权值设置等操作后，初始化Allocator
```
// Initialize the allocator.
 allocator->initialize(
     flags.allocation_interval,
     defer(self(), &Master::offer, lambda::_1, lambda::_2),
     defer(self(), &Master::inverseOffer, lambda::_1, lambda::_2),
     weights,
     flags.fair_sharing_excluded_resource_names);
```
* 时钟开始计时
```
startTime = Clock::now();
```
* 注册消息处理函数，伪码如下（对原代码做了整理，对几个重要的消息处理函数添加了注释）：
```
install<SubmitSchedulerRequest>();
install<RegisterFrameworkMessage>(); //Framework注册
install<ReregisterFrameworkMessage>();
install<UnregisterFrameworkMessage>();
install<DeactivateFrameworkMessage>();
install<ResourceRequestMessage>(); //Slave发送来资源的要求
install<LaunchTasksMessage>(); //Framework发送来启动Task的消息
install<ReviveOffersMessage>(); 
install<KillTaskMessage>(); //Framework发送来终止Task的消息
install<StatusUpdateAcknowledgementMessage>(); 
install<FrameworkToExecutorMessage>();
install<RegisterSlaveMessage>(); //Slave注册
install<ReregisterSlaveMessage>();
install<UnregisterSlaveMessage>();
install<StatusUpdateMessage>(); //状态更新
install<ExecutorToFrameworkMessage>();
install<ReconcileTasksMessage>();
install<ExitedExecutorMessage>();
install<UpdateSlaveMessage>(); //Slave更新
install<AuthenticateMessage>();
```
* 设置http路由
* 开始竞争成为Leader，或者检测当前的Leader
```
 // Start contending to be a leading master and detecting the current
 // leader.
 contender->contend()
   .onAny(defer(self(), &Master::contended, lambda::_1));
 detector->detect()
   .onAny(defer(self(), &Master::detected, lambda::_1));
```

### 2、slave

mesos-1.1.0/src/slave/slave.cpp对Slave进行了初始化，主要是通过<code>initialize()</code>初始化函数。初始化之前，相关命令行解析等工作前文已经提到，下面仅分析初始化函数。

在<code>slave::initialize()</code>初始化函数中：

* 与<code>master::initialize()</code>类似，在完成权限认证等一系列预备工作后，初始化资源预估器
```
Try<Nothing> initialize =
  resourceEstimator->initialize(defer(self(), &Self::usage));
```
* 确认slave工作目录存在
```
// Ensure slave work directory exists.
CHECK_SOME(os::mkdir(flags.work_dir))
  << "Failed to create agent work directory '" << flags.work_dir << "'";
```
* 确认磁盘可达
* 初始化attributes
```
Attributes attributes;
  if (flags.attributes.isSome()) {
    attributes = Attributes::parse(flags.attributes.get());
  }
```
* 初始化hostname
* 初始化statusUpdateManager
```
statusUpdateManager->initialize(defer(self(), &Slave::forward, lambda::_1)
    .operator std::function<void(StatusUpdate)>());
```
* 注册消息处理函数，伪码如下（对原代码做了整理，对几个重要的消息处理函数添加了注释）：
```
install<SlaveRegisteredMessage>(); //Slave注册成功的消息
install<SlaveReregisteredMessage>();
install<RunTaskMessage>(); //运行一个Task的消息
install<RunTaskGroupMessage>();
install<KillTaskMessage>(); //停止运行一个Task的消息
install<ShutdownExecutorMessage>();
install<ShutdownFrameworkMessage>();
install<FrameworkToExecutorMessage>();
install<UpdateFrameworkMessage>();
install<CheckpointResourcesMessage>();
install<StatusUpdateAcknowledgementMessage>();
install<RegisterExecutorMessage>(); //注册一个Executor的消息
install<ReregisterExecutorMessage>();
install<StatusUpdateMessage>(); //状态更新消息
install<ExecutorToFrameworkMessage>();
install<ShutdownMessage>();
install<PingSlaveMessage>();
```


## 四、Mesos的资源调度算法

### 1、DRF算法
* Mesos默认的资源调度算法是DRF（主导资源公平算法 Dominant Resource Fairness），它是一种支持多资源的最大-最小公平分配机制。类似网络拥塞时带宽的分配，在公平的基础上，尽可能满足更大的需求。但Mesos更为复杂一些，因为有主导资源（支配性资源）的存在。比如假设系统中有9个CPU，18GB RAM，A用户请求的资源为（1 CPU, 4 GB），B用户请求的资源为（3 CPU， 1 GB），那么A的支配性资源为内存（CPU占比1/9，内存占比4/18），B的支配性资源为CPU（CPU占比3/9，内存占比1/18）。
* DRF算法的目标是使每个用户获得相同比例的支配性资源，给A用户（3 CPU，12 GB），B用户（6 CPU，2 GB），这样A获得了2/3的内存资源，B获得了2/3的CPU资源。
* DRF鼓励用户去共享资源，如果资源是在用户之间被平均地切分，会保证没有用户会拿到更多资源。
* DRF是strategy-proof，即用户不能通过欺骗来获取更多地资源分配。
* DRF是envy-free（非嫉妒）的，没有一个用户会与其他用户交换资源分配。
* DRF分配是Pareto efficient，即不可能通过减少一个用户的资源分配来提升另一个用户的资源分配。
### 2、在源码中的位置
DRF算法的源码位于mesos-1.1.0/src/master/allocator/sorter/drf文件夹中，其中的sorter.cpp用来对framework进行排序、add、remove、update等操作。mesos-1.1.0/src/master/allocator/mesos/hierarchical.cpp文件是分层分配器，它调用了sorter.cpp和sorter.hpp进行功能上的具体实现。



## 五、设计一个能完成简单工作的框架












