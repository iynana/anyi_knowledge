
Executors 是一个工厂类，它提供了用于创建不同类型线程池的静态方法
线程池不允许使用 Executors 去创建，而是通过ThreadPoolExecutor的方式，规避资源耗尽的风险
Executors 返回的线程池对象的弊端如下：

- **newFixedThreadPool()** 和 **newSingleThreadPool()**
	- 允许的==请求队列长度==为 Integer.MAX_VALUE，可能会堆积大量的请求，从而导致 OOM
- **newCachedThreadPool()**
	- 允许的==创建线程数量==为 Integer.MAX VALUE，可能会创建大量的线程，从而导致 OOM