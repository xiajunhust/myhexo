---
title: 'InfluxDB写入失败-partial write: max-values-per-tag limit exceeded'
date: 2016-12-13 11:05:46
tags: InfluxDB
---

最近线上向InfluxDB写入数据突然报错，大概意思是tag的值超限，详细错误信息：

```
2016-12-02 20:47:11.485 [] [] [] ERROR c.y.t.p.m.m.i.OrderCreateMonitorManageImpl - InfluxDB写入异常!influxDO=InfluxDO(measurement=com.XXX.trade.create.request.result, timeStamp=1480682831481, tagKeyList=[cluster, host, count_time, count, count_type], tagValList=[DEFAULT_CLUSTER, xxx-inc.com, 1480682831481, 3, bill_new], fieldKeyList=[result, cost], fieldValList=[true, 112]), exception={}
java.lang.RuntimeException: {"error":"partial write: max-values-per-tag limit exceeded (100000/100000): measurement=\"com.XXX.trade.create.request.result\" tag=\"count_time\" value=\"count_time\" dropped=1"}

    at org.influxdb.impl.InfluxDBImpl.execute(InfluxDBImpl.java:266) ~[influxdb-java-2.4.jar:na]
    at org.influxdb.impl.InfluxDBImpl.write(InfluxDBImpl.java:167) ~[influxdb-java-2.4.jar:na]
    at org.influxdb.impl.InfluxDBImpl.write(InfluxDBImpl.java:157) ~[influxdb-java-2.4.jar:na]
    at com.XXX.trade.process.monitor.dal.impl.InfluxDBDAOImpl.insert(InfluxDBDAOImpl.java:104) ~[trade-process-service-1.0.0-SNAPSHOT.jar:na]
    at com.XXX.trade.process.monitor.manage.impl.OrderCreateMonitorManageImpl.save(OrderCreateMonitorManageImpl.java:40) ~[trade-process-service-1.0.0-SNAPSHOT.jar:na]
    at com.XXX.trade.process.nsq.processor.impl.MonitorProcessor.process(MonitorProcessor.java:69) [trade-process-service-1.0.0-SNAPSHOT.jar:na]
    at com.XXX.trade.process.nsq.NSQSubscriber.lambda$init$0(NSQSubscriber.java:71) [trade-process-service-1.0.0-SNAPSHOT.jar:na]
    at com.XXX.trade.process.nsq.NSQSubscriber$$Lambda$8/1829287142.process(Unknown Source) [trade-process-service-1.0.0-SNAPSHOT.jar:na]
```

Google搜索了下，这是InfluxDB v1.1.0 [2016-11-14]版本新加的一个特性([CHANGELOG.md](https://github.com/influxdata/influxdb/blob/master/CHANGELOG.md))：

<I>
max-values-per-tag was added with a default of 100,000, but can be disabled by setting it to 0. Existing measurements with tags that exceed this limit will continue to load, but writes that would cause the tags cardinality to increase will be dropped and a partial write error will be returned to the caller. This limit can be used to prevent high cardinality tag values from being written to a measurement.
</I>

增加了对tag的值的大小的校验，最大值是10000.若写入的数据的tag的值超限，则调用方会收到报错。用来防止写入到measurement的数据的series-cardinality([series-cardinality](https://docs.influxdata.com/influxdb/v1.1/concepts/glossary/#series-cardinality))过大。

<I>
关于series cardinality：  
The number of unique database, measurement, and tag set combinations in an InfluxDB instance.
</I>

<B>这样做的用意是什么呢？</B>  
[issue-7146：Add option max-tags-per-database to limit high cardinality data](https://github.com/influxdata/influxdb/issues/7146)  

<I>
Much as max-series-per-database in [#7095](https://github.com/influxdata/influxdb/pull/7095), add a max-tags-per-database to limit high cardinality data.

Writes beyond that should be dropped and logged with a rate limit on the log (one summary log per minute or something saying x writes dropped).

This is important because if you accidentally load in a huge amount of high cardinality data, you can easily get into a place that InfluxDB will OOM if you attempt to delete the data, so your only choice is to try to move the files on disk out the way which deletes other data.

---

Configuration setting for managing series cardinality

Limiting your series cardinality is an essential part of working with InfluxDB. The new max-values-per-tag configuration setting can help you do just that.

The setting limits the number of tag values allowed per tag key. The default setting is 10,000 (so 10,000 tag values allowed per tag key). If a write causes the number of tag values for a tag key to exceed 10,000, InfluxDB will not write the point and it returns a partial write error.
</I>

如果我们写入了大量的series-cardinality很高的数据，那么当我们删除数据的时候，InfluxDB会OOM。

<b>为什么我们需要关注series cardility？</b>  
原因是InfluxDB在内存中维护了系统中每个series数据的索引。随着具有唯一性的series数据数量的增长，RAM的使用也会增长。过高的series cardinality会导致操作系统kill掉InfluxDB进程，抛出OOM异常。  

<I>
InfluxDB maintains an in-memory index of every series in the system. As the number of unique series grows, so does the RAM usage. High series cardinality can lead to the operating system killing the InfluxDB process with an out of memory (OOM) exception. See Querying for series cardinality to learn how to query for series cardinality.
</I>



## 参考资料

- [https://github.com/influxdata/influxdb/issues/7146](https://github.com/influxdata/influxdb/issues/7146)
- [https://github.com/influxdata/influxdb/blob/master/CHANGELOG.md](https://github.com/influxdata/influxdb/blob/master/CHANGELOG.md)
- [https://www.influxdata.com/tldr-influxdb-tech-tips-november-15-2016/](https://www.influxdata.com/tldr-influxdb-tech-tips-november-15-2016/)
- [FAQ-how-can-i-query-for-series-cardinality](https://docs.influxdata.com/influxdb/v1.1/troubleshooting/frequently-asked-questions/#how-can-i-query-for-series-cardinality)