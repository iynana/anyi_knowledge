Offset：消息位移，它表示分区中每条消息的位置信息，是一个**单调递增且不变的值**。
换句话说，offset可以用来**唯一的标识分区中每一条记录**。

Kafka为了使我们能够专注于自己的业务逻辑，提供了自动提交offset的功能，这也是默认配置项。

当消费者配置 `enable.auto.commit=true` 时，消费者会在后台启动一个**定时任务**，
每隔 `auto.commit.interval.ms`（默认 5 秒）自动提交一次 offset

自动提交的 offset 存储在 Kafka 的内部系统主题`consumer_offsets`中