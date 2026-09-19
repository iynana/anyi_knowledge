```
SELECT name, COUNT(*)
FROM employees
JOIN departments ON employees.department id = departments.id
WHERE employees.salary > 50000
GROUP BY departments.name
HAVING COUNT(*) > 10
ORDER BY name
LIMIT 5;
```
下面是 InnoDB 处理 SQL 查询的大致执行顺序，但注意这个顺序是逻辑上的执行顺序，实际的物理执行可能会有所不同。数据库优化器可能会根据统计信息、索引、查询类型等因素以不同的方式执行查询。
1. **FROM**：识别确定出数据来源的表
2. **ON**：处理 JOIN 的关联条件
3. **JOIN**：合并关联的表数据
4. **WHERE**：对JOIN操作的结果应用WHERE中的过滤条件，过滤行数据
5. **GROUP BY**：对数据按指定字段进行分组，分组操作通常在WHERE条件过滤之后进行
6. **HAVING**：用于筛选过滤分组后的数据
7. **SELECT**：选择需要返回的字段（计算列、别名等）
8. **DISTINCT**：去除重复的行
9. **ORDER BY**：对结果进行排序
10. **LIMIT**：限制返回的行数。这通常是整个查询过程的最后一步