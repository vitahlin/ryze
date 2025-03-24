---
title: MySQL单表查询如何计算成本
slug: mysql-single-table-query-cost
categories:
  - Database
tags:
  - MySQL
featuredImage: https://sonder.vitah.me/ryze/62008f9dcf20b813ec036e808a1502ed.webp
desc: 在 MySQL 中，索引的选择和使用直接影响查询的执行成本。本文将详细介绍如何计算索引查询的代价，包括范围区间数量、回表次数、索引选择、排序代价、扫描行数等关键因素。通过 EXPLAIN 和 TRACE 命令，我们可以深入分析 SQL 查询的执行过程，理解 MySQL 优化器的决策逻辑，并针对索引选择、查询优化提供实用的优化策略，帮助开发者提升数据库性能。
---

## 问题

1. 什么是查询成本
2. 如何知道 MySQL 的查询成本
3. 单表查询的全表扫描成本如何计算
4. 单表查询的使用索引的成本如何计算

## 示例SQL数据

示例的数据：[https://gist.github.com/vitahlin/ee6419991bb58cde242ccc856b84715c](https://gist.github.com/vitahlin/ee6419991bb58cde242ccc856b84715c)

## 什么是查询成本

MySQL 一条查询语句的执行成本由2部分组成：
- I/O成本
- CPU成本

什么是I/O成本？
> 平常数据和索引都是存储到磁盘上的，当想查询表中的记录时，需要先把数据或者索引加载到内存中然后再操作。从**磁盘加载到内存损耗的时间称之为I/O成本**。

什么是CPU成本？
> 读取、检测记录是否满足对应的搜索条件、对结果集进行排序等操作损耗大时间是CPU成本。对于InnoDB存储引擎来说，页是磁盘和内存交互的基本单位，MySQL规定**读取一个页的成本是1.0，读取以及检测一条记录是否符合搜索条件的成本是0.2。**

注意，**不论读取记录时需不需要检测是否满足搜索条件，其成本都是0.2**。

## 如何知道MySQL计算的查询成本

我们执行如下命令来查看全表扫描的成本：

```sql
set session optimizer_trace = "enabled=on";

select *
from order_exp
where order_no in ('DD00_6S', 'DD00_9S', 'DD00_10S')
  and expire_time > '2021-03-22 18:28:28'
  and expire_time <= '2021-03-22 18:35:09'
  and insert_time > order_exp.expire_time
  and order_note like '%7排1%'
  and order_status = 0;
  
select *
from information_schema.optimizer_trace;
```

分析结果如下：

```json
{
  "steps": [
    {
      "join_preparation": {
        "select#": 1,
        "steps": [
          {
            "IN_uses_bisection": true
          },
          {
            "expanded_query": "/* select#1 */ select `order_exp`.`id` AS `id`,`order_exp`.`order_no` AS `order_no`,`order_exp`.`order_note` AS `order_note`,`order_exp`.`insert_time` AS `insert_time`,`order_exp`.`expire_duration` AS `expire_duration`,`order_exp`.`expire_time` AS `expire_time`,`order_exp`.`order_status` AS `order_status` from `order_exp` where ((`order_exp`.`order_no` in ('DD00_6S','DD00_9S','DD00_10S')) and (`order_exp`.`expire_time` > '2021-03-22 18:28:28') and (`order_exp`.`expire_time` <= '2021-03-22 18:35:09') and (`order_exp`.`insert_time` > `order_exp`.`expire_time`) and (`order_exp`.`order_note` like '%71%') and (`order_exp`.`order_status` = 0))"
          }
        ] /* steps */
      } /* join_preparation */
    },
    {
      "join_optimization": {
        "select#": 1,
        "steps": [
          {
            "condition_processing": {
              "condition": "WHERE",
              "original_condition": "((`order_exp`.`order_no` in ('DD00_6S','DD00_9S','DD00_10S')) and (`order_exp`.`expire_time` > '2021-03-22 18:28:28') and (`order_exp`.`expire_time` <= '2021-03-22 18:35:09') and (`order_exp`.`insert_time` > `order_exp`.`expire_time`) and (`order_exp`.`order_note` like '%71%') and (`order_exp`.`order_status` = 0))",
              "steps": [
                {
                  "transformation": "equality_propagation",
                  "resulting_condition": "((`order_exp`.`order_no` in ('DD00_6S','DD00_9S','DD00_10S')) and (`order_exp`.`expire_time` > '2021-03-22 18:28:28') and (`order_exp`.`expire_time` <= '2021-03-22 18:35:09') and (`order_exp`.`insert_time` > `order_exp`.`expire_time`) and (`order_exp`.`order_note` like '%71%') and multiple equal(0, `order_exp`.`order_status`))"
                },
                {
                  "transformation": "constant_propagation",
                  "resulting_condition": "((`order_exp`.`order_no` in ('DD00_6S','DD00_9S','DD00_10S')) and (`order_exp`.`expire_time` > '2021-03-22 18:28:28') and (`order_exp`.`expire_time` <= '2021-03-22 18:35:09') and (`order_exp`.`insert_time` > `order_exp`.`expire_time`) and (`order_exp`.`order_note` like '%71%') and multiple equal(0, `order_exp`.`order_status`))"
                },
                {
                  "transformation": "trivial_condition_removal",
                  "resulting_condition": "((`order_exp`.`order_no` in ('DD00_6S','DD00_9S','DD00_10S')) and (`order_exp`.`expire_time` > '2021-03-22 18:28:28') and (`order_exp`.`expire_time` <= '2021-03-22 18:35:09') and (`order_exp`.`insert_time` > `order_exp`.`expire_time`) and (`order_exp`.`order_note` like '%71%') and multiple equal(0, `order_exp`.`order_status`))"
                }
              ] /* steps */
            } /* condition_processing */
          },
          {
            "substitute_generated_columns": {
            } /* substitute_generated_columns */
          },
          {
            "table_dependencies": [
              {
                "table": "`order_exp`",
                "row_may_be_null": false,
                "map_bit": 0,
                "depends_on_map_bits": [
                ] /* depends_on_map_bits */
              }
            ] /* table_dependencies */
          },
          {
            "ref_optimizer_key_uses": [
            ] /* ref_optimizer_key_uses */
          },
          {
            "rows_estimation": [
              {
                "table": "`order_exp`",
                "range_analysis": {
                  "table_scan": {
                    "rows": 10579,
                    "cost": 2214.9
                  } /* table_scan */,
                  "potential_range_indexes": [
                    {
                      "index": "PRIMARY",
                      "usable": false,
                      "cause": "not_applicable"
                    },
                    {
                      "index": "u_idx_day_status",
                      "usable": false,
                      "cause": "not_applicable"
                    },
                    {
                      "index": "idx_order_no",
                      "usable": true,
                      "key_parts": [
                        "order_no",
                        "id"
                      ] /* key_parts */
                    },
                    {
                      "index": "idx_expire_time",
                      "usable": true,
                      "key_parts": [
                        "expire_time",
                        "id"
                      ] /* key_parts */
                    }
                  ] /* potential_range_indexes */,
                  "setup_range_conditions": [
                  ] /* setup_range_conditions */,
                  "group_index_range": {
                    "chosen": false,
                    "cause": "not_group_by_or_distinct"
                  } /* group_index_range */,
                  "analyzing_range_alternatives": {
                    "range_scan_alternatives": [
                      {
                        "index": "idx_order_no",
                        "ranges": [
                          "DD00_10S <= order_no <= DD00_10S",
                          "DD00_6S <= order_no <= DD00_6S",
                          "DD00_9S <= order_no <= DD00_9S"
                        ] /* ranges */,
                        "index_dives_for_eq_ranges": true,
                        "rowid_ordered": false,
                        "using_mrr": false,
                        "index_only": false,
                        "rows": 58,
                        "cost": 72.61,
                        "chosen": true
                      },
                      {
                        "index": "idx_expire_time",
                        "ranges": [
                          "0x99a92d271c < expire_time <= 0x99a92d28c9"
                        ] /* ranges */,
                        "index_dives_for_eq_ranges": true,
                        "rowid_ordered": false,
                        "using_mrr": false,
                        "index_only": false,
                        "rows": 39,
                        "cost": 47.81,
                        "chosen": true
                      }
                    ] /* range_scan_alternatives */,
                    "analyzing_roworder_intersect": {
                      "usable": false,
                      "cause": "too_few_roworder_scans"
                    } /* analyzing_roworder_intersect */
                  } /* analyzing_range_alternatives */,
                  "chosen_range_access_summary": {
                    "range_access_plan": {
                      "type": "range_scan",
                      "index": "idx_expire_time",
                      "rows": 39,
                      "ranges": [
                        "0x99a92d271c < expire_time <= 0x99a92d28c9"
                      ] /* ranges */
                    } /* range_access_plan */,
                    "rows_for_plan": 39,
                    "cost_for_plan": 47.81,
                    "chosen": true
                  } /* chosen_range_access_summary */
                } /* range_analysis */
              }
            ] /* rows_estimation */
          },
          {
            "considered_execution_plans": [
              {
                "plan_prefix": [
                ] /* plan_prefix */,
                "table": "`order_exp`",
                "best_access_path": {
                  "considered_access_paths": [
                    {
                      "rows_to_scan": 39,
                      "access_type": "range",
                      "range_details": {
                        "used_index": "idx_expire_time"
                      } /* range_details */,
                      "resulting_rows": 39,
                      "cost": 55.61,
                      "chosen": true
                    }
                  ] /* considered_access_paths */
                } /* best_access_path */,
                "condition_filtering_pct": 100,
                "rows_for_plan": 39,
                "cost_for_plan": 55.61,
                "chosen": true
              }
            ] /* considered_execution_plans */
          },
          {
            "attaching_conditions_to_tables": {
              "original_condition": "((`order_exp`.`order_status` = 0) and (`order_exp`.`order_no` in ('DD00_6S','DD00_9S','DD00_10S')) and (`order_exp`.`expire_time` > '2021-03-22 18:28:28') and (`order_exp`.`expire_time` <= '2021-03-22 18:35:09') and (`order_exp`.`insert_time` > `order_exp`.`expire_time`) and (`order_exp`.`order_note` like '%71%'))",
              "attached_conditions_computation": [
              ] /* attached_conditions_computation */,
              "attached_conditions_summary": [
                {
                  "table": "`order_exp`",
                  "attached": "((`order_exp`.`order_status` = 0) and (`order_exp`.`order_no` in ('DD00_6S','DD00_9S','DD00_10S')) and (`order_exp`.`expire_time` > '2021-03-22 18:28:28') and (`order_exp`.`expire_time` <= '2021-03-22 18:35:09') and (`order_exp`.`insert_time` > `order_exp`.`expire_time`) and (`order_exp`.`order_note` like '%71%'))"
                }
              ] /* attached_conditions_summary */
            } /* attaching_conditions_to_tables */
          },
          {
            "refine_plan": [
              {
                "table": "`order_exp`",
                "pushed_index_condition": "((`order_exp`.`expire_time` > '2021-03-22 18:28:28') and (`order_exp`.`expire_time` <= '2021-03-22 18:35:09'))",
                "table_condition_attached": "((`order_exp`.`order_status` = 0) and (`order_exp`.`order_no` in ('DD00_6S','DD00_9S','DD00_10S')) and (`order_exp`.`insert_time` > `order_exp`.`expire_time`) and (`order_exp`.`order_note` like '%71%'))"
              }
            ] /* refine_plan */
          }
        ] /* steps */
      } /* join_optimization */
    },
    {
      "join_execution": {
        "select#": 1,
        "steps": [
        ] /* steps */
      } /* join_execution */
    }
  ] /* steps */
}
```

查看上述扫描成本，全表扫描成本如下，成本为2214.9：

```json
{
    "table_scan": {
        "rows": 10579,
        "cost": 2214.9
    }
}
```

索引 `idx_order_no` 的成本为如下，为72.61

```json
{
    "index": "idx_order_no",
    "ranges": [
        "DD00_10S <= order_no <= DD00_10S",
        "DD00_6S <= order_no <= DD00_6S",
        "DD00_9S <= order_no <= DD00_9S"
    ],
    "index_dives_for_eq_ranges": true,
    "rowid_ordered": false,
    "using_mrr": false,
    "index_only": false,
    "rows": 58,
    "cost": 72.61,
    "chosen": true
}
```

索引 `idx_expire_time` 成本如下，为47.81

```json
{
    "index": "idx_expire_time",
    "ranges": [
        "0x99a92d271c < expire_time <= 0x99a92d28c9"
    ],
    "index_dives_for_eq_ranges": true,
    "rowid_ordered": false,
    "using_mrr": false,
    "index_only": false,
    "rows": 39,
    "cost": 47.81,
    "chosen": true
}
```

可以看到成本排序如下：

全表扫描 2214.9 < 索引 idx_order_no 72.61 < 索引 idx_expire_time 47.81

然后执行 `explain` 语句，看执行器最终的方案是什么：

```sql
explain
select *
from order_exp
where order_no in ('DD00_6S', 'DD00_9S', 'DD00_10S')
  and expire_time > '2021-03-22 18:28:28'
  and expire_time <= '2021-03-22 18:35:09'
  and insert_time > order_exp.expire_time
  and order_note like '%7排1%'
  and order_status = 0;
```

结果如下：
```json
{
    "id": 1,
    "select_type": "SIMPLE",
    "table": "order_exp",
    "partitions": null,
    "type": "range",
    "possible_keys": "idx_order_no,idx_expire_time",
    "key": "idx_expire_time",
    "key_len": "5",
    "ref": null,
    "rows": 39,
    "filtered": 0.13,
    "Extra": "Using index condition; Using where"
}
```

发现，最终优化器选择了 索引 `idx_expire_time` 作为最终的执行计划，因为它的成本最低。 

## 单表查询的成本

MySQL 检测查询成本的过程如下：
1. 根据搜索条件，找出所有可能使用的索引
2. 计算全表扫描的代价
3. 计算不同索引执行查询的成本
4. 对比各种方案，找出成本最低的

我们以上述中的查询语句为例，依次分析全表扫描和各个可能使用的索引的查询成本。

### 计算全表扫描的成本

对于InnoDB存储引擎，全表扫描就是把聚簇索引中的记录都依次和给定的搜索条件做一下比较，把符合搜索条件的记录加入到结果集，所以需要将聚簇索引对应的页面加载到内存中，然后再检测记录是否符合搜索条件。

所以计算全表扫描的代价需要两个信息：

1. 聚簇索引占用的页面数
2. 该表中的记录数

那么这两数据从哪里来？可以查看 MySQL 表的统计信息：

```sql
show table status like 'order_exp';

{
    "Name": "order_exp",
    "Engine": "InnoDB",
    "Version": 10,
    "Row_format": "Dynamic",
    "Rows": 10579,
    "Avg_row_length": 150,
    "Data_length": 1589248,
    "Max_data_length": 0,
    "Index_length": 1179648,
    "Data_free": 4194304,
    "Auto_increment": 10819,
    "Create_time": "2022-06-22 09:22:30",
    "Update_time": "2022-06-22 10:08:16",
    "Check_time": null,
    "Collation": "utf8_general_ci",
    "Checksum": null,
    "Create_options": "row_format=DYNAMIC",
    "Comment": ""
}
```

#### Rows

表示表中的记录条数，对于MyISAM存储引擎来说，这个值是准确的，对于InnoDB存储引擎来说，这个值是估计值。

#### Data_length

表示表占用的存储空间字节数。对于MyISAM来说，该值就是数据文件的大小。对于InnoDB来说，该值就相当于聚簇索引占用的存储空间大小，也就是说：**Data_length = 聚簇索引的页面数量*每页大小**

默认的页面大小为16k，可以计算聚簇索引数量：聚簇索引数量=1589248/16/1024=97

这样就可以计算 `I/O成本 = 97 * 1.0 + 1.1 =98.1`，
- 97是聚簇索引占用页面数
- 1.0指加载一个页面的 I/O 成本常数
- 后面的1.1是一个微调值（微调值是硬编码在代码里的，没有注释且值比较小，不做具体意义分析）

`CPU成本 = 10579 * 0.2 + 1.0 = 2116.8` 
- 10579是表中记录数
- 0.2是访问一条记录所需的CPU成本常数
- 后面是1.0是微调值。

`总成本 = I/O成本 + CPU成本 = 98.1 + 2116.8 = 2214.9`

全表扫描成本公式如下：
$$
\text{全表扫描成本} = \left( \frac{\text{Data\_length}}{16 \times 1024} \times 1.0 + 1.1 \right) + \left( \text{Rows} \times 0.2 + 1.0 \right)
$$

> 表中的记录其实都存储在聚簇索引对应B+树的叶子节点，所以只要通过根节点获得了最左边的叶子节点，就可以沿着叶子节点 组成的双向链表把所有记录都查看一遍。也就是全表扫描这个过程有的B+树非叶子节点是不需要访问的，但是Mysql在计算全表扫描成本时直接使用聚簇索引占用的页面数作为计算I/O成本的依据，是不区分叶子节点和非叶子节点的。

## 计算索引执行查询的代价

### 使用 idx_expire_time 索引执行查询的成本分析
搜索条件是  

```sql
expire_time > '2021-03-22 18:28:28'  and expire_time <= '2021-03-22 18:35:09'
```

就是这个范围区间是 `(2021-03-22 18:28:28, '2021-03-22 18:35:09)`。使用 `idx_expire_time` 会使用**二级索引+回表**的方式查询，计算这种查询成本依赖两个方面的数据:

#### 范围区间数量

不论某个范围区间的二级索引到底占用了多少页面，查询优化器认为读取索引的一个范围区间的 I/O成本和读取一个界面是相同的。
本例中 idx_expire_time 的范围区间只有一个，所以相当于访问这个范围区间的二级索引付出的I/O成本是：
$$
\text{范围区间数量成本} = 1 \times 1.0 = 1.0
$$

#### 需要回表的记录数
优化器需要计算二级索引的某个范围区间到底包含多少条记录，对于本例来说就是要计算 idx_expire_time 在 (2021-03-22 18:28:28, 2021-03-22 18:35:09) 这个范围区间中包含多少二级索引记录，计算过程是这样的：

1. 根据访问 idx_expire_time索引访问 expire_time>2021-03-22 18:28:28这个条件的第一条记录，称为区间最左记录。在B+树中定位第一条记录的过程是很快的，是常数级别的，所以这个过程的性能消耗是可以忽略不计的；
2. 再根据 expire_time<= 2021-03-22 18:28:28 这个条件继续从索引中找最后一条满足这个条件的记录，称为区间最右记录，这个过程的性能消耗也可以忽略不计。
3. 如果区间最左记录和区间最右记录相隔不远（不大于10个页面），那就可以精确统计出满足这个条件的二级索引记录条数。否则，只沿着区间最左记录向右读10个页面，计算平均每个页面中包含多少记录，然后这个用这个平均值乘以区间最左记录和最右记录之间的页面数量就可以了。

##### 计算回表记录数
那么怎么估算区间最左记录和区间最右记录有多少个页面呢？

> 我们假设区间最左记录在页b中，区间最右记录在页c中，那么我们想计算区间最左记录和区间最右记录之间的页面数量就相当于计算页b和页c之间有多少页面，而它们父节点中记录的每一条目录项记录都对应一个数据页，所以计算页b和页c之间有多少页面就相 当于计算它们父节点(也就是页a)中对应的目录项记录之间隔着几条记录。在一个页面中统计两条记录之间有几条记录的成本就很小了。
>
> 不过还有问题，如果页b和页c之间的页面实在太多，以至于页b和页c对应的目录项记录 都不在一个父页面中怎么办?既然是树，那就继续递归，之前我们说过一个B+树有4层高已经很了不得了，所以这个统计过程也不是很耗费性能。
>

我们查看上述区间的记录数量：

```sql
explain
select *
from order_exp
where expire_time > '2021-03-22 18:28:28'
  and expire_time <= '2021-03-22 18:35:09';
  
{
    "id": 1,
    "select_type": "SIMPLE",
    "table": "order_exp",
    "partitions": null,
    "type": "range",
    "possible_keys": "idx_expire_time",
    "key": "idx_expire_time",
    "key_len": "5",
    "ref": null,
    "rows": 39,
    "filtered": 100,
    "Extra": "Using index condition"
}
```

可以看到 idx_expire_time 在时间区间内大约有39条记录。

读取这39条记录的：`CPU成本=39*0.2+0.01=7.81`

39是需要读取的二级索引记录条数，0.2是读取一条记录的成本常量，0.01是微调值。

##### 根据记录的主键值到聚簇索引中做回表操作
Mysql评估回表操作的I/O成本很简单，它认为每次回表操作相当于访问一个页面，页，就是二级索引范围区间内有多少记录就需要进行多少次回表操作。所以这里的回表操作带来的I/O成本是：

> 回表I/O成本=39 * 1.0=39.0
> 39是预计的二级索引记录数，1.0是一个页面的I/O成本常数

##### 回表操作后得到完整记录，再检测其他搜索条件是否成立
回表操作的本质是通过二级索引记录的主键值到聚簇索引中找到完成的用户信息，然后再检测其他条件是否成立。

通过范围区间获取到的二级索引记录是39条，就是对应聚簇索引中39条完整的用户记录，读取并检测这些记录的是否符合其他搜索条件的CPU成本如下：

> CPU成本=39*0.2=7.8
>
> 0.2是检测一条记录是否符合给定搜索条件的成本常数。
>

所以，idx_expire_time执行查询的成本就如下所示：

> I/O成本=1.0+39*10.=40（范围区间数量+预估的二级索引记录条数）
>
> CPU成本=39*0.2+0.01+39*0.2=15.61（读取二级索引记录的成本+读取并检测回表后聚簇索引记录的成本）
>
> 总成本=40+15.61=55.61
>

这里我们计算出来是55.61，但是实际上Mysql计算出来的成本是47.81，为什么呢？

> MySQL 在实际计算中，和全文扫描比较成本时，使用索引的成本会去除读取并检测回表后聚簇索引记录的成本。55.61-47.81=7.8
>

### 使用索引 idx_order_no

#### 范围区间数量
查询条件是 `order_no in ('DD00_6S', 'DD00_9S', 'DD00_10S')`，即有3个单点区间，所以成本如下： 
$$
\text{范围区间数量成本} = 3 \times 1.0 = 3.0
$$

#### 需要回表记录数
```sql
explain
select *
from order_exp
where expire_time > '2021-03-22 18:28:28'
  and expire_time <= '2021-03-22 18:35:09';
  
{
    "id": 1,
    "select_type": "SIMPLE",
    "table": "order_exp",
    "partitions": null,
    "type": "range",
    "possible_keys": "idx_order_no",
    "key": "idx_order_no",
    "key_len": "152",
    "ref": null,
    "rows": 58,
    "filtered": 100,
    "Extra": "Using index condition"
}
```

_**rows=58**_
$$
\text{读取二级索引记录 CPU 成本} = 58 \times 0.2 + 0.1 = 11.61
$$
$$
\text{回表操作 I/O 成本} = 58 \times 1.0 = 58
$$

#### 回表后再比较成本

回表后得到完成的记录，然后再比较其他搜索条件是否成立。
$$
\text{CPU 成本} = 58 \times 0.2 = 11.6
$$

#### 计算总成本

$$
\text{总成本} = \text{范围区间数量成本 (I/O)} + \text{读取二级索引记录成本 (CPU)} + \text{回表操作成本 (I/O)} + \text{回表后再比较其他条件 (CPU)} 
$$
$$
= 3.0 + 11.61 + 58 + 11.6 = 84.21
$$

## 总结

$$
\text{全表扫描成本} = \left( \frac{\text{Datalength}}{16 \times 1024} \times 1.0 + 1.1 \right) + \left( \text{Rows} \times 0.2 + 1.0 \right)
$$

$$
\text{索引查询成本} = \text{范围区间数量} \times 1 + (\text{Rows} \times 0.2 + 0.1) + (\text{Rows} \times 1) + (\text{Rows} \times 0.2)
$$
