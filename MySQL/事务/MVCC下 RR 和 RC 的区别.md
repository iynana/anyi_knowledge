**快照读**
+ **读已提交隔离级别下**：**生成最新的readview**，事务中的每次查询前都会创建一个新的ReadView。新建的ReadView会更新creator_trx_id以外的其余字段，因此不可重复读现象依然存在
+ **可重复读隔离级别下**：**生成最初的readview**，只会在事务中的第一次查询时创建ReadView，同一个事务中后续所有的查询共用一个ReadView，由此便解决了不可重复读的问题

**锁机制**
+ 在 RC 中，只会对索引增加 `Record Lock`，不会添加 `Gap Lock` 和 `Next-Key Lock`
+ 在 RR 中，为了解决幻读的问题，在支持 `Record Lock` 的同时，还支持 `Gap Lock` 和 `Next-Key Lock`

**主从同步**
+ RC 隔离级别只支持 **row** 格式的 binlog（因为RC是非确定性执行）
+ RR 的隔离级别同时支持 **statement、row 以及 mixed** 三种(最好还是用row)