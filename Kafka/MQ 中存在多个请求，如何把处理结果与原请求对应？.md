MQ 中多个请求并发时，一般给每个请求生成**唯一的 requestId**，并让后续所有消息和结果都携带这个 ID。

请求方维护 `requestId -> Future` 的 hash，结果到达后根据 requestId 找回原请求并完成回调。
+ 发请求的线程执行 `future.get()` 等待。
+ MQ 消费线程执行 `future.complete(result)` 填入结果并唤醒等待线程。