## Leader 选举过程

### 控制器的选举

在Kafka集群中会有一个或多个broker，其中有一个broker会被选举为控制器（Kafka Controller），它负责管理整个集群中所有分区和副本的状态等工作。比如当某个分区的 Leader 副本出现故障时，由控制器负责为该分区选举新的 Leader 副本。再比如当检测到某个分区的 ISR(In-Sync Replicas) 集合发生变化时，由控制器负责通知所有 broker 更新其元数据信息。

Kafka Controller 的选举是依赖 Zookeeper 来实现的，在 Kafka 集群中哪个 broker 能够成功创建 `/controller` 这个临时（EPHEMERAL）节点他就可以成为 Kafka Controller。Kafka Controller 的出现是处于性能考虑，当 Kafka 集群规模很大，partition 达到成千上万时，当 broker 宕机时，造成集群内大量的调整，会造成大量 Watch 事件被触发，Zookeeper负载会过重。

#### Controller 脑裂

kafka 中只有一个控制器 controller 负责分区的 leader 选举，同步 broker 的新增或删除消息，但有时由于网络问题，可能同时有两个 broker 认为自己是 controller ，这时候其他的 broker 就会发生脑裂。

解决方案：每当新的 controller 产生的时候就会在 ZK 中生成一个全新的、数值更大的 controller epoch 的标识，并同步给其他的 broker 进行保存，这样当第二个 controller 发送指令时，其他的 broker 就会自动忽略。

### 分区 Leader 的选举

分区 Leader 副本的选举由 Kafka Controller 负责具体实施。当创建分区（创建主题或增加分区都有创建分区的动作）或分区上线（比如分区中原先的 Leader 副本下线，此时分区需要选举一个新的 Leader 上线来对外提供服务）的时候都需要执行 Leader 的选举动作。

### 消费者相关的选举

组协调器 GroupCoordinator 需要为消费组内的消费者选举出一个消费组的 Leader，这个选举的算法也很简单，分两种情况分析。如果消费组内还没有 Leader，那么第一个加入消费组的消费者即为消费组的 Leader。如果某一时刻 Leader 消费者由于某些原因退出了消费组，那么会重新选举一个新的 Leader。