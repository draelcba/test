# 第二次作业

## 一、Mesos组成结构、在源码中具体位置与工作流程

### 1、Mesos组成结构
![image](http://github.com/draelcba/test/raw/master/2/Mesos框架图.png)

如上图所示，Mesos主要组件有：

* Zookeeper：选举出Mesos master。
* Mesos master：接收Mesos slave和Framework scheduler的注册，分配资源。
* Standby master：作为备用Master，与Master节点运行在同一集群中。在Leader宕机后Zookeeper可以很快地从其中选举出新的Leader，恢复状态。
* Mesos slave：接收Mesos master发来的Task，调度Framework executor去执行。
* Framework：例如Spark，Hadoop等，包括Scheduler和Executor两部分。Scheduler启动后注册到Master，决定是否接收Master发送来的Resource offer消息，并反馈给Master。Executor由Slave调用，执行Framework的Task。
* Task：Task由Slave调度Exexutor执行，可以是长生命周期的，也可以是短生命周期的。

## 二、框架在Mesos上的运行过程，与传统操作系统的对比

## 三、master和slave的初始化过程

## 四、Mesos的资源调度算法

## 五、设计一个能完成简单工作的框架
