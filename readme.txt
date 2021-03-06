apache rocketMQ

开源的分布式消息传递和流处理平台

低延迟
高吞吐量
线性性能

rocketmq在低延迟和高可靠性方面更好
kafka主要用于日志收集

kafka的分区设计
生产者的并行性受分区数量的限制
消费者消费并行也受分区的数量限制

为什么kafka不支持更多的分区
每个分区都存储整个消息数据。尽管每个分区都按顺序写入磁盘，但随着并发写入分区数量的增加，从操作系统的角度来看，写入变得随机。
由于分散的数据文件，很难使用Linux IO组提交机制。

如何在RocketMQ中支持更多分区？
所有消息数据都存储在提交日志文件中。所有写入都是完全顺序的，而读取是随机的。
ConsumeQueue存储实际的用户消费位置信息，这些信息也以顺序方式刷新到磁盘。

优点
每个使用队列都是轻量级的，并且包含有限数量的元数据。
对磁盘的访问是完全顺序的，这可以避免磁盘锁争用，并且在创建大量队列时不会导致高磁盘IO等待。

缺点
消息消耗将首先读取消耗队列，然后提交日志。在最坏的情况下，该过程会带来一定的成本。
提交日志和使用队列需要在逻辑上保持一致，这为编程模型带来了额外的复杂性。

设计动机：

随机阅读。尽可能多地读取以提高页面缓存命中率，并减少读取IO操作。如此大的内存仍然是可取的。如果累积大量消息，读取性能是否会严重降低？答案是否定的，原因如下：
即使消息的大小仅为1KB，系统也会提前读取更多数据，请参阅PAGECACHE预取以供参考。这意味着对于续集数据读取，它将访问将执行的主存储器而不是慢速磁盘IO读取。
从磁盘随机访问CommitLog。如果在SSD的情况下将I / O调度程序设置为NOOP，则读取qps将大大加速，因此比其他电梯调度程序算法快得多。
鉴于ConsumeQueue仅存储固定大小的元数据，主要用于记录消费进度，因此可以很好地支持随机读取。利用页面缓存预取，访问ConsumeQueue与访问主内存一样快，即使是在大量消息累积的情况下也是如此。因此，ConsumeQueue不会对读取性能带来明显的损失。
CommitLog几乎存储所有信息，包括消息数据。与关系数据库的重做日志类似，只要提交日志存在，就可以完全恢复消耗队列，消息密钥索引和所有其他所需数据。

activeMQ 推 不支持批量 支持广播 支持消息过滤 不支持服务器触发重新传递 jdbc、日志存储 支持消息追溯 zk高可用 不支持消息跟踪
kafka 拉 分区内确保顺序消费 异步生产者 不支持广播 kafka streams可以进行消息过滤 不支持服务器触发重新传递 高性能文件存储 支持偏移量消息追溯 zk高可用 不支持消息跟踪
rocketMQ 拉 严格排序 同步生产者确保消息不丢失 支持广播 支持sql92消息过滤 支持服务器触发重新传递 高性能低延迟的文件存储 支持时间戳和偏移量消息追溯 主从高可用 支持消息跟踪


集群搭建
rocketmq分布式协调用的是namesrv

》多master模式
一个集群无Slave，全是Master，例如2个Master或者3个Master

优点：配置简单，单个Master宕机或重启维护对应用无影响，在磁盘配置为RAID10时，即使机器宕机不可恢复情况下，由于RAID10磁盘非常可靠，消息也不会丢（异步刷盘丢失少量消息，同步刷盘一条不丢）。性能最高。
缺点：单台机器宕机期间，这台机器上未被消费的消息在机器恢复之前不可订阅，消息实时性会受到影响。

》多Master多Slave模式，异步复制
每个Master配置一个Slave，有多对Master-Slave，HA采用异步复制方式，主备有短暂消息延迟，毫秒级。

优点：即使磁盘损坏，消息丢失的非常少，且消息实时性不会受影响，因为Master宕机后，消费者仍然可以从Slave消费，此过程对应用透明，不需要人工干预。性能同多Master模式几乎一样。
缺点：Master宕机，磁盘损坏情况，会丢失少量消息。

》多Master多Slave模式，同步双写
每个Master配置一个Slave，有多对Master-Slave，HA采用同步双写方式，主备都写成功，向应用返回成功。

优点：数据与服务都无单点，Master宕机情况下，消息无延迟，服务可用性与数据可用性都非常高。
缺点：性能比异步复制模式略低，大约低10%左右，发送单个消息的RT会略高。
官方建议金融行业交易系统采用这种方案

NameServer是一个几乎无状态节点，可集群部署，节点之间无任何信息同步。
每个Broker与NameServer集群中的所有节点建立长连接，定时注册 Topic 信息到所有 NameServer。
Producer与NameServer集群中的其中一个节点（随机选择）建立长连接，定期从NameServer取Topic路由信息，并向提供Topic服务的Master建立长连接，且定时向Master发送心跳。Producer完全无状态，可集群部署。
Consumer与NameServer集群中的其中一个节点（随机选择）建立长连接，定期从NameServer取Topic路由信息，并向提供Topic服务的Master、Slave建立长连接，且定时向Master、Slave发送心跳。Consumer既可以从Master订阅消息，也可以从Slave订阅消息，订阅规则由Broker配置决定
