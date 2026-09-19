`ConcurrentHashMap` + 双向链表
+ `ConcurrentHashMap` 支持高并发、高 QPS 读
+ 高并发下，`get()`后把访问事件写入缓冲区，交给异步任务执行，热点请求合并
+ 缓存击穿：`CompletableFuture`合并并发请求
+ 缓存穿透：缓存空值
+ 缓存雪崩：随机值