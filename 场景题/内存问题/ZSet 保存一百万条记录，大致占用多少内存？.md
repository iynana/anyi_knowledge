大型 ZSet 底层同时有 dict 和 skiplist。

+ dict：key(16 B) + value(8 B)
+ skiplist：平均层数为1.33，每个节点有sds(24B) + sds指针(8 B) + value(8 B) + backward(8 B)，总数1.3倍的 zskiplistLevel ={forward, span} (16 B)
