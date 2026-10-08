**背景知识：一个消费者组可以订阅多个Topic**

Kafka中的Rebalance称之为再均衡
是Kafka中确保消费者组下所有消费者如何达成一致，分配（订阅的Topic的）分区的机制

**Rebalance触发的时机有**：
- 消费者组中消费者的个数发生变化，例如有新的消费者加入消费者组，或者某个消费者停止了
- 消费者组订阅的 Topic 的 Partition 个数发生变化，Partition的个数增加或减少
- 消费者组订阅的 Topic 个数发生变化，订阅的Topic 个数增加或减少

**Rebalance的不良影响**：
- 发生Rebalance的时候，消费者组下的**所有消费者都会协调在一起共同参与**，Kafka使用分配策略，尽可能达到最公平的分配。
- Rebalance的过程会对消费者组产生非常严重的影响，Rebalance的过程中所有的消费者**都将停止工作，直到Rebalance完成**。