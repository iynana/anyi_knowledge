```java
// key -> 缓存节点
Map<Integer, Node> cache;

// freq -> 该频率下的节点链表
Map<Integer, DoublyLinkedList> freqMap;

// 当前缓存中最小访问频率
int minFreq;
```

- **查 `get(key)`**：先通过 `HashMap` 找到节点，找到后，从原 `freq` 对应的双向链表删除，若原频率等于 `minFreq` 且桶空了，则 `minFreq ++`；然后节点 `freq ++`，加入新频率桶的链表头部，最后返回 value。
    
- **写 `put(key, value)`**，新节点 `freq=1`，放入 `freq=1` 链表头部，同时设置 `minFreq=1`。
    - **Key 已存在**：更新 `value`，然后和 `get` 一样执行一次频率提升。
    - **Key 不存在且容量已满**：找到 `minFreq` 对应的链表，淘汰链表尾部节点（即**频率最低且最久未使用**的节点）。


一句话面试版：
> **LFU 查询时通过 HashMap O(1) 找节点并将其从 `freq` 桶移动到 `freq + 1` 桶；写入时如果容量满，就从 `minFreq` 桶尾部淘汰节点，再以 `freq=1` 插入新节点，因此 get 和 put 都能做到 O(1)。**