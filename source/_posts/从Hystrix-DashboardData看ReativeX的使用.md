---
title: 从Hystrix DashboardData看ReativeX的使用
date: 2016-12-18 14:23:43
tags: [ReactiveX,Hystrix]
---

最近在将系统中服务降级框架[Hystrix](https://github.com/Netflix/Hystrix)的运行时的一些指标数据接入到监控平台，需要获取Hystrix的Dashboard数据。发现用到了[ReactiveX](http://reactivex.io/)的知识，因此总结下。

注：本文基于的Hystrix的版本是：

```
<hystrix.version>1.5.7</hystrix.version>
<dependency>
    <groupId>com.netflix.hystrix</groupId>
    <artifactId>hystrix-core</artifactId>
    <version>${hystrix.version}</version>
</dependency>
```

## 1 如何获取Hystrix的Dashboard数据  
获取Hystrix的Dashboard数据比较简单，实现一个观察者即可接收Dashboard数据，如下代码：

```
Observable<HystrixDashboardStream.DashboardData> 
       dashboardDataObservable = HystrixDashboardStream.getInstance().observe();
   
dashboardDataObservable.subscribe(new Action1<HystrixDashboardStream.DashboardData>() {
	@Override
	public void call(HystrixDashboardStream.DashboardData dashboardData) {
	
	//do something
	}
});
```

## 2 HystrixDashboard设计 

<!-- more -->

Hystrix Dashboard数据获取的核心类：HystrixDashboardStream。

## 2.1 Hystrix Dashboard数据结构  
Hystrix Dashboard数据结构封装类是HystrixDashboardStream.DashboardData:  

```
public static class DashboardData {
    final Collection<HystrixCommandMetrics> commandMetrics;
    final Collection<HystrixThreadPoolMetrics> threadPoolMetrics;
    final Collection<HystrixCollapserMetrics> collapserMetrics;

    public DashboardData(Collection<HystrixCommandMetrics> commandMetrics, Collection<HystrixThreadPoolMetrics> threadPoolMetrics, Collection<HystrixCollapserMetrics> collapserMetrics) {
        this.commandMetrics = commandMetrics;
        this.threadPoolMetrics = threadPoolMetrics;
        this.collapserMetrics = collapserMetrics;
    }

    public Collection<HystrixCommandMetrics> getCommandMetrics() {
        return commandMetrics;
    }

    public Collection<HystrixThreadPoolMetrics> getThreadPoolMetrics() {
        return threadPoolMetrics;
    }

    public Collection<HystrixCollapserMetrics> getCollapserMetrics() {
        return collapserMetrics;
    }
}
```

包括了三类数据： 

- 每个Command的metrics  
比如当前并发执行的数量、执行时间、执行的成功数、失败数以及失败率。
- 每个线程池的metrics  
比如总共执行的、总共被拒绝的、当前活跃的线程数目等等。
- 请求合并调用

如下是一个数据输出示例：

```
2016-12-16 10:28:08.809 [] [] [] INFO  c.y.t.t.MonitorHystrixJob - HystrixDashboardStream.DashboardData:
2016-12-16 10:28:08.809 [] [] [] INFO  c.y.t.t.MonitorHystrixJob - commandMetrics:
2016-12-16 10:28:08.809 [] [] [] INFO  c.y.t.t.MonitorHystrixJob - Properties.circuitBreakerEnabled=true,CommandKey=GoodsPostageCommand,CurrentConcurrentExecutionCount=0,ExecutionTimeMean=0,HealthCounts=HealthCounts[0 / 2 : 0%] ,TotalTimeMean=0
2016-12-16 10:28:08.809 [] [] [] INFO  c.y.t.t.MonitorHystrixJob - Properties.circuitBreakerEnabled=true,CommandKey=DeliverySettingCommand,CurrentConcurrentExecutionCount=0,ExecutionTimeMean=0,HealthCounts=HealthCounts[0 / 3 : 0%] ,TotalTimeMean=0
2016-12-16 10:28:08.810 [] [] [] INFO  c.y.t.t.MonitorHystrixJob - Properties.circuitBreakerEnabled=true,CommandKey=GoodsBriefInfoCommand,CurrentConcurrentExecutionCount=0,ExecutionTimeMean=0,HealthCounts=HealthCounts[0 / 5 : 0%] ,TotalTimeMean=0
2016-12-16 10:28:08.810 [] [] [] INFO  c.y.t.t.MonitorHystrixJob - Properties.circuitBreakerEnabled=true,CommandKey=UmpBillPreferenceCommand,CurrentConcurrentExecutionCount=0,ExecutionTimeMean=0,HealthCounts=HealthCounts[0 / 3 : 0%] ,TotalTimeMean=0
2016-12-16 10:28:08.810 [] [] [] INFO  c.y.t.t.MonitorHystrixJob - Properties.circuitBreakerEnabled=true,CommandKey=LocalDeliveryCommand,CurrentConcurrentExecutionCount=0,ExecutionTimeMean=0,HealthCounts=HealthCounts[0 / 1 : 0%] ,TotalTimeMean=0
2016-12-16 10:28:08.810 [] [] [] INFO  c.y.t.t.MonitorHystrixJob - threadPoolMetrics:
2016-12-16 10:28:08.810 [] [] [] INFO  c.y.t.t.MonitorHystrixJob - ThreadPoolKey=DeliveryGroup,CumulativeCountThreadsExecuted=1,CumulativeCountThreadsRejected=0,CurrentActiveCount=0,CurrentCompletedTaskCount=4,CurrentCorePoolSize=15,CurrentPoolSize=4,CurrentQueueSize=0,CurrentTaskCount=4
2016-12-16 10:28:08.810 [] [] [] INFO  c.y.t.t.MonitorHystrixJob - ThreadPoolKey=UmpServiceGroup,CumulativeCountThreadsExecuted=1,CumulativeCountThreadsRejected=0,CurrentActiveCount=0,CurrentCompletedTaskCount=3,CurrentCorePoolSize=15,CurrentPoolSize=3,CurrentQueueSize=0,CurrentTaskCount=3
2016-12-16 10:28:08.810 [] [] [] INFO  c.y.t.t.MonitorHystrixJob - ThreadPoolKey=GoodsServiceGroup,CumulativeCountThreadsExecuted=4,CumulativeCountThreadsRejected=0,CurrentActiveCount=0,CurrentCompletedTaskCount=7,CurrentCorePoolSize=15,CurrentPoolSize=7,CurrentQueueSize=0,CurrentTaskCount=7
```

## 2.2 设计思路  
这里采用了ReactiveX，创建了一个被观察者Observable，欲监控Dashboard数据，只需创建观察者Observer，接收Observable发射出来的数据即可。  

### 2.2.1 什么是ReactiveX  
微软给的定义是，Rx是一个函数库，让开发者可以利用可观察序列和LINQ风格查询操作符来编写异步和基于事件的程序，使用Rx，开发者可以用Observables表示异步数据流，用LINQ操作符查询异步数据流， 用Schedulers参数化异步数据流的并发处理，Rx可以这样定义：Rx = Observables + LINQ + Schedulers。

ReactiveX.io给的定义是，Rx是一个使用可观察数据流进行异步编程的编程接口，ReactiveX结合了观察者模式、迭代器模式和函数式编程的精华。

### 2.2.2 使用ReactiveX的好处  
使用观察者模式可以  

- 创建：Rx可以方便的创建事件流和数据流  
- 组合：Rx使用查询式的操作符组合和变换数据流  
- 监听：Rx可以订阅任何可观察的数据流并执行操作  

带来如下好处：  

- 函数式风格：对可观察数据流使用无副作用的输入输出函数，避免了程序里错综复杂的状态
- 简化代码：Rx的操作符通通常可以将复杂的难题简化为很少的几行代码
- 异步错误处理：传统的try/catch没办法处理异步计算，Rx提供了合适的错误处理机制
- 轻松使用并发：Rx的Observables和Schedulers让开发者可以摆脱底层的线程同步和各种并发问题

Java中对ReactiveX的实现是RxJava。

### 2.2.2 Hystrix Dashboard对ReactiveX的使用
在HystrixDashboardStream的构造函数中创建了Observable：

```
private static final DynamicIntProperty dataEmissionIntervalInMs =
            DynamicPropertyFactory.getInstance().getIntProperty("hystrix.stream.dashboard.intervalInMilliseconds", 500);

private HystrixDashboardStream(int delayInMs) {
    this.delayInMs = delayInMs;
    this.singleSource = Observable.interval(delayInMs, TimeUnit.MILLISECONDS)
            .map(new Func1<Long, DashboardData>() {
                @Override
                public DashboardData call(Long timestamp) {
                    return new DashboardData(
                            HystrixCommandMetrics.getInstances(),
                            HystrixThreadPoolMetrics.getInstances(),
                            HystrixCollapserMetrics.getInstances()
                    );
                }
            })
            .doOnSubscribe(new Action0() {
                @Override
                public void call() {
                    isSourceCurrentlySubscribed.set(true);
                }
            })
            .doOnUnsubscribe(new Action0() {
                @Override
                public void call() {
                    isSourceCurrentlySubscribed.set(false);
                }
            })
            .share()
            .onBackpressureDrop();
}
```

这里创建一个按固定时间间隔发射整数序列的Observable，默认间隔时间是500ms。

![operators.interval](http://o8sltkx20.bkt.clouddn.com/rx-001-001.png)

RxJava中[Observable.interval](http://o8sltkx20.bkt.clouddn.com/rx-001-002.png).

创建了Observable之后，紧接着调用了变换函数Map。  

![operators.map](http://o8sltkx20.bkt.clouddn.com/rx-001-002.png)

它对Observable发射的每一项数据应用一个函数，执行变换操作。这里的函数是调用HystrixCommandMetrics、HystrixThreadPoolMetrics、HystrixCollapserMetrics的实例获取方法，返回对应的Dashboard数据。

doOnSubscribe-当被订阅的时候，执行对应的操作。  
doOnUnsubscribe-当没有订阅者的时候，执行对应的操作。

# 3 参考资料  
- [ReactiveX官方网站](http://reactivex.io/)
- [RxJava](https://github.com/ReactiveX/RxJava)
