- String：不可变字符串序列，每次修改都会生成新的String对象，线程安全。
- StringBuffer：可变字符串序列，线程安全（方法用synchronized修饰），效率较低。
- StringBuilder：可变字符串序列，非线程安全，但效率最高。

字符串**不经常变化**用String，**多线程**共享可变字符串用StringBuffer，**单线程**字符串缓冲区用StringBuilder