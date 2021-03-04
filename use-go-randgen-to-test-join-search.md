---
title: 使用 go-randgen 测试 join 查询
author: ['陈朋']
date: 2020-12-25
summary: 本文将向大家介绍在过去的几个月，使用 go-randgen 框架对 TiDB 进行测试。
tags: ['TiDB']
---

在数据库的查询中，join 是最常用的查询之一，由于 join 算法实现的复杂性，出现问题的概率较大，我们对 TiDB 中出现过的 join 问题进行分析，将易发生问题的场景归为如下几类 ：

1. 相同的 join 查询，join key 为不同的数据类型

2. 在分区表上进行 join 

3. 相同的 join 查询，不同的 join 实现算法

4. 特殊的查询条件

我们从这些场景入手，在过去的几个月，使用 go-randgen 框架对 TiDB 进行测试。下文将对go-randgen 测试框架，及我们测试工作中的相关内容进行详述，分为以下 4 部分展开:

- go-randgen 测试框架的简介

- 通过示例对 go-randgen 的使用方法进行介绍

- go-randgen 工具在 TiDB 测试中的实践及效果

- 未来可以基于 go-randgen 的进一步工作，以及其它相关工作的介绍 

## go-randgen 简介

[go-randgen](https://github.com/pingcap/go-randgen) 是一个完全可配置的测试框架，它允许创建随机数据集，并对其运行随机生成的查询，进而通过 A/B test 验证查询结果的正确性。

## go-randgen 使用示例

以 join 测试为例，go-randgen 相关语法格式可以参考 [https://github.com/pingcap/go-randgen](https://github.com/pingcap/go-randgen。我们使用)，分为 3 个步骤:

1. 定义 zz 文件，指定表的生成规则，如数据类型，表类型，行数等

2. 定义 yy 文件，指定随机 SQL 的生成规则

3. 使用生成的表结构和 SQL 运行 A/B test

下文通过示例，分别对每个步骤进行详述：

1.定义 join.zz.lua。该例中的 zz 文件，可以生成 6 张表，每张表中都有 17  个字段与 fileds.types 中定义的类型对应。这 6 张表分别是:

   - table_400_undef_undef_1（400 行数据）
    
   - table_400_undef_4_1（400 行数据且有 4 个分区）
    
   - table_300_undef_undef_1（300 行数据）
    
   - table_300_undef_4_1（300 行数据且 4 个分区）
    
   - table_290_undef_undef_1（290 行数据）
    
   - table_290_undef_4_1（290 行数据且 4 个分区）

```
tables = {
    rows = {400, 300, 290},
    partitions = {'undef', 4},
}

fields = {
    types = {'int', 'tinyint', 'smallint', 'bigint', 'decimal(40, 20)', 'float', 'double', 'char(20)', 'varchar(20)', 'enum', 'set', 'datetime', 'bool', 'bit(64)', 'timestamp', 'year', 'date'},
    keys = {'key'},
}

data = {
    numbers = {'null', 'tinyint', 'smallint',
        'decimal',
    },
    smallint = {null, 'smallint'},
    mediumint = {null, 'mediumint'},
    tinyint = {null, 'tinyint'},
    bool = {1, 0, null},
    year = {'null', 'year'},
    datetime = {'null', 'datetime'},
    timestamp = {'null', 'datetime'},
    date = {'null', 'date'},
    strings = {'null', 'letter', 'english'},
}
```

2.定义 join.yy 文件。该例中的 yy 文件，通过 hint 指定生成 inl_merge_join 和 inl_hash_join 算法查询语句。生成的 sql 语句中除指定字段外，查询条件中的表和字段将随机组合而成。

生成的的 SQL 示例：
```
SELECT /*+ inl_hash_join(t1) */ t1.pk, t2.pk from table_290_undef_undef_1 t1, table_400_undef_undef_1 t2 where t1. `col_enum_key_signed` = t2. `col_int_key_signed` and t1. `col_smallint_key_signed` < -5418830167423061551 order by t1.pk, t2.pk;
```

```
query:
    select

select:
    SELECT hint_begin inl_merge_join(t1, t2) */ col_list FROM _table  t1, _table t2 where condition and condition1 order by t1.pk, t2.pk;
    SELECT hint_begin inl_hash_join(t1) */ col_list from _table  t1, _table t2 where condition and condition1 order by t1.pk, t2.pk;
    
col_list:
    t1.pk, t2.pk

condition:
    t1. _field = t2. _field

condition1:
    t1. _field_int < _int

hint_begin:
    /*+
```

3.根据 join.zz.lua 和 join.yy 文件运行 A/B test。本例中，会对 TiDB 查询结果与 MySQL 查询结果进行对比 。查询结果不一致的 SQL，将会被记录在当前目录的 dump 子目录中。

```
./go-randgen exec -Z join.zz.lua -Y join.yy --dsn1 "root:password@tcp(127.0.0.1:3306)/test" --dsn2 "root:@tcp(127.0.0.1:4000)/test" -Q 2000
2020/12/25 16:37:18 Open DB ok, starting generate data in two db by ddls
2020/12/25 16:37:18 load zz from join.zz.lua
2020/12/25 16:37:20 generating data ok
2020/12/25 16:37:20 starting execute sqls generated by yy
2020/12/25 16:37:20 load yy from join.yy
2020/12/25 16:37:32 dump ok
```

## go-randgen 在 TiDB 测试中的实践

通过 go-randgen 对 TiDB 的 join 算法进行测试，我们目前已发现 10 个正确性相关的问题，例如：

- 通过对不同类型覆盖，进行列值比较。如：`select * from _table where _field > _field`。发现时间列和 year 列比较错误，记录在 [tidb/issues/20121](https://github.com/pingcap/tidb/issues/20121) 中。

- 对 distinct 语句进行测试。如： `select count(distinct(t1. _field)), count(distinct t1. _field, t1. _field)	from table_400_utf8_undef t1, table_290_utf8_undef t2 where t1. _field = t2. _field and t1. _field = t2. _field and t1. _field_int  != _int`。发现 distinct 计算错误，记录在 [tidb/issues/20237](https://github.com/pingcap/tidb/issues/20237) 中。

- 除了随机类型外，通过扩大单条语句的覆盖范围，随机组合语句，使 SQL 语句上下文具有关联性。如：`alter table _table add index {print(string.format("t%d", math.random(10,2000000)))} (_field); SELECT  t1.pk, t2.pk from t t1 left join t t2 on t1. _field = t2. _field  where t1. _field != _int order by t1.pk, t2.pk`。发现添加索引后，查询报错，记录在 [tidb/issues/20698](https://github.com/pingcap/tidb/issues/20698) 中。 

发现的这些 issue 提醒我们对 TiDB 质量要有敬畏之心，并且也印证了从过往发现的问题进行分析，归纳场景进而扩大测试点范围的方法是可行的。后续的 join 测试，也将继续覆盖更多的数据类型，尝试更多的语句组合、场景组合，例如在事务中添加数据、删除数据，再与 join 查询随机组合。

## 未来工作

我们可以通过不断完善 zz 和 yy 文件，来提高 join 测试覆盖面。但是通过 go-randgen 生成的 SQL 具有十分固定的结构，如果不能提前知道测试点，就不能构造出查询语句，进行有效的测试覆盖。那有没有方法随机生成 join 查询语句，进行测试呢？目前我们正在 [Horoscope](https://github.com/chaos-mesh/horoscope)（优化器检测工具）中实现随机生成 join 查询的功能。此外，我们参考了 Manuel Rigger 的 “[Testing Database Engines via Pivoted Query Synthesis](https://arxiv.org/pdf/2001.04174.pdf)” 论文中的思路，Horoscope 会在某些表中随机选择一行数据作为 pivot row 去构建查询，使得查询返回的结果会包含这些选择的行。

由于 join 查询的复杂性，join 测试会是一个长期但价值大的事情，除此之外，优化器测试、region 测试、TiDB 集群 chaos 测试以及事务测试也是非常重要并且价值大的工作。如果你有兴趣，欢迎使用 go-randgen 或其他工具对 TiDB 进行测试，对于发现的问题可以在 [GitHub](https://github.com/pingcap/tidb/issues) 创建 issue 反馈给我们。如果你有更好的测试方法、测试思路和测试工具，欢迎在 [TUG](https://asktug.com/) 中和我们进行交流。