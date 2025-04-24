# MySQL执行计划分析

优化 SQL 的第一步应该是读懂 SQL 的执行计划。本篇文章，我们一起来学习下 MySQL `EXPLAIN` 执行计划相关知识。

## 什么是执行计划？

**执行计划** 是指一条 SQL 语句在经过 **MySQL 查询优化器** 的优化后，具体的执行方式。

执行计划通常用于 SQL 性能分析、优化等场景。通过 `EXPLAIN` 的结果，可以了解到如数据表的查询顺序、数据查询操作的操作类型、哪些索引可以被命中、哪些索引实际会命中、每个数据表有多少行记录被查询等信息。

## 如何获取执行计划？

MySQL 为我们提供了 `EXPLAIN` 命令，来获取执行计划的相关信息。

需要注意的是，`EXPLAIN` 语句并不会真的去执行相关的语句，而是通过查询优化器对语句进行分析，找出最优的查询方案，并显示对应的信息。

`EXPLAIN` 执行计划支持 `SELECT`、`DELETE`、`INSERT`、`REPLACE` 以及 `UPDATE` 语句。我们一般多用于分析 `SELECT` 查询语句，使用起来非常简单，语法如下：

    EXPLAIN + SELECT 查询语句；

我们简单来看下一条查询语句的执行计划：

    mysql> explain SELECT * FROM dept_emp WHERE emp_no IN (SELECT emp_no FROM dept_emp GROUP BY emp_no HAVING COUNT(emp_no)>1);
    +----+-------------+----------+------------+-------+-----------------+---------+---------+------+--------+----------+-------------+
    | id | select_type | table    | partitions | type  | possible_keys   | key     | key_len | ref  | rows   | filtered | Extra       |
    +----+-------------+----------+------------+-------+-----------------+---------+---------+------+--------+----------+-------------+
    |  1 | PRIMARY     | dept_emp | NULL       | ALL   | NULL            | NULL    | NULL    | NULL | 331143 |   100.00 | Using where |
    |  2 | SUBQUERY    | dept_emp | NULL       | index | PRIMARY,dept_no | PRIMARY | 16      | NULL | 331143 |   100.00 | Using index |
    +----+-------------+----------+------------+-------+-----------------+---------+---------+------+--------+----------+-------------+

可以看到，执行计划结果中共有 12 列，各列代表的含义总结如下表：

| **列名**      | **含义**                                     |
| ------------- | -------------------------------------------- |
| id            | SELECT 查询的序列标识符                      |
| select_type   | SELECT 关键字对应的查询类型                  |
| table         | 用到的表名                                   |
| partitions    | 匹配的分区，对于未分区的表，值为 NULL        |
| type          | 表的访问方法                                 |
| possible_keys | 可能用到的索引                               |
| key           | 实际用到的索引                               |
| key_len       | 所选索引的长度                               |
| ref           | 当使用索引等值查询时，与索引作比较的列或常量 |
| rows          | 预计要读取的行数                             |
| filtered      | 按表条件过滤后，留存的记录数的百分比         |
| Extra         | 附加信息                                     |

## 如何分析 EXPLAIN 结果？

为了分析 `EXPLAIN` 语句的执行结果，我们需要搞懂执行计划中的重要字段。

### id

`SELECT` 标识符，用于标识每个 `SELECT` 语句的执行顺序。

id 如果相同，从上往下依次执行。id 不同，id 值越大，执行优先级越高，如果行引用其他行的并集结果，则该值可以为 NULL。

### select_type

查询的类型，主要用于区分普通查询、联合查询、子查询等复杂的查询，常见的值有：

-   **SIMPLE**：简单查询，不包含 UNION 或者子查询。
-   **PRIMARY**：查询中如果包含子查询或其他部分，外层的 SELECT 将被标记为 PRIMARY。
-   **SUBQUERY**：子查询中的第一个 SELECT。
-   **UNION**：在 UNION 语句中，UNION 之后出现的 SELECT。
-   **DERIVED**：在 FROM 中出现的子查询将被标记为 DERIVED。
-   **UNION RESULT**：UNION 查询的结果。

### table

查询用到的表名，每行都有对应的表名，表名除了正常的表之外，也可能是以下列出的值：

-   **`<unionM,N>`** : 本行引用了 id 为 M 和 N 的行的 UNION 结果；
-   **`<derivedN>`** : 本行引用了 id 为 N 的表所产生的的派生表结果。派生表有可能产生自 FROM 语句中的子查询。
-   **`<subqueryN>`** : 本行引用了 id 为 N 的表所产生的的物化子查询结果。

### type（重要）

查询执行的类型，描述了查询是如何执行的。所有值的顺序从最优到最差排序为：

system > const > eq\_ref > ref > fulltext > ref\_or\_null > index\_merge > unique\_subquery > index\_subquery > range > index > ALL

常见的几种类型具体含义如下：

-   **system**：如果表使用的引擎对于表行数统计是精确的（如：MyISAM），且表中只有一行记录的情况下，访问方法是 system ，是 const 的一种特例。
-   **const**：表中最多只有一行匹配的记录，一次查询就可以找到，常用于使用主键或唯一索引的所有字段作为查询条件。
-   **eq\_ref**：当连表查询时，前一张表的行在当前这张表中只有一行与之对应。是除了 system 与 const 之外最好的 join 方式，常用于使用主键或唯一索引的所有字段作为连表条件。
-   **ref**：使用普通索引作为查询条件，查询结果可能找到多个符合条件的行。
-   **index\_merge**：当查询条件使用了多个索引时，表示开启了 Index Merge 优化，此时执行计划中的 key 列列出了使用到的索引。
-   **range**：对索引列进行范围查询，执行计划中的 key 列表示哪个索引被使用了。
-   **index**：查询遍历了整棵索引树，与 ALL 类似，只不过扫描的是索引，而索引一般在内存中，速度更快。
-   **ALL**：全表扫描。

