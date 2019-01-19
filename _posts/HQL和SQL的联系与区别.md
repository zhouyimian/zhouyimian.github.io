---
layout:     post
title:      HQL和SQL的联系与区别
subtitle:   HQL和SQL
date:       2019-01-19
author:     BY KiloMeter
header-img: img/charlotte/charlotte.jpg
catalog: true
tags:
    - 大数据
    - Hive
---


####   联系

>  Hive提供了HQL(Hive查询语言)，语法和SQL语句相似，可以用于查询Hadoop集群中的数据，HQL在底层转   换成了MR程序，方便开发人员和数据库管理员进行管理。

  #### 区别

> * 一般的SQL语句在查询方面所花费的时间都是按秒来计的，而由于HQL查询的底层是MR程序，MR程序主要是批处理，所以运行时间比较长，即使数据量很少，也需要花费不少时间，因此HQL不适用于实时查询，比较适合于静态数据分析，不需要快速响应给出结果，而且数据本身不会频繁发生变化的场景。
> * 由于Hive是基于Hadoop以及HDFS的设计而产生的，因此Hive所能完成的工作也受到了限制，其中最大的限制就是Hive不支持记录级别的更新、插入或者删除
> * 此外，Hive不支持事务
