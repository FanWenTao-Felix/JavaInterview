## 负载均衡

### Producers 负载均衡

对于同一个 topic 的不同 partition，Kafka 会尽力将这些 partition 分布到不同的 broker 服务器上，这种均衡策略实际上是基于 ZooKeeper 实现的。在一个 broker 启动时，会首先完成 broker 的注册过程，并注册一些诸如 “有哪些可订阅的 topic” 之类的元数据信息。producers 启动后也要到 ZooKeeper 下注册，创建一个临时节点来监听 broker 服务器列表的变化。由于在 ZooKeeper 下 broker 创建的也是临时节点，当 brokers 发生变化时，producers 可以得到相关的通知，从改变自己的 broker list。其它的诸如 topic 的变化以及 broker 和 topic 的关系变化，也是通过 ZooKeeper 的这种 Watcher 监听实现的。

在生产中，必须指定 topic；但是对于 partition，有两种指定方式：

- 明确指定 `partition(0-N)`，则数据被发送到指定 partition；
- 设置为 `RD_KAFKA_PARTITION_UA` ，则 Kafka 会回调 `partitioner` 进行均衡选取， `partitioner` 方法需要自己实现。可以轮询或者传入 key 进行 hash。未实现则采用默认的随机方法 `rd_kafka_msg_partitioner_random` 随机选择。

### Consumer 负载均衡

Kafka 保证同一 consumer group 中只有一个 consumer 可消费某条消息，实际上，Kafka 保证的是稳定状态下每一个 consumer 实例只会消费某一个或多个特定的数据，而某个 partition 的数据只会被某一个特定的 consumer 实例所消费。这样设计的劣势是无法让同一个 consumer group 里的 consumer 均匀消费数据，优势是每个 consumer 不用都跟大量的 broker 通信，减少通信开销，同时也降低了分配难度，实现也更简单。另外，因为同一个 partition 里的数据是有序的，这种设计可以保证每个 partition 里的数据也是有序被消费。

#### consumer 数量不等于 partition 数量

如果某 consumer group 中 consumer 数量少于 partition 数量，则至少有一个 consumer 会消费多个 partition 的数据；如果 consumer 的数量与 partition 数量相同，则正好一个 consumer 消费一个 partition 的数据，而如果 consumer 的数量多于 partition 的数量时，会有部分 consumer 无法消费该 topic 下任何一条消息。

#### 借助 ZooKeeper 实现负载均衡

关于负载均衡，对于某些低级别的 API，consumer 消费时必须指定 topic 和 partition，这显然不是一种友好的均衡策略。基于高级别的 API，consumer 消费时只需制定 topic，借助 ZooKeeper 可以根据 partition 的数量和 consumer 的数量做到均衡的动态配置。

consumers 在启动时会到 ZooKeeper 下以自己的 `conusmer-id` 创建临时节点 `/consumer/[group-id]/ids/[conusmer-id]`，并对 `/consumer/[group-id]/ids` 注册监听事件，当消费者发生变化时，同一 group 的其余消费者会得到通知。当然，消费者还要监听 broker 列表的变化。kafka 通常会将 partition 进行排序后，根据消费者列表，进行轮流的分配。