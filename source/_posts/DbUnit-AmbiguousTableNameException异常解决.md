---
title: DbUnit AmbiguousTableNameException异常解决
date: 2016-07-14 20:45:12
tags: 测试技术
---

# 问题描述  
今天在用DbUnit写DAO单元测试（使用的是mysql数据库）的时候，抛异常：

<!-- more -->

```
org.dbunit.database.AmbiguousTableNameException: ACTIVITY

	at org.dbunit.dataset.OrderedTableNameMap.add(OrderedTableNameMap.java:198)
	at org.dbunit.database.DatabaseDataSet.initialize(DatabaseDataSet.java:231)
	at org.dbunit.database.DatabaseDataSet.getTableMetaData(DatabaseDataSet.java:281)
	at org.dbunit.operation.DeleteAllOperation.execute(DeleteAllOperation.java:109)
	at org.dbunit.operation.CompositeOperation.execute(CompositeOperation.java:79)
	at org.dbunit.AbstractDatabaseTester.executeOperation(AbstractDatabaseTester.java:190)
	at org.dbunit.AbstractDatabaseTester.onSetup(AbstractDatabaseTester.java:103)
	at com.youzan.trade.demo.BaseTest.before(BaseTest.java:43)
	at com.youzan.trade.demo.BaseDAOTest.before(BaseDAOTest.java:28)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.junit.runners.model.FrameworkMethod$1.runReflectiveCall(FrameworkMethod.java:50)
	at org.junit.internal.runners.model.ReflectiveCallable.run(ReflectiveCallable.java:12)
	at org.junit.runners.model.FrameworkMethod.invokeExplosively(FrameworkMethod.java:47)
	at org.junit.internal.runners.statements.RunBefores.evaluate(RunBefores.java:24)
	at org.springframework.test.context.junit4.statements.RunBeforeTestMethodCallbacks.evaluate(RunBeforeTestMethodCallbacks.java:74)
	at org.springframework.test.context.junit4.statements.RunAfterTestMethodCallbacks.evaluate(RunAfterTestMethodCallbacks.java:83)
	at org.springframework.test.context.junit4.statements.SpringRepeat.evaluate(SpringRepeat.java:72)
	at org.springframework.test.context.junit4.SpringJUnit4ClassRunner.runChild(SpringJUnit4ClassRunner.java:231)
	at org.springframework.test.context.junit4.SpringJUnit4ClassRunner.runChild(SpringJUnit4ClassRunner.java:88)
	at org.junit.runners.ParentRunner$3.run(ParentRunner.java:290)
	at org.junit.runners.ParentRunner$1.schedule(ParentRunner.java:71)
	at org.junit.runners.ParentRunner.runChildren(ParentRunner.java:288)
	at org.junit.runners.ParentRunner.access$000(ParentRunner.java:58)
	at org.junit.runners.ParentRunner$2.evaluate(ParentRunner.java:268)
	at org.springframework.test.context.junit4.statements.RunBeforeTestClassCallbacks.evaluate(RunBeforeTestClassCallbacks.java:61)
	at org.springframework.test.context.junit4.statements.RunAfterTestClassCallbacks.evaluate(RunAfterTestClassCallbacks.java:71)
	at org.junit.runners.ParentRunner.run(ParentRunner.java:363)
	at org.springframework.test.context.junit4.SpringJUnit4ClassRunner.run(SpringJUnit4ClassRunner.java:174)
	at org.junit.runner.JUnitCore.run(JUnitCore.java:137)
	at com.intellij.junit4.JUnit4IdeaTestRunner.startRunnerWithArgs(JUnit4IdeaTestRunner.java:69)
	at com.intellij.rt.execution.junit.JUnitStarter.prepareStreamsAndStart(JUnitStarter.java:234)
	at com.intellij.rt.execution.junit.JUnitStarter.main(JUnitStarter.java:74)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at com.intellij.rt.execution.application.AppMain.main(AppMain.java:144)
```

# AmbiguousTableNameException的含义
Google搜了下<b>AmbiguousTableNameException</b>关键字，得到如下信息：  

```
This exception is thrown by IDataSet when multiple tables having the same name are accessible.This usually occurs when the database connection have access to multiple schemas containing identical table names.
```

大概意思是这个异常通常在构建IDataSet的时候会抛出，原因是由于数据库连接能访问到超过2个相同名字的数据表。  

一开始并未注意到异常后面的ACTIVITY，而恰巧这个时候我是在做另一个数据表buyer\_address的测试，因此排查花费了不少时间。使用mysql客户端查看数据库下面的所有数据表，发现具有名字buyer_address的数据表只有1个。无奈只能在DbUnit jar中添加断点:  
![debug point](http://o8sltkx20.bkt.clouddn.com/0008.png)  

debug发现，DbUnit会去扫数据库中的所有表，由于数据库中有2个表：activity、Activity，因此就抛出了异常。同时也可以看出，并不会区分大小写。下面我们分析下为什么没有区分大小写。  

看OrderedTableNameMap的getTableName函数：

```
    /**
     * Returns the table name in the correct case (for example as upper case string)
     * @param tableName The input table name to be resolved
     * @return The table name for the given string in the correct case.
     */
    public String getTableName(String tableName) 
    {
        if(LOGGER.isDebugEnabled())
            LOGGER.debug("getTableName(tableName={}) - start", tableName);
        
        String result = tableName;
        if(!_caseSensitiveTableNames)
        {
            // "Locale.ENGLISH" Fixes bug #1537894 when clients have a special
            // locale like turkish. (for release 2.4.3)
            result = tableName.toUpperCase(Locale.ENGLISH);
        }

        if(LOGGER.isDebugEnabled())
            LOGGER.debug("getTableName(tableName={}) - end - result={}", tableName, result);
        
        return result;
    }
```

而_caseSensitiveTableNames变量定义默认是false，因此不会区分大小写：  

```
	/**
	 * Whether or not case sensitive table names should be used. Defaults to false.
	 */
	private boolean _caseSensitiveTableNames = false;
```
	
# 解决方法  
解决方法有如下几种：  
（1）如果同一个数据库中有2个同名的数据表（不区分大小写），则删除数据库中同名的数据库。  
（2）如果数据库连接没有指定scheme，则指定数据库连接的schema。

# 参考资料
- [FAQ-AmbiguousTableNameException](http://dbunit.sourceforge.net/faq.html#AmbiguousTableNameException)
- [AmbiguousTableNameException](http://dbunit.sourceforge.net/apidocs/org/dbunit/database/AmbiguousTableNameException.html)