**BinLog**： 是记录所有**数据库表结构变更** 以及 **表数据修改**的二进制日志，不会记录Select和Show操作
**用途**：数据备份、灾难恢复、主从同步
**写入时机**：事务提交时

## 为什么不能只要binlog?
+ binlog不支持WAL（binlog是逻辑描述，redo是物理页变更描述），无法恢复“中间态”（而且binlog是逻辑恢复，速度慢）
+ 事务提交时才写入，无法回滚事务

## 事务写完binlog或redolog直接返回会导致主从不一致或不可恢复
故mysql采用了两阶段提交：redolog prepare --> 写binlog --> redolog commit
+ 崩溃恢复规则：如果redo是prepare状态，看binlog是否存在：有则提交，无则回滚。
+ 为什么不先写binlog？写binlog后崩溃会导致从库有数据主库没有。
## BinLog的三种格式
MySQL 的BinLog支持三种格式： Statement、Row、Mixed
- **Statement**
	- BinLog里面记录的是 SQL 语句==原文==
	- 这种格式有bug，例如使用now()等时间函数，导致主从同步后数据不相同

- **Row**（生成环境常用）
	- BinLog里面记录的是 ==数据的具体变化==
	- 例如 会记录修改的表名称、操作类型
		- Insert操作 Row 格式会记录新插入的行的 所有字段列的值
		- Update操作 Row 格式会记录行 更新前后的 所有字段列的值
		- Delete操作 Row 格式会记录行删除前的 所有字段列的值

	- **优点**：可以更好的保证主从数据同步时数据一致性，避免了`STATEMENT`格式中可能出现的不一致性问题
	- **缺点**：
		- 需要记录的内容更多，例如批量修改、插入等，就需要把每条记录都记录，存在性能问题
		- 文件体积大，在主从同步时，网络IO更高，耗费时间长

- **Mixed**
	- 由MySQL根据具体情况自主选择使用Statement格式或Row格式来记录
	- 默认使用Statement格式，记录SQL语句
	- 在必要时候，对于某些可能导致主从不一致的SQL语句，自动使用Row格式，记录数据变更
