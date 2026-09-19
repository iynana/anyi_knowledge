Leader 会维护一组与自己数据**基本同步的 Follower 副本集合**（ISR），当 Leader 故障时，优先从 ISR 中选举新的 Leader，以保证高可用并尽量避免数据丢失

**‌ISR集合的定义‌**：ISR是指与Leader副本保持同步的Follower副本集合。这些副本已经复制了Leader副本的所有数据，并且它们的落后时间在一定范围内（由replica.lag.time.max.ms参数配置），因此被认为是可靠的、可以用于故障转移和数据恢复的副本。当Follower副本的延迟超过replica.lag.time.max.ms（默认10秒）时，会被移出ISR集合。

- **选举保证节点容灾‌**：当Leader副本出现故障时，Kafka会从ISR集合中选举一个新的Leader副本。由于ISR中的副本与之前的Leader副本保持同步，新的Leader副本能够继续提供服务，而不会丢失数据。这确实保证了节点的容灾能力。

- **‌Follower副本保证备份‌**：ISR中的Follower副本不仅作为备份存在，它们还积极参与消息的复制过程。当消息被写入Leader副本时，Leader副本会将消息复制给ISR中的所有Follower副本。这样，即使Leader副本出现故障，ISR中的Follower副本也能提供完整的数据备份。

- **‌ISR的动态管理‌**：Kafka会动态地管理ISR集合。如果某个Follower副本无法跟上Leader副本的更新速度（即落后时间超过replica.lag.time.max.ms），它将被移出ISR集合。一旦该副本重新追上Leader副本，它将被重新加入ISR集合。这种动态管理机制确保了ISR集合中的副本始终是可靠的。

- **数据一致性保证：生产者在写入数据时，可以通过设置 acks 参数来控制数据的一致性级别**。设置 acks=all（或acks=-1）：当消息被写入 Kafka 的分区时，它首先会被写入 Leader，然后 Leader 将消息复制给 ISR 中的所有副本。只有当 ISR 中的所有副本都成功地接收到并确认了消息后，主副本才会认为消息已成功提交。这种机制确保了数据的可靠性和致性。
	- ack=0，不等待确认
	- ack=1，等待 Leader 副本确认
	- ack=-1，等待 ISR 中所有副本确认（需配置 `min.insync.replicas`定义最小副本）