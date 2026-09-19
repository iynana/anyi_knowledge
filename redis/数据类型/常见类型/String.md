+ Session Id，Token，序列化对象，图片路径存储
+ 计数
+ 分布式锁 ==（SET key value NX EX）==

redis的String基于SDS，相对于C中的string的优势有：1.实现了获取len，保证了二进制安全 2.预分配，减少内存分配的次数