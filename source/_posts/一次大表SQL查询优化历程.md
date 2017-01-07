---
title: 一次大表SQL查询优化历程
date: 2017-01-07 10:56:56
tags: MySQL
---

本文主要介绍了一次查询SQL查询性能优化的整个过程。

## 1 背景  
有一张存储了订单数据的单表，为了防止表过大，需要删除历史的已达到终态的订单，只保留近期的数据以及历史的未达到终态的订单数据。这张表trade_order的大致结构是这样的（内容有缩减）：

```
CREATE TABLE "trade_order" (
  "id" int(10) unsigned NOT NULL AUTO_INCREMENT,
  "order_no" varchar(30) NOT NULL DEFAULT '',
  "update_time" datetime DEFAULT NULL COMMENT '订单更新时间',
  "book_time" datetime DEFAULT NULL COMMENT '预订时间',
  "gmt_create_time" datetime NOT NULL COMMENT 'DB数据创建时间',
  "gmt_modify_time" datetime NOT NULL COMMENT 'DB数据更新时间',
  "version" tinyint(4) unsigned NOT NULL DEFAULT '0' COMMENT '乐观锁',
  PRIMARY KEY ("id"),
  UNIQUE KEY "order_no" ("order_no"),
  KEY "update_time_state" ("update_time","state"),
  KEY "idx_stbyway_bktime" ("state","buy_way","book_time","id")
)


```

数据量：  
500-800w条左右。

我们需要删除更新时间在某个阈值之前，并且已达到终态（终态对应的状态有2个）的订单数据。我们不能直接这么做：

```
delete from trade_order where update_time < #{updateTimeThreshold} and (state = 99 or state = 100).
```

原因是：  

- update_time并不是唯一索引，这条SQL会加GAP锁。
- 由于数据量比较大，会被kill。

因此只能先查询出满足条件的订单，然后依据订单id（主键）去执行删除操作。重点来了，如果查询SQL设计的不合理，就会导致效率低下被kill。

## 2 SQL查询优化过程  
### 2.1 最原始的SQL查询  
为了记录删除了哪些订单，因此我们需要记录被删除的订单的订单号。在查询的时候不仅需要查询出id，还需要查询出orderNo。    
我们批量查询，每次查询一批（比如1000条），删除这1000条数据，循环执行，直到查询到的满足条件的订单列表为空。  
最简单的满足功能的查询语句：  

```
select id,order_no from trade_order where update_time < #{updateTimeThreshold} and (state = 99 or state = 100) order by id limit 1000;
```

以及：

```
select id,order_no from trade_order where update_time < #{updateTimeThreshold} and state in (99,100) limit 1000;
```

关于in和or二者的性能对比，查阅了一些资料，in的性能要优于or。原因是，如果要判断的值是常量，则in可以通过排序然后二分查找。具体可以参考文末的参考资料。  
我们可以使用mysql的explain命令来查看二者的区别：  

![explain-or VS in](http://o8sltkx20.bkt.clouddn.com/mysql-query-optimize-001-001.jpeg)

我们发现extra处：or是Using where，而in是Using index condition。  
> Using where:  
> mysql从存储引擎收到行后再进行“后过滤（Post-filter）”。后过滤：先读取整行数据，再检查是否符合where的条件，符合就留下，不符合便丢弃。检测是在读取行后进行的，所以叫后过滤。  
> Using index:  
> 此查询使用了覆盖索引（Convering Index），即通过索引就能返回结果，无需访问表。弱没显示“Using Index”表示读取了表数据。  
> Using index where:  
> 这里涉及到mysql的[Index Condition Pushdown (ICP)](http://dev.mysql.com/doc/refman/5.7/en/index-condition-pushdown-optimization.html)优化机制。当开启了ICP优化机制后，如果where语句中的部分判断能够仅仅依据索引字段来判断，那么mysql server会将这部分判断下沉到存储引擎层面。存储引擎会依据索引来做判断，如果满足才会去读表中的行。ICP能够减少存储引擎访问基础表的次数以及mysql访问存储引擎的次数。  

可以发现Using index where优于Using where，因此in优于or。

发现执行了一段时间后就被系统kill了。

### 2.2 采用union all替代in/or  
in以及or经常导致mysql放弃使用索引而扫全表，如果两个判断不会产生重复的数据，可以考虑拆成union all：

```
select id,order_no from trade_order where state = 99 and update_time < #{updateTimeThreshold} limit 1000   
union all   
select id,order_no from trade_order where state = 100 and update_time < #{updateTimeThreshold} limit 1000;
```

发现结果依然是运行一段时间就被kill了。

主要原因还是因为使用的是Using where，导致需要扫描很多行数据：  
![explain-union all](http://o8sltkx20.bkt.clouddn.com/mysql-query-optimize-001-002.jpeg)

### 2.3 延迟关联(deferred join/delayed JOIN)  
前面的几步中的SQL都是同时取出id和orderNo，而id是主键，但是orderNo不是主键，这种情况是使用延迟关联的好机会:

```
SELECT  a.id,order_no  FROM  trade_order a,   
(select  id  from  trade_order where  state = 100 and update_time < ‘2016-12-22 22:00:00' limit 1000) b    
WHERE  a.id = b.id;
```
  
何谓"延迟关联" ：通过使用<b>覆盖索引(covering index)</b>查询返回需要的主键,再根据主键关联原表获得需要的数据。  

> 何为“覆盖索引（covering index）”  
> A covering index refers to the case when all fields selected in a query are covered by an index, in that case InnoDB (not MyISAM) will never read the data in the table, but only use the data in the index, significantly speeding up the select.  
> Note that in InnoDB the primary key is included in all secondary indexes, so in a way all secondary indexes are compound indexes.

我们可以用explain查看下上述SQL的执行计划：  
![explain-union all](http://o8sltkx20.bkt.clouddn.com/mysql-query-optimize-001-003.jpeg)

结果：  
较前几步的性能有了非常大的提高：一次性删除了359万条数据才最终被kill。  
![explain-deferred join](http://o8sltkx20.bkt.clouddn.com/mysql-query-optimize-001-004.png)

SQL最终还是没有执行成功，还需继续优化。

### 2.4 时间段拆分  
通常我们写sql要注意一点是少用>、<。但是这里不可避免。能做的只能增加一个时间段下限判断，减少扫描的行的数量。  
我们将时间段划分为多个区间，每个区间长度是1个月，同时将2个状态分开处理，这样顺利的删除了表中所有的过期数据。

```
    <select id="getExpiredCloseStateOrder" resultMap="OrderResultMap">
        SELECT
        a.id,order_no
        FROM
        <include refid="tableName"/> a,
        (select
        id
        from
        <include refid="tableName"/>
        where
        <![CDATA[
            state = 99 and update_time >= #{updateTimeStart} and update_time < #{updateTimeEnd} limit #{limitSize}
        ]]> ) b
        WHERE
        a.id = b.id
    </select>

    <select id="getExpiredSuccessStateOrder" resultMap="OrderResultMap">
        SELECT
        a.id,order_no
        FROM
        <include refid="tableName"/> a,
        (select
        id
        from
        <include refid="tableName"/>
        where
        <![CDATA[
            state = 100 and update_time >= #{updateTimeStart} and update_time < #{updateTimeEnd} limit #{limitSize}
        ]]> ) b
        WHERE
        a.id = b.id
    </select>
```

## 3 参考资料
- [Subject-IN vs. OR on performance](http://lists.mysql.com/mysql/216945)
- [stackoverflow-mysql-or-vs-in-performance](http://stackoverflow.com/questions/782915/mysql-or-vs-in-performance)
- [push down optimization](https://dev.mysql.com/doc/refman/5.6/en/index-condition-pushdown-optimization.html)
- [mysql-explain-using-index-vs-using-index-condition](http://stackoverflow.com/questions/1687548/mysql-explain-using-index-vs-using-index-condition)
- [whats-the-difference-between-using-index-and-using-where-using-index-in-the](http://stackoverflow.com/questions/25672552/whats-the-difference-between-using-index-and-using-where-using-index-in-the)
- [【MySQL】 性能优化之 延迟关联 ](http://blog.itpub.net/22664653/viewspace-1176153/)
- [Mysql covering vs composite vs column index](http://stackoverflow.com/questions/8213235/mysql-covering-vs-composite-vs-column-index)

