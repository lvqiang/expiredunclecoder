---
title: gluster存储
date: 2022-12-26 10:48:18
tags:
categories: devops
---

GlusterFS系统是一个可扩展的网络文件系统，相比其他分布式文件系统，GlusterFS具有高扩展性、高可用性、高性能、可横向扩展等特点，并且其没有元数据服务器的设计，让整个服务没有单点故障的隐患。GlusterFS是Scale-Out存储解决方案Gluster的核心，它是一个开源的分布式文件系统，具有强大的横向扩展能力，通过扩展能够支持数PB存储容量和处理数千客户端。**GlusterFS 适合存储大文件，小文件性能较差，还存在很大优化空间。**

## 原理

### 架构

<img src="https://ask.qcloudimg.com/http-save/6552462/67988kz8zt.png?imageView2/2/w/1620" alt="img" style="zoom:50%;" />

GlusterFS总体架构主要由存储服务器（Brick Server）、客户端以及 NFS/Samba [存储网关](https://cloud.tencent.com/product/csg?from=10680)组成。不难发现，GlusterFS 架构中没有元数据服务器组件，这是其最大的设计这点，对于提升整个系统的性能、可靠性和稳定性都有着决定性的意义。

- GlusterFS 支持 TCP/IP 和 InfiniBand RDMA 高速[网络互联](https://cloud.tencent.com/product/ccn?from=10680)。
- 客户端可通过原生 GlusterFS 协议访问数据，其他没有运行 GlusterFS 客户端的终端可通过 NFS/CIFS 标准协议通过存储网关访问数据（存储网关提供弹性卷管理和访问代理功能）。
- 存储服务器主要提供基本的数据存储功能，客户端弥补了没有元数据服务器的问题，承担了更多的功能，包括数据卷管理、I/O 调度、文件定位、数据缓存等功能，利用 FUSE（File system in User Space）模块将 GlusterFS 挂载到本地文件系统之上，实现 POSIX 兼容的方式来访问系统数据。

### 场景

GlusterFS 在企业中应用场景理论和实践上分析，**GlusterFS目前主要适用大文件存储场景，对于小文件尤其是海量小文件，存储效率和访问性能都表现不佳。海量小文件LOSF问题是工业界和学术界公认的难题，GlusterFS作为通用的分布式文件系统，并没有对小文件作额外的优化措施，性能不好也是可以理解的;**

### 概念

**Node**：集群node服务器节点，其中服务器间的关系是对等的，也就是说每个节点服务器都掌握了集群的配置信息。每个节点的信息更新都会向其他节点通告，保证节点间信息的一致性。但如果集群规模较大，节点众多时，信息同步的效率就会下降，节点信息的非一致性概率就会大大提高。

**Brick**：GlusterFS中的存储单元，通过是一个节点服务器的一个导出目录。可以通过主机名和目录名来标识，如'SERVER:EXPORT'。**Brick是底层的RAID或磁盘经XFS或ext4文件系统格式化而来，所以继承了文件系统的限制**。**每个节点上的brick数是不限的**。**理想的状况是，一个集群的所有Brick大小都一样。**

<img src="http://cdn.expiredunclecoder.tech/image-20221227093933522.png" alt="image-20221227093933522" style="zoom:67%;" />

**Volume**：一组bricks的逻辑集合;**创建时命名来识别**。**Volume是一个可挂载的目录**。**每个节点上的brick数是不变的**。**一个节点上的不同brick可以属于不同的卷**。

**GFID**：GlusterFS卷中的每个文件或目录都有一个唯一的128位的数据相关联，其用于模拟inode;

### 集群模式

GlusterFS分布式存储集群的模式只数据在集群中的存放结构，类似于磁盘阵列中的级别。

#### 分布式卷(Distributed Volume)

默认模式，基于 Hash 算法将文件分布到所有 brick server，只是扩大了磁盘空间，不具备容错能力。由于distribute volume 使用本地文件系统，因此存取效率并没有提高，相反会因为网络通信的原因使用效率有所降低，另外本地存储设备的容量有限制，因此支持超大型文件会有一定难度。

```bash
$ gluster volume create test-volume server1:/exp1 server2:/exp2 server3:/exp3
```

<img src="https://ask.qcloudimg.com/http-save/yehe-5334888/4528ec16178beb8afeeb9b368ab4e868.png?imageView2/2/w/1620" alt="img" style="zoom: 67%;" />

#### 条带卷(Striped Volume)

相当于raid0，文件是分片均匀写在各个节点的硬盘上的，优点是分布式读写，性能整体较好。缺点是没冗余，分片随机读写可能会导致硬盘IOPS饱和。

```bash
$ gluster volume create test-volume stripe 2 transport tcp server1:/exp1 server2:/exp2
```

<img src="https://ask.qcloudimg.com/http-save/yehe-5334888/b2d284137cc0d55b541b291e7a37822a.png?imageView2/2/w/1620" alt="img" style="zoom:67%;" />

#### 复制卷(Replicated Volume)

复制模式，AFR相当于raid1，复制的份数，决定集群的大小，通常与分布式卷或者条带卷组合使用，解决前两种存储卷的冗余缺陷。缺点是磁盘利用率低。复本卷在创建时可指定复本的数量，通常为2或者3，复本在存储时会在卷的不同brick上，因此有几个复本就必须提供至少多个brick，当其中一台服务器失效后，可以从另一台服务器读取数据，因此复制GlusterFS卷提高了数据可靠性的同时，还提供了数据冗余的功能。也是在容器存储中较为推崇的一种。

```bash
$ gluster volume create test-volume replica 2 transport tcp server1:/exp1 server2:/exp2
```

<img src="https://ask.qcloudimg.com/http-save/yehe-5334888/2fcc1ea874c3541535a2f9c254cec913.png?imageView2/2/w/1620" alt="img" style="zoom:67%;" />

#### 分布式复制卷（Distributed Replicated Volume)

Brick server 数量是镜像数的倍数，兼具distribute和replica卷的特点,可以在2个或多个节点之间复制数据。分布式的复制卷，volume中bric 所包含的存储服务器数必须是replica 的倍数(>=2倍)，兼顾分布式和复制式的功能。

```bash
$ gluster volume create test-volume replica 2 transport tcp server1:/exp1 server2:/exp2 server3:/exp3 server4:/exp4
```

<img src="https://ask.qcloudimg.com/http-save/yehe-5334888/820967cc5545060877882ac1bdbfacfc.png?imageView2/2/w/1620" alt="img" style="zoom:67%;" />

#### 分布式条带卷(Distributed Striped Volume)

Brick server数量是条带数的倍数，兼具distribute和stripe卷的特点。分布式的条带卷volume中brick所包含的存储服务器数必须是stripe的倍数(>=2倍)，兼顾分布式和条带式的功能。每个文件分布在四台共享服务器上，通常用于大文件访问处理，最少需要 4 台服务器才能创建分布条带卷。

```bash
$ gluster volume create test-volume stripe 4 transport tcp server1:/exp1 server2:/exp2 server3:/exp3 server4:/exp4 server5:/exp5 server6:/exp6 server7:/exp7 server8:/exp8
```

<img src="https://ask.qcloudimg.com/http-save/yehe-5334888/bc18612d93a935ee21cbdbca1e35afea.png?imageView2/2/w/1620" alt="img" style="zoom:67%;" />

#### 条带复制卷(Stripe Replica Volume)

类似 RAID 10，同时具有条带卷和复制卷的特点。

<img src="https://ask.qcloudimg.com/http-save/6552462/abfnnsu3xo.jpeg?imageView2/2/w/1620" alt="img" style="zoom:67%;" />

#### 分布式条带复制卷(Distribute Stripe Replica Volume)

三种基本卷的复合卷，通常用于类 Map Reduce 应用。

<img src="https://ask.qcloudimg.com/http-save/6552462/42cwgm72rg.jpeg?imageView2/2/w/1620" alt="img" style="zoom:67%;" />



### 特点

#### 无元数据设计

元数据是用来描述一个文件或给定区块在分布式文件系统中所在的位置，简而言之就是某个文件或某个区块存储的位置。传统分布式文件系统大都会设置元数据服务器或者功能相近的管理服务器，主要作用就是用来管理文件与数据区块之间的存储位置关系。相较其他分布式文件系统而言，GlusterFS并没有集中或者分布式的元数据的概念，取而代之的是弹性哈希算法。集群中的任何服务器和客户端都可以利用哈希算法、路径及文件名进行计算，就可以对数据进行定位，并执行读写访问操作;这种设计带来的好处是极大的提高了扩展性，同时也提高了系统的性能和可靠性；另一显著的特点是如果给定确定的文件名，查找文件位置会非常快。但是如果要列出文件或者目录，性能会大幅下降，因为列出文件或者目录时，需要查询所在节点并对各节点中的信息进行聚合。此时有元数据服务的分布式文件系统的查询效率反而会提高许多。