### possible_keys

possible\_keys 列表示 MySQL 执行查询时可能用到的索引。如果这一列为 NULL ，则表示没有可能用到的索引；这种情况下，需要检查 WHERE 语句中所使用的的列，看是否可以通过给这些列中某个或多个添加索引的方法来提高查询性能。

### key（重要）

key 列表示 MySQL 实际使用到的索引。如果为 NULL，则表示未用到索引。

### key_len

key\_len 列表示 MySQL 实际使用的索引的最大长度；当使用到联合索引时，有可能是多个列的长度和。在满足需求的前提下越短越好。如果 key 列显示 NULL ，则 key\_len 列也显示 NULL 。

### rows

rows 列表示根据表统计信息及选用情况，大致估算出找到所需的记录或所需读取的行数，数值越小越好。

#### `rows` 的评估原理

1. **统计基础**：
   - 基于表的统计信息和索引的基数(cardinality)
     - 表的统计信息基本表信息里包含：
       - **TABLE_ROWS**：表中行数的估计值（MyISAM 精确，InnoDB 估算）
     - 索引统计信息里面包含了索引的基数
       - 索引基数是通过采样来的：
         - 随机选取索引的N个叶子页（默认8页）
         - 计算这些页面上不同值的数量
       - 总基数 = 采样页的不同值数 × (总页数 / 采样页数)
       - 当表数据变化超过1/16时会触发重新统计
   - 使用 `SHOW INDEX FROM table_name` 可以看到索引的基数估计值
2. **估算方法**：
   - 对于全表扫描：`rows` ≈ 表的总行数
   - 对于索引扫描：
     - 等值查询(WHERE col=value)：基于索引的选择性估算
     - 范围查询(WHERE col>value)：基于索引的直方图统计(MySQL 8.0+)或固定比例估算
3. **影响因素**：
   - 表的统计信息时效性(ANALYZE TABLE 更新)
   - 索引的选择性
   - 查询条件的复杂度

#### 不同查询类型的估算示例

1. **等值查询**：

   ```
   EXPLAIN SELECT * FROM users WHERE id = 100;
   ```

   - 使用主键：rows ≈ 1 (精确匹配)
   - 使用普通唯一索引：rows ≈ 1
   - 使用非唯一索引：基于索引选择性估算

2. **范围查询**：

   ```
   EXPLAIN SELECT * FROM orders WHERE amount > 1000;
   ```

   - 基于索引直方图或固定比例(如30%)估算

3. **多条件查询**：

   ```
   EXPLAIN SELECT * FROM products WHERE category='electronics' AND price>500;
   ```

   - 各条件选择性相乘
   - 如category过滤50%，price过滤40%，则rows ≈ 总行数×50%×40%

#### 常见问题

1. **为什么rows不准确？**

   - 统计信息过时(需要ANALYZE TABLE)
   - 数据分布不均匀
   - 关联查询的复杂度导致估算误差累积

2. **如何提高rows准确性？**

   ```
   ANALYZE TABLE table_name;  -- 更新统计信息
   ```

   - MySQL 8.0+ 使用直方图统计：`ANALYZE TABLE table_name UPDATE HISTOGRAM ON column_name`

3. **rows与实际行数的关系**

   - 只是优化器的预估值
   - 实际执行可能完全不同
   - 偏差大时应考虑优化查询或更新统计信息

#### 执行计划选择中的角色

优化器通过比较不同执行计划的`rows`估算值来选择成本最低的方案：

- 计算每个潜在计划的读取成本(rows × 过滤成本)
- 选择总成本最低的执行计划

### filter

`filtered` 字段表示存储引擎返回的数据经过服务器层进一步过滤后，最终保留的预估百分比，如：

1. **存储引擎层**：首先根据索引条件（如 `a >= 1`）检索出一批数据
2. **服务器层**：然后对这些数据应用其他无法用索引过滤的条件（如 `b >= 2` 如果 `b` 不在索引中，或索引无法完全覆盖该条件）
3. **filtered 值**：表示经过第2步过滤后，预计能保留的数据比例

- 当所有条件都能被索引覆盖时，filtered=100%（因为存储引擎已经完成了全部过滤）
- 当有非索引条件时，filtered会显示该条件的预估过滤效果

### Extra（重要）

这列包含了 MySQL 解析查询的额外信息，通过这些信息，可以更准确的理解 MySQL 到底是如何执行查询的。常见的值如下：

-   **Using filesort**：在排序时使用了外部的索引排序，没有用到表内索引进行排序。
-   **Using temporary**：MySQL 需要创建临时表来存储查询的结果，常见于 ORDER BY 和 GROUP BY。
-   **Using index**：表明查询使用了覆盖索引，不用回表，查询效率非常高。
-   **Using index condition**：表示查询优化器选择使用了索引条件下推这个特性。
-   **Using where**：表明查询使用了 WHERE 子句进行条件过滤。一般在没有使用到索引的时候会出现。
-   **Using join buffer (Block Nested Loop)**：连表查询的方式，表示当被驱动表的没有使用索引的时候，MySQL 会先将驱动表读出来放到 join buffer 中，再遍历被驱动表与驱动表进行查询。

这里提醒下，当 Extra 列包含 Using filesort 或 Using temporary 时，MySQL 的性能可能会存在问题，需要尽可能避免。