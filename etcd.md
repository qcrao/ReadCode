# Etcd源码分析

看了比较久的etcd源码了，开始做总结了。这个系列主要分析的是其中的一个etcd的example，代码分支为：release-3.1, 路径为：contrib/raftexample. 

## 主流程启动
![contrib/raftexample/main.go:24](https://user-images.githubusercontent.com/7698088/43619835-4604931c-9702-11e8-9924-64a870848800.png)

L2-6解析启动时传入的命令行参数，cluster为节点间的通信地址，id为节点id, kvport为对外提供服务的端口，join为false时表示，这是一个新建的集群。

源码中给出的一个常见启动样例如下：
![carbon 3](https://user-images.githubusercontent.com/7698088/43619775-04474672-9702-11e8-83c4-527b0b17009f.png)

在本地执行这三条命令，启动了3个etcd服务进程，通过端口来区分它们。
