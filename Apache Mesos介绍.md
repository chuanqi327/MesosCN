#Apache Mesos简介
Apahce Mesos将CPU，内存，存储以及其他计算资源从机器中抽象出来（机器可以是物理机也可以是虚拟机）
轻松地建立并有效得运行具有容错机制并且可以弹性扩张的分布式系统。

##Mesos是什么？
###一个分布式系统内核
Mesos通过Linux内核的规则创建，只是在不同的层次上进行抽象。
Mesos内核将运行在这个分布式系统的每一台机器中，通过API向应用（如Hadoop的，Spark，Kafka，Elastic Search）提供跨数据中心和云环境的资源管理和调度服务。

##产品特性：
- 轻松扩展至万级节点。
- 通过Zookeeper来进行master与slave的容错
- 支持Docker容器
- 通过Linux容器与任务原生隔离
- 多重资源调度（内存，CPU，硬盘，端口）
- 通过Java，Python与C++的API开发新的并行应用
- 具有Web UI进行可视化操作


