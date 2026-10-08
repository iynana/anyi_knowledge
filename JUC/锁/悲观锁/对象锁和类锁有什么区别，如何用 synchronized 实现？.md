对象锁锁的是某个实例，类锁锁的是这个类对应的 `Class` 对象。

**对象锁**：锁当前实例 `this`
+ 方法签名上加 `synchronized`，如```public synchronized void increment()```
+ `synchronized`包围，`synchronized(this) {}`

**类锁**：
+ 方法签名上加 `synchronized`，如```public static synchronized void increment()```
+ `synchronized`包围，`synchronized(类.this) {}`