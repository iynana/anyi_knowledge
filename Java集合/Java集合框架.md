# Collection
- List：有序、可重复的集合，如 ArrayList、LinkedList。
- Set：无序（通常指不保证顺序）、不可重复的集合，如 HashSet、TreeSet。
- Queue：用于模拟队列或双端队列，如 LinkedList、PriorityQueue。

# Map
+ Map 是存储键值对（key-value）的接口，每个键唯一映射到一个值，如 HashMap、TreeMap。
+ `ArrayList` 允许你使用泛型来确保类型安全，`Array` 则不可以。

# 两大体系的区别：
- 存储内容不同：Collection 存储单个元素，Map 存储键值对。
- 访问方式不同：Collection 通过元素本身操作，Map 通过键来访问值。