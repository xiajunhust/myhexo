---
title: MySQL的日期时间类型关于秒以下精度的处理
date: 2016-08-28 20:58:10
tags: MySQL
---

# 1 MySQL中日期和时间的表示

MySQL中有多种数据类型可以用于日期和时间的表示：

![](http://o8sltkx20.bkt.clouddn.com/MySQL-base-001.png)

<!-- more -->

常用的是如下三种：  
（1）DATE：表示年月日  
（2）DATETIME：表示年月日时分秒  
（3）TIME：只需要表示时分秒  

DATETIME和TIMESTAMP很类似，但是有如下区别：  
（1）TIMESTAMP支持的时间范围较小  
（2）在插入或者更新时不明确指定TIMESTAMP列的值时，系统将会设置为当前日期和时间。当值超出最大值值，将会使用默认值“0000-00-00 00:00”  
（3）TIMESTAMP的插入和更新都受到当地时区的影响  
（4）TIMESTAMP受MySQL版本和服务器SQLMode的影响很大  

# 2 MySQL日期时间对秒以下精度的支持

<b>MySQL在5.6.4以及更高版本提供了对秒以下精度时间的存储支持，在以前的版本是会将秒以下的精度忽略掉的。</b>TIME、TIMESTAMP、DATETIME均提供了小数点后6位的支持（微秒）。如果我们想定义一个能存储秒以下精度的日期时间字段，可以如下做，在类型字段后面指定小数点后的精度位数：

```
CREATE TABLE t1 (t TIME(3), dt DATETIME(6));
```

如果我们朝数据库中插入一条记录，时间的精度位数大于表字段能表示的位数，则会造成四舍五入：

```
mysql> CREATE TABLE fractest( c1 TIME(2), c2 DATETIME(2), c3 TIMESTAMP(2) );
Query OK, 0 rows affected (0.33 sec)

mysql> INSERT INTO fractest VALUES
     > ('17:51:04.777', '2014-09-08 17:51:04.777', '2014-09-08 17:51:04.777');
Query OK, 1 row affected (0.03 sec)

mysql> SELECT * FROM fractest;
+-------------+------------------------+------------------------+
| c1          | c2                     | c3                     |
+-------------+------------------------+------------------------+
| 17:51:04.78 | 2014-09-08 17:51:04.78 | 2014-09-08 17:51:04.78 |
+-------------+------------------------+------------------------+
1 row in set (0.00 sec)
```

<b>并且不会有任何警告，这点尤其要注意。</b>

截断在数据库server端，connector是会将毫秒部分一并提交给server的。我们可以看下mysql-connector-java jar中的源码：

```

/**
 * Set a parameter to a java.sql.Timestamp value. The driver converts this
 * to a SQL TIMESTAMP value when it sends it to the database.
 * 
 * @param parameterIndex
 *            the first parameter is 1, the second is 2, ...
 * @param x
 *            the parameter value
 * @param tz
 *            the timezone to use
 * 
 * @throws SQLException
 *             if a database-access error occurs.
 */
private void setTimestampInternal(int parameterIndex,
		Timestamp x, Calendar targetCalendar,
		TimeZone tz, boolean rollForward) throws SQLException {
	synchronized (checkClosed().getConnectionMutex()) {
		if (x == null) {
			setNull(parameterIndex, java.sql.Types.TIMESTAMP);
		} else {
			checkClosed();
			
			if (!this.useLegacyDatetimeCode) {
				newSetTimestampInternal(parameterIndex, x, targetCalendar);
			} else {
				Calendar sessionCalendar = this.connection.getUseJDBCCompliantTimezoneShift() ?
						this.connection.getUtcCalendar() : 
							getCalendarInstanceForSessionOrNew();
					
				synchronized (sessionCalendar) {
					x = TimeUtil.changeTimezone(this.connection, 
							sessionCalendar,
							targetCalendar,
							x, tz, this.connection
						.getServerTimezoneTZ(), rollForward);
				}
	
				if (this.connection.getUseSSPSCompatibleTimezoneShift()) {
					doSSPSCompatibleTimezoneShift(parameterIndex, x, sessionCalendar);
				} else {
					synchronized (this) {
						if (this.tsdf == null) {
							this.tsdf = new SimpleDateFormat("''yyyy-MM-dd HH:mm:ss", Locale.US); //$NON-NLS-1$
						}
						
						StringBuffer buf = new StringBuffer();
						buf.append(this.tsdf.format(x));

						if (this.serverSupportsFracSecs) {
							int nanos = x.getNanos();
							//精确到了纳秒
							if (nanos != 0) {
								buf.append('.');
								buf.append(TimeUtil.formatNanos(nanos, this.serverSupportsFracSecs, true));
							}
						}

						buf.append('\'');

						setInternal(parameterIndex, buf.toString()); // SimpleDateFormat is not
																	  // thread-safe
					}
				}
			}
			
			this.parameterTypes[parameterIndex - 1 + getParameterIndexOffset()] = Types.TIMESTAMP;
		}
	}
}
```

如果使用的是mysql 5.6以下的版本，可以增加一个字段专门用来存储秒以下的精度：

```
CREATE TABLE your_table (
  dt datetime,
  us int
);

INSERT INTO your_table VALUES
('2011-11-11 11:11:11.111111', MICROSECOND('2011-11-11 11:11:11.111111'));
```

# 3 参考资料

(1)[MySQL Reference Manual](https://dev.mysql.com/doc/refman/5.6/en/fractional-seconds.html)  
(2)[http://www.yufengof.com/2015/08/17/mysql-datetime-type-millisecond-rounding/](http://www.yufengof.com/2015/08/17/mysql-datetime-type-millisecond-rounding/)  
(3)[how-to-insert-a-microsecond-precision-datetime-into-mysql](http://stackoverflow.com/questions/14038746/how-to-insert-a-microsecond-precision-datetime-into-mysql)