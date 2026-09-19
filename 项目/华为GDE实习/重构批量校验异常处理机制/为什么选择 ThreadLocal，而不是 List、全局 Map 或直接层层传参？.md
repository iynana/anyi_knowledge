+ 校验链路比较深，如果把 `ErrorCollector` 作为参数逐层传递，会侵入大量已有接口
+ `ThreadLocal` 可以在当前线程保存上下文，并且隔离不同线程