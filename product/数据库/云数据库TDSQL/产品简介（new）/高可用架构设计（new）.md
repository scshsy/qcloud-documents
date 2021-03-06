## 1.高可用架构
### 1.1 高可用架构基础
在生产系统中，通常都需要用高可用方案来保证系统不间断运行；数据库作为系统数据存储和服务的核心能力，其可用要求高于计算服务资源。目前，数据库的高可用方案通常是让多个数据库服务协同工作，当一台数据库故障，余下的立即顶替上去工作，这样就可以做到不中断服务或只中断很短时间；或者是让多台数据库同时提供服务，用户可以访问任意一台数据库，当其中一台数据库故障，立即更换访问另外数据库即可。
当然，由于数据库中记录了数据，想要在多台数据库中切换，需要进行数据同步，所以**数据同步技术是数据库高可用方案的基础**。

### 1.2 常见高可用架构介绍
- **共享存储方案**：使用共享存储，如SAN存储。SAN的原理是多台数据库服务器共享同一个存储区域，这样多台数据库都可以“读写”同一份数据。当主库发生故障时，第三方高可用软件把文件系统在备库上挂起，然后在备库上启动数据库即完成切换。
- **日志同步或流复制同步**：数据库最常见复制模式，如MySQL数据库。每当写入数据，MySQL Master Server将自己的Binary Log通过复制线程传输给Slave，Slave接收到Binary Log以后，依照Binary Log内容，写入相同数据到文件系统。目前MySQL已经提供：
  - 异步复制：异步复制可以确保得到快速的响应结构，但是不能确保二进制日志确实到达了slave上，即无法保障数据一致性；
  - 半同步复制：（由google提供的同步插件）半同步复制对于客户的请求响应稍微慢点，在超时等情况下，会退化为异步，即基本保障数据一致性，但无法保证数据完全一致性。
- **基于触发器的同步**：试用触发器记录数据变化，然后同步到另一台数据库上。
- **基于中间件的同步**：系统不直接连接到底层数据库，而是连接到一个中间件，中间件把数据库变更发送到底层多台数据库上，从而完成数据同步。早几年，由于业务需求，数据库性能、同步机制等问题，某些软件开发商通常采用类似架构。

## 2.TDSQL高可用架构简介
### 2.1	异步多线程强同步复制方案（强同步方案）
同步技术发展过程中，提供了异步复制、半同步等同步技术，这两种技术面向普通用户群体，在用户要求不高、网络条件较好、性能压力不大的情况下，能够基本保障数据同步；但通常情况下，采用异步复制、半同步机制容易经常出现数据不一致问题，直接影响系统可靠性，甚至出现丢失交易数据，带来直接或间接经济损失。
在腾讯内部业务中的多年积累，自主研发出数据库异步多线程强同步复制方案（Multi-thread Asynchronous Replication MAR），相比于Oracle的NDB引擎，Percona XtraDB Cluster和MariaDB Galera Cluster，其性能、效率和适用性更据优势。简单来说，MAR强同步方案强同步技术具有以下特点
-	一致性的同步复制，**保证节点间数据强一致性**；
-	**对业务层面完全透明**，业务层面无需做读写分离或同步强化工作； 
-	将串行同步线程异步化，**引入线程池能力**，大幅度提高性能
-	支持**集群**架构；
-	支持**自动成员控制**，故障节点自动从集群中移除；
-	支持**自动节点加入**，无需人工干预；
-	每个**节点都包含完整的数据副本**，可以随时切换；
-	**无需共享存储设备**
腾讯MAR方案强同步技术，只有当备机数据同步后，才由主机向应用返回事务应答，示意图如下：
![](//mccdn.qcloud.com/static/img/aee81e2ae246bd8e08d83f37132d7684/image.png)

从性能上优于其他主流同步方案，通过对比，在跨可用区(IDC机房)同样的测试方案下，我们发现其MAR技术性能优于MySQL半同步约5倍，优于MariaDB Galera Cluster性能1.5倍（此处使用sysbench标准用例测试）。
![](//mccdn.qcloud.com/static/img/60b6e6b80ccccad6692f9d68d93d7a51/image.png)

### 2.2 TDSQL集群架构
TDSQL采用集群架构，一套独立TDSQL系统至少需要十余系统或组件组成，架构简图如下
![](//mccdn.qcloud.com/img56834007cd44f.png)
其中，TDSQL最核心的三个主要模块是：决策调度集群（Tschedule）、数据库节点组（SET）和接入网关集群（TProxy），三个模块的交互都是通过配置集群（TzooKeeper）完成。
![](//mccdn.qcloud.com/img5683403e58125.png)
- **数据库节点组（又称“分片”）**：由兼容MySQL数据库引擎、监控和信息采集（Tagent）组成， 其架构由”一个主节点（Master）、若干备节点（Slave_n）、若干异地备份节点（Watcher_m）”，通常情况下：
 -  部署在跨机架、跨机房的服务器中；
 -  通过心跳监控和信息采集模块（Tagent）监控，确保集群的健壮性；
 -  分布式架构下，基于水平拆分，若干个分片（数据库节点组）提供一个“逻辑统一，物理分散”分布式的数据库实例。

- 决策调度集群：作为集群的管理调度中心，主要管理SET的正常运行，记录并分发数据库全局配置，其包括
 -  调度作业集群（TDSQL Scheduler）帮助DBA或者数据库用户自动调度和运行各种类型的作业，比如数据库备份、收集监控、生成各种报表或者执行业务流程等等，TDSQL把Schedule、zookeeper、OSS（运营支撑系统）结合起来通过时间窗口激活指定的资源计划，完成数据库在资源管理和作业调度上的各种复杂需求，Oralce也用DBMS_SCHEDULER支持类似的能力。
 -  程序协调与配置集群（TzooKeeper）：它是TDSQL提供配置维护、选举决策、路由同步等，并能支撑数据库节点组（分片）的创建、删除、替换等工作，并统一下发和调度所有DDL（数据库模式定义语言）操作。
 -  运维支撑系统（OSS）：基于TDSQL定制开发的一套综合的业务运营和管理平台，同时也是真正融合了数据库管理特点，将网络管理、系统管理、监控服务有机整合在一起。
 -  决策调度集群独立部署在腾讯云全国三大机房中（跨机房部署，异地容灾）。

-  接入网关集群（TProxy）:在网络层连接管理SQL解析、分配路由。
 - 与数据库引擎部署数量相同，分担负载并实现容灾；
 - 从配置集群（TzooKeeper）拉取数据库节点（分片）状态，提供分片路由，实现透明读写；
 - 记录并监控SQL执行信息，分析SQL执行效率，记录并监控用户接入信息，进行安全性鉴权，阻断风险操作；
 - TProxy前端部署为腾讯网关系统TGW，对用户提供唯一一个虚拟IP服务。

这种集群架构极大简化了各个节点之间的通信机制，也简化了对于硬件的需求，这就意味着即使是简单的x86服务器，也可以搭建出类似于小型机、共享存储等一样稳定可靠的数据库。而且，TDSQL保持了MySQL原有协议，保证基于MySQL开发的系统、应用都不需要修改。

### 2.2（金融级）高可用架构

TDSQL提供金融级的容灾架构，可以支持到两地三中心部署结构——同城节点直线距离大于10KM，异地节点直线距离大于100km，使用腾讯自主研发的高可用调度方案（High Availability HA）实现。示意图如下：
![](//mccdn.qcloud.com/img56834737b4b73.png)
>**请注意:使用“强同步”复制时，如果主库与备库自建网络中断或备库出现问题，主库也会被锁住（hang），而此时如果只有一个主库或一个备库，那么是无法做高可用方案的。—— 因为单一服务器服务，如果股指则直接导致部分数据完全丢失，不符合金融级数据安全要求。
>因此，（金融级云）数据库CDB for TDSQL的方案是使用“两个”备库，只要有一个备库是正常的，主库就不会被hang住。**

除此之外，未避免人为原因造成数据误删除，CDB for TDSQL通过数据备份服务(HDFS)提供可配天数的，至少3份完全备份和增量备份，进一步保障数据安全，做到不丢，不断，不错。
