## 简介
#### Hadoop基于几个重要的概念
1. 使用商业服务器集群来同时存储和处理大量数据比使用高端的强劲服务器更便宜。换句话说，hadoop使用横向扩展（scale-out）架构，而不是纵向扩展（scale-up）架构
2. 以软件形式来实现容错比通过硬件实现更便宜。容错服务器很贵，而Hadoop不依赖于容错服务器，它假设服务器会出错，并透明地处理服务器错误。应用开发者不需要操心处理硬件错误，哪些繁杂的细节可以交给Hadoop来处理
3. 通过网络把代码从一台计算机转到另一台比通过相同的网络移动大数据集更有效、更快速。
4. 把核心数据处理逻辑和分布式计算逻辑分开的方式，使得编写一个分布式应用更加简单。Hadoop提供了一个框架，隐藏了编写分布式应用的复杂性，使得各个组织有更多可用的应用开发者。

#### Hadoop组件
Hadoop由三个关键组件组成：集群管理器、分布式计算引擎和分布式文件系统
| Hadoop关键概念组件 | 关键Hadoop组件 |
| :----------------: | :------------: |
|     集群管理器     |      YARN      |
|   分布式计算引擎   |   MapReduce    |
|   分布式文件系统   |      HDFS      |

## 1.1 HDFS
1. HDFS 是一个块结构的文件系统，把文件分成固定大小的块，通常叫作分块或分片。默认的块大小事128M，可以配置。
2. HDFS会把一个文件的各个块分布在不同机器上，应用可以并行文件级别的读和写，使得读写跨越不通过计算机，分布在大量磁盘中的大HDFS文件比读写存储在单一磁盘上的大文件更迅速。
3. 把一个文件分布到多台机器上会增加集群中某台机器宕机时文件不可用的风险，HDFS通过复制每个文件到多台机器来降低风险，默认的复制因子是3
4. 一个HDFS集群包含两种类型的节点
   - NameNode：管理文件系统的命名空间，存储一个文件的所有元数据。比如，追踪文件名、权限和文件块位置。为了更快地访问元数据，NameNode把所有元数据都存储在内存中
   - DataNode以文件块的形式存储实际的文件内容。
5. NameNode周期性接收来自HDFS集群中DataNode的两种类型的消息：心跳消息和块报告消息
    - DataNode发送一个心跳消息来告知NameNode 工作正常
    - 块报告消息包含一个DataNode上所有数据块的列表
6. 客户端读文件：
    1. 首先访问NameNode，NameNode以组成文件的所有文件块的位置做响应。块的位置标识了持有对应文件块数据的dataNode
    2. 客户端向DataNode发送请求，获得每个文件块。NameNode不参与从dataNode到客户端的时间数据传输
7. 客户端写数据：
    1. 首先访问NameNode并在HDFS创建一个新的条目；NameNode会检查同名文件是否已存在以及客户端是否有权限创建新文件
    2. 客户端请求NameNode为文件的第一块选择DataNode。会在所有持有块的复制节点之间创建一个管道，并把块发送到管道中的第一个DataNode。第一个DataNode在本地存储数据块，然后转发给第二个DataNode；第二个dataNode在本地存储相应数据，并转发给第三个DataNode。在所有委派的dataNode上都存储了第一个块之后，客户端请求为第二个块分配DataNde。
    这个过程持续到所有文件块都已在DataNode上存储。最后，客户端告知NameNode文件写操作已完成。
   
## MapReduce
1. MapReduce是Hadoop提供的分布式计算引擎。
2. Mapreduce提供集群中并行处理大数据集的计算框架，抽象了集群计算，提供了编写分布式数据处理应用的高级结构
3. Mapreduce框架自动在集群中各计算节点上调度应用的执行，它会负载均衡、节点宕机和复杂的节点内通信
4. Mapreduce 应用的基本组成块是两个函数：Map 和 Reduce，名称借鉴于函数式编程，所有数据处理作业都用这个两个函数表达
