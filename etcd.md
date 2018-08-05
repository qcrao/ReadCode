Etcd源码分析

这段时间看了比较久的etcd源码，开始做总结。这个系列主要分析的是其中的一个etcd的example，代码分支为：release-3.1, 路径为：contrib/raftexample. 

# 主流程启动
![contrib/raftexample/main.go:24](https://user-images.githubusercontent.com/7698088/43619835-4604931c-9702-11e8-9924-64a870848800.png)

L2-6解析启动时传入的命令行参数，cluster为节点间的通信地址，id为节点id, kvport为对外提供服务的端口，join为false时表示，这是一个新建的集群。

L8新建了一个proposeC, L9新建了一个confChangeC, 前者是kvstore结构体中的一个channel, 后者是httpHandler的一个channel. 在启动http server之后，当收到`Put`请求时，向proposeC中发送k-v消息；当收到`POST`请求时，会向confChangeC中发送增加节点的消息，而当收到`DELETE`请求时，会向confChangeC中发送删除节点的消息。

L15定义获取Snapshot的方法，L16新建了一个RaftNode, 返回一个已提交信息的channel, 一个错误信息的channel. 两者都是非阻塞的channel. 以及一个容量为1的快照channel. RaftNode会消费confChangeC及proposeC里的消息，并作出相应的反应。

L18新建了kvstore, 它向proposeC中发送消息，从commitC中消费消息。

L21会启动http服务。处理用户的各种请求，包括set/get value, 及集群节点的变更请求等。

源码中给出的一个启动样例如下：
![carbon 3](https://user-images.githubusercontent.com/7698088/43619775-04474672-9702-11e8-83c4-527b0b17009f.png)

在本地执行这三条命令，启动了3个etcd服务进程，通过端口来区分它们。

最后，初步形成的整体架构如下图，后续会继续完善。

![image](https://user-images.githubusercontent.com/7698088/43684021-c94d325a-98ca-11e8-87b4-dae0f28b74f1.png)

# RaftNode启动

函数newRaftNode中，首先新建一个RaftNode, 将各种变量初始化好，然后调用startRaft()方法启动。我们将这个函数分成四部分来看。

![image](https://user-images.githubusercontent.com/7698088/43684555-ddd1e716-98d4-11e8-9e79-e74b18dd123f.png)

第一部分，L2-12 重放wal日志

根据snapshot文件及wal日志文件，将服务器状态恢复到上次停止时的状态。

第二部分，L14-39启动node，处理raft相关事务

创建node, 处理raft相关事务。

第三部分，L41-56 启动transport, 建立和其他节点的通信通道

主动去向其他peer节点发送http请求。

第四部分，L59-60启动监听协程

serveRaft()监听创建时的端口，用于和其他节点进行通信。第三部分节点会向其他节点发送请求。

serveChannels()会一直监听proposeC和confChangeC, 获取到消息后，调用node的Propose和ProposeConfChange方法进行处理。同时，启动一个定时器，每100ms触发一次node的tick. 当有日志追加的时候，会将日志追加到wal日志，同时发送nil到commit通道，触发kvstore将已经提交的日志应用到状态机中。

至此，etcd server已经启动完毕。