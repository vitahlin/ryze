---
title: MySQL索引优化规则
slug: mysql-index-optimization-rules
categories:
  - Database
tags:
  - MySQL
desc: MySQL 索引是数据库查询优化的关键，合理的索引设计可以显著提高查询性能。本篇博客将介绍 MySQL 索引的优化规则，包括索引类型、最佳实践、常见误区以及如何使用 EXPLAIN 进行优化分析，帮助你构建高效的数据库查询方案。
featuredImage: https://sonder.vitah.me/featured/66ce95748a56ffdf2c755970ea6574c4.webp
---

## 示例表

```sql
DROP TABLE IF EXISTS employees;

CREATE TABLE
  `employees` (
    `id` int (11) NOT NULL AUTO_INCREMENT,
    `name` varchar(24) NOT NULL DEFAULT '' COMMENT '姓名',
    `age` int (11) NOT NULL DEFAULT '0' COMMENT '年龄',
    `position` varchar(20) NOT NULL DEFAULT '' COMMENT '职位',
    `hire_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '入职时间',
    PRIMARY KEY (`id`),
    KEY `idx_name_age_position` (`name`, `age`, `position`) USING BTREE
  ) ENGINE = InnoDB AUTO_INCREMENT = 4 DEFAULT CHARSET = utf8 COMMENT = '员工记录表';

INSERT INTO
  employees (name, age, position, hire_time)
VALUES
  ('LiLei', 22, 'manager', NOW ());

INSERT INTO
  employees (name, age, position, hire_time)
VALUES
  ('HanMeimei', 23, 'dev', NOW ());

INSERT INTO
  employees (name, age, position, hire_time)
VALUES
  ('Lucy', 23, 'dev', NOW ());
```

## 索引优化规则

### 1. 最左匹配原则

如果索引了多列，要遵守最左匹配原则。查询从索引最左前列开始并且不跳过索引中间的列。

### 2. 不在索引列上做操作

不在索引列上做任何操作 (计算、函数、类型转换 (自动/手动))，会导致索引失效而转向全表扫描。

```sql
EXPLAIN SELECT * FROM employees WHERE left(name, 3) = 'LiLei';
```

```json
{
  "id": 1,
  "select_type": "SIMPLE",
  "table": "employees",
  "partitions": null,
  "type": "ALL",
  "possible_keys": null,
  "key": null,
  "key_len": null,
  "ref": null,
  "rows": 99909,
  "filtered": 100,
  "Extra": "Using where"
}
```

### 3. 存储引擎不能使用索引中范围条件右边的列

#### 走索引示例

```sql
EXPLAIN SELECT * FROM employees
WHERE name = 'LiLei'
  AND age = 22
  AND position = 'manager';
```

```json
{
  "id": 1,
  "select_type": "SIMPLE",
  "table": "employees",
  "partitions": null,
  "type": "ref",
  "possible_keys": "idx_name_age_position",
  "key": "idx_name_age_position",
  "key_len": "140",
  "ref": "const,const,const",
  "rows": 1,
  "filtered": 100,
  "Extra": null
}
```

#### 不走索引示例

```sql
EXPLAIN SELECT * FROM employees
WHERE name = 'LiLei'
  AND age > 22
  AND position = 'manager';
```

```json
{
  "id": 1,
  "select_type": "SIMPLE",
  "table": "employees",
  "partitions": null,
  "type": "range",
  "possible_keys": "idx_name_age_position",
  "key": "idx_name_age_position",
  "key_len": "78",
  "ref": null,
  "rows": 1,
  "filtered": 10,
  "Extra": "Using index condition"
}
```

### 4. 尽量使用覆盖索引，减少 select *

#### 覆盖索引示例

```sql
EXPLAIN SELECT name, age
FROM employees
WHERE name = 'LiLei'
  AND age = 23
  AND position = 'manager';
```

```json
{
  "id": 1,
  "select_type": "SIMPLE",
  "table": "employees",
  "partitions": null,
  "type": "ref",
  "possible_keys": "idx_name_age_position",
  "key": "idx_name_age_position",
  "key_len": "140",
  "ref": "const,const,const",
  "rows": 1,
  "filtered": 100,
  "Extra": "Using index"
}
```

#### 非覆盖索引 select *

```sql
EXPLAIN
SELECT *
FROM employees
WHERE name = 'LiLei'
  AND age = 23
  AND position = 'manager';
```

```json
{
  "id": 1,
  "select_type": "SIMPLE",
  "table": "employees",
  "partitions": null,
  "type": "ref",
  "possible_keys": "idx_name_age_position",
  "key": "idx_name_age_position",
  "key_len": "140",
  "ref": "const,const,const",
  "rows": 1,
  "filtered": 100,
  "Extra": null
}
```

### 5. 使用!=, <>, not in, not exist 的时候不会用到索引

< 小于、 > 大于、 <=、>= 这些，mysql 内部优化器会根据检索比例、表大小等多个因素整体评估是否使用索引

```sql
EXPLAIN
SELECT *
FROM employees
WHERE name != 'LiLei';
```

```json
{
  "id": 1,
  "select_type": "SIMPLE",
  "table": "employees",
  "partitions": null,
  "type": "ALL",
  "possible_keys": "idx_name_age_position",
  "key": null,
  "key_len": null,
  "ref": null,
  "rows": 99909,
  "filtered": 50,
  "Extra": "Using where"
}
```

### 6. Is null, is not null 一般情况下也无法用到索引

```sql
EXPLAIN SELECT *
FROM employees
WHERE name IS NULL
```

```json
{
  "id": 1,
  "select_type": "SIMPLE",
  "table": null,
  "partitions": null,
  "type": null,
  "possible_keys": null,
  "key": null,
  "key_len": null,
  "ref": null,
  "rows": null,
  "filtered": null,
  "Extra": "Impossible WHERE"
}
```

### 7. Like 以通配符开头 ('$abc...')索引会失效

```sql
EXPLAIN SELECT * FROM employees
WHERE name LIKE '%Lei'
```

```json
{
  "id": 1,
  "select_type": "SIMPLE",
  "table": "employees",
  "partitions": null,
  "type": "ALL",
  "possible_keys": null,
  "key": null,
  "key_len": null,
  "ref": null,
  "rows": 99909,
  "filtered": 11.11,
  "Extra": "Using where"
}
```

#### 如何解决 `like '%abc%'` 索引失效问题？

##### 覆盖索引

使用覆盖索引，查询字段必须是建立覆盖索引字段，例如

```sql
EXPLAIN SELECT name, age, position
FROM employees
WHERE name LIKE '%lei%';
```

```json
{
  "id": 1,
  "select_type": "SIMPLE",
  "table": "employees",
  "partitions": null,
  "type": "index",
  "possible_keys": null,
  "key": "idx_name_age_position",
  "key_len": "140",
  "ref": null,
  "rows": 99909,
  "filtered": 11.11,
  "Extra": "Using where; Using index"
}
```

##### 搜索引擎

如果不能使用覆盖索引，则可借助搜索引擎，如 ES。

### 8. 字符串不加单引号会导致索引失效

```sql
EXPLAIN SELECT * FROM employees WHERE name = 1000;
```

```json
{
  "id": 1,
  "select_type": "SIMPLE",
  "table": "employees",
  "partitions": null,
  "type": "ALL",
  "possible_keys": "idx_name_age_position",
  "key": null,
  "key_len": null,
  "ref": null,
  "rows": 99909,
  "filtered": 10,
  "Extra": "Using where"
}
```

这其实是类型转换的问题

### 9. 少用 or 或者 in

少用 or 或 in，用它查询时，mysql**不一定使用索引**，mysql 内部优化器会根据检索比例、表大小等多个因素整体评估是否使用索引。例如：

```sql
EXPLAIN SELECT * FROM employees
WHERE name = 'LiLei'
   OR name = 'HanMeimei';
```

```json
{
  "id": 1,
  "select_type": "SIMPLE",
  "table": "employees",
  "partitions": null,
  "type": "ALL",
  "possible_keys": "idx_name_age_position",
  "key": null,
  "key_len": null,
  "ref": null,
  "rows": 3,
  "filtered": 66.67,
  "Extra": "Using where"
}
```

### 10. 范围查询

#### 是否会走索引？

先给年龄字段添加单值索引

```sql
ALTER TABLE `employees`
    ADD INDEX `idx_age` (`age`) USING BTREE;
```

```sql
EXPLAIN SELECT *
FROM employees
WHERE age >= 1
  AND age <= 2000;
```

```json
{
  "id": 1,
  "select_type": "SIMPLE",
  "table": "employees",
  "partitions": null,
  "type": "ALL",
  "possible_keys": null,
  "key": null,
  "key_len": null,
  "ref": null,
  "rows": 99909,
  "filtered": 11.11,
  "Extra": "Using where"
}
```

**没走索引原因：mysql 内部优化器会根据检索比例、表大小等多个因素整体评估是否使用索引。比如这个例子，可能是由于单次数据量查询过大导致优化器最终选择不走索引。**

#### 如何优化范围查询

优化方法：将大的范围拆分成多个小范围。

```sql
EXPLAIN SELECT *
FROM employees
WHERE age >= 1001
  AND age <= 2000;
```

```json
{
  "id": 1,
  "select_type": "SIMPLE",
  "table": "employees",
  "partitions": null,
  "type": "range",
  "possible_keys": "idx_age",
  "key": "idx_age",
  "key_len": "4",
  "ref": null,
  "rows": 1000,
  "filtered": 100,
  "Extra": "Using index condition"
}
```

### 11. 不要在小基数字段上建立索引

**索引基数是指这个字段在表里总共有多少个不同的值**，比如性别字段，值不是男就是女，该字段基数就是 2。
如果对这种小基数字段建立索引的话，还不如全表扫描来，因为你的索引树里面就包含男和女两种值，根本没办法快速的二分查找，那用索引就没有意义了。
一般建立索引，尽量使用那些基数比较大的字段，就是值比较多的字段，才能发挥出 B+树快速二分查找的优势。

### 12. 长字符串可以采用前缀索引

尽量对字段类型比较小的列设计索引，比如说 tinyint 之类的，因为字段类型比较小的话，占用磁盘空间也会比较小。那么当我们需要对 varhcar (255) 这种字段建立索引怎么办？
我们可以针对这个字段对前 20 个字符建立索引，就是说把这个字段里对每个值对前 20 个字符放在索引树里，如 index (name (20), age, position)。
此时在 where 条件搜索的时候，如果是根据 name 字段来搜索，就会先到索引树里根据 name 字段的前 20 个字符去搜索，定位到前 20 个字符的前缀匹配的部分数据之后，再回到聚簇索引提前出来完整的 name 字段值进行比对。
但是如果要是 order by name，那么此时 name 在索引树里面仅仅包含前 20 个字符，所以这个排序是没法用上索引的，group by 也是同理。

### 13. Where 与 order by 冲突时优先 where

Where 与 order by 出现索引设计冲突时，到底是根据 where 去设计，还是针对 order by 去设计索引？
一般都是让 where 条件去使用索引来快速筛选一部分指定顺序，接着再进行排序。
大多数情况基于索引进行 where 筛选可以更快速度筛选你要的少部分数据，然后做排序的成本会小很多。

### 14. 基于慢查询 sql 做优化

什么是慢查询？
> MySQL 的慢查询，全名是慢查询日志，是 MySQL 提供的一种日志记录，用来记录在 MySQL 中响应时间超过阀值的语句。
> 具体环境中，运行时间超过 long_query_time 值的 SQL 语句，则会被记录到慢查询日志中。
> Long_query_time 的默认值为 10，意思是记录运行 10 秒以上的语句。
> 默认情况下，MySQL 数据库并不启动慢查询日志，需要手动来设置这个参数。
> 当然，如果不是调优需要的话，一般不建议启动该参数，因为开启慢查询日志会或多或少带来一定的性能影响。

## 索引使用总结

假设索引 `index(a,b,c)`

| where 语句 | 索引是否被使用 |
| --- | --- |
| where a=3 | 有，使用到了 a |
| where a=3 and b=5 | 有，使用到了 a, b |
| where a=3 and b=4 and c=5 | 有，使用到了 a, b, c |
| where b=3 | 没有 |
| where b=3 and c=4 | 没有 |
| where c=4 | 没有 |
| where a=3 and c=5 | 有，但只使用了 a |
| where a=3 and b>4 and c=5 | 有，使用了 a, b，c 不能用在范围之后，b 之后断了 |
| where a=3 and b like 'kk%' and c=4 | 有，使用到了 a, b, c |
| where a=3 and b like '%kk' and c=4 | 有，只用到了 a |
| where a=3 and b like '%kk%' and c=4 | 有，只用到了 a |
| where a=3 and b like 'k%kk%' and c=4 | 有，使用到了 a, b, c |

`like 'kk%'` 相当于等于常量，`%kk` 和 `%kk%` 相当于范围。
