Spring事务隔离级别对应数据库的隔离级别，通过 @Transactional 的 isolation 属性设置，包括：
1. **DEFAULT**：使用底层数据库默认隔离级别。
2. **READ_UNCOMMITTED**：读未提交，允许脏读。
3. **READ_COMMITTED**：读已提交，防止脏读。
4. **REPEATABLE_READ**：可重复读，防止脏读和不可重复读。
5. **SERIALIZABLE**：串行化，最高级别，防止脏读、不可重复读和幻读。