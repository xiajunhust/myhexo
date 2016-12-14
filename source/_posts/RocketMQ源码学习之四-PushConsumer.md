---
title: RocketMQ源码学习之四-PushConsumer
date: 2016-11-17 23:06:03
tags: RocketMQ
---

本文主要讲解RocketMQ的push消息消费者的实现原理。

# 1 RocketMQ的两种消息消费者  
RocketMQ支持两种形式的消息消费者：  

* Push Consumer  
通常是应用向Consumer对象注册一个Listener，一旦收到消息，Consumer对象立即回调Listener接口方法。底层采用的是Pull长轮询拉取消息的方式。
* Pull Consumer  
应用通常主动调用Consumer的拉消息方法从Broker拉消息，主动权由应用控制。

# 2 Push Consumer的一个demo  

<!-- more -->

demo：

```
public class Consumer2 {
  public static void main(String[] args) throws InterruptedException, MQClientException {
    DefaultMQPushConsumer
        consumer = new DefaultMQPushConsumer("please_rename_unique_group_name_4");
    consumer.setVipChannelEnabled(false);
    consumer.setNamesrvAddr(Constants.hostIpStr + ":9876");

    /**
     * 设置Consumer第一次启动是从队列头部开始消费还是队列尾部开始消费<br>
     * 如果非第一次启动，那么按照上次消费的位置继续消费
     */
    consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);

    consumer.subscribe("TopicTest2", "*");

    consumer.registerMessageListener(new MessageListenerOrderly() {

      Random random = new Random();

      public ConsumeOrderlyStatus consumeMessage(List<MessageExt> msgs, ConsumeOrderlyContext context) {
        context.setAutoCommit(true);
        System.out.print(Thread.currentThread().getName() + " Receive New Messages: " );
        for (MessageExt msg: msgs) {
          System.out.println(msg + ", content:" + new String(msg.getBody()));
        }
        try {
          //模拟业务逻辑处理中...
          TimeUnit.SECONDS.sleep(random.nextInt(10));
        } catch (Exception e) {
          e.printStackTrace();
        }
        return ConsumeOrderlyStatus.SUCCESS;
      }
    });

    consumer.start();

    System.out.println("Consumer2 Started.");
  }
}
```

运行结果：

```
consumeMessageThread_2 Receive New Messages: MessageExt [queueId=0, storeSize=209, queueOffset=30, sysFlag=0, bornTimestamp=1479262699041, bornHost=/172.17.8.240:59605, storeTimestamp=1479262699053, storeHost=/172.17.8.240:10911, msgId=AC1108F000002A9F000000000000246D, commitLogOffset=9325, bodyCRC=2087030154, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message [topic=TopicTest2, flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=31, KEYS=KEY0, UNIQ_KEY=AC1108F00B4E1218025C4F757A200000, WAIT=true, TAGS=TagA}, body=36]], content:2016-11-16 10:18:18 Hello RocketMQ 0
ConsumeMessageThread_2 Receive New Messages: MessageExt [queueId=0, storeSize=209, queueOffset=31, sysFlag=0, bornTimestamp=1479262699062, bornHost=/172.17.8.240:59605, storeTimestamp=1479262699063, storeHost=/172.17.8.240:10911, msgId=AC1108F000002A9F000000000000253E, commitLogOffset=9534, bodyCRC=191020316, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message [topic=TopicTest2, flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=32, KEYS=KEY1, UNIQ_KEY=AC1108F00B4E1218025C4F757A360001, WAIT=true, TAGS=TagB}, body=36]], content:2016-11-16 10:18:18 Hello RocketMQ 1
```

# 3 Push消费方式原理分析  

## 3.1 消费者启动  
Push消费方式-消费者启动过程序列图如下：  
![Push消费者启动序列图](http://o8sltkx20.bkt.clouddn.com/rocketmq-sourcecode-4-2.jpg)

### 3.1.1 设置属性  
NamesrvAddr  
ConsumeFromWhere-第一次启动从队列头部还是尾部开始消费  
订阅的topic以及tag  
### 3.1.2 监听器注册  
这里监听器有2种类型：  
MessageListenerOrderly-顺序消费消息  
MessageListenerConcurrently-并发消费消息  
### 3.1.3 订阅配置拷贝  
将订阅关系设置到rebalanceImpl的subscriptionInner中。
### 3.1.4 创建并加载OffsetStore实例  
OffsetStore保存了消息消费的进度。会依据MessageModel来创建不同类型的实例：  
BROADCASTING-LocalFileOffsetStore  
CLUSTERING-RemoteBrokerOffsetStore
### 3.1.5 ConsumeMessageService实例创建  
会依据监听器的类型分别创建对应的ConsumeMessageService实例，ConsumeMessageService负责完成具体的消息消费：  
顺序消费-ConsumeMessageOrderlyService  
并发消费-ConsumeMessageConcurrentlyService  
后面会做详细介绍。
### 3.1.6 MQClientInstance启动  
这一步非常重要：启动了消息拉取服务PullMessageService，拉取消息供ConsumeMessageService消费。 

## 3.2 Pull长轮询拉取消息消费实现原理  
整个Pull长轮询拉取消息消费执行序列图如下：  
![Pull长轮询拉取消息消费执行序列图](http://o8sltkx20.bkt.clouddn.com/rocketmq-sourcecode-4-4.jpg)

源码分析：  

```
MQClientInstance.start

public void start() throws MQClientException {

    synchronized (this) {
        switch (this.serviceState) {
            case CREATE_JUST:
                this.serviceState = ServiceState.START_FAILED;
                // If not specified,looking address from name server
                if (null == this.clientConfig.getNamesrvAddr()) {
                    this.clientConfig.setNamesrvAddr(this.mQClientAPIImpl.fetchNameServerAddr());
                }
                // Start request-response channel
                this.mQClientAPIImpl.start();
                // Start various schedule tasks
                this.startScheduledTask();
                // Start pull service
                this.pullMessageService.start();
                // Start rebalance service
                this.rebalanceService.start();
                // Start push service
                this.defaultMQProducer.getDefaultMQProducerImpl().start(false);
                log.info("the client factory [{}] start OK", this.clientId);
                this.serviceState = ServiceState.RUNNING;
                break;
            case RUNNING:
                break;
            case SHUTDOWN_ALREADY:
                break;
            case START_FAILED:
                throw new MQClientException("The Factory object[" + this.getClientId() + "] has been created before, and failed.", null);
            default:
                break;
        }
    }
}
```
    
这里面非常重要的异步就是消息拉取服务PullMessageService的启动，PullMessageService通过长轮训拉取消息，然后交给ConsumeMessageService完成具体的消息消费。  
我们看下PullMessageService启动做了什么事情，启动了一个线程，run函数中核心代码：

```
public void run() {
    while (!this.isStoped()) {
    	 ...
    	 //从队列中获取PullRequest，调用拉取消息方法
        PullRequest pullRequest = this.pullRequestQueue.take();
        if (pullRequest != null) {
            this.pullMessage(pullRequest);
        }
        ...
    }
}

//拉取消息
private void pullMessage(final PullRequest pullRequest) {
	//获取消费者，调用消费者的消息拉取方法
    final MQConsumerInner consumer = this.mQClientFactory.selectConsumer(pullRequest.getConsumerGroup());
    if (consumer != null) {
        DefaultMQPushConsumerImpl impl = (DefaultMQPushConsumerImpl) consumer;
        impl.pullMessage(pullRequest);
    } else {
        log.warn("No matched consumer for the PullRequest {}, drop it", pullRequest);
    }
}
```

这里对应的消费者是DefaultMQPushConsumerImpl，我们接下来看DefaultMQPushConsumerImpl的pullMessage方法。  
（1）流控，如果超出流量限制则，或者broker锁定失败，将PullRequest提交到PullMessageService的单线程线程池。  

```
public void executePullRequestLater(final PullRequest pullRequest, final long timeDelay) {
    this.scheduledExecutorService.schedule(new Runnable() {

        @Override
        public void run() {
            PullMessageService.this.executePullRequestImmediately(pullRequest);
        }
    }, timeDelay, TimeUnit.MILLISECONDS);
}

public void executePullRequestImmediately(final PullRequest pullRequest) {
    try {
    	 //将拉取请求放入阻塞队列
        this.pullRequestQueue.put(pullRequest);
    } catch (InterruptedException e) {
        log.error("executePullRequestImmediately pullRequestQueue.put", e);
    }
}
```
    
（2）启动拉取PullAPIWrapper.pullKernelImpl

```
this.pullAPIWrapper.pullKernelImpl(//
        pullRequest.getMessageQueue(), // 1
        subExpression, // 2
        subscriptionData.getSubVersion(), // 3
        pullRequest.getNextOffset(), // 4
        this.defaultMQPushConsumer.getPullBatchSize(), // 5
        sysFlag, // 6
        commitOffsetValue, // 7
        BrokerSuspendMaxTimeMillis, // 8
        ConsumerTimeoutMillisWhenSuspend, // 9
        CommunicationMode.ASYNC, // 10
        pullCallback// 11
);
    

public PullResult pullKernelImpl(//
                                 final MessageQueue mq,// 1
                                 final String subExpression,// 2
                                 final long subVersion,// 3
                                 final long offset,// 4
                                 final int maxNums,// 5
                                 final int sysFlag,// 6
                                 final long commitOffset,// 7
                                 final long brokerSuspendMaxTimeMillis,// 8
                                 final long timeoutMillis,// 9
                                 final CommunicationMode communicationMode,// 10
                                 final PullCallback pullCallback// 11
) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    FindBrokerResult findBrokerResult =
            this.mQClientFactory.findBrokerAddressInSubscribe(mq.getBrokerName(),
                    this.recalculatePullFromWhichNode(mq), false);
    if (null == findBrokerResult) {
        this.mQClientFactory.updateTopicRouteInfoFromNameServer(mq.getTopic());
        findBrokerResult =
                this.mQClientFactory.findBrokerAddressInSubscribe(mq.getBrokerName(),
                        this.recalculatePullFromWhichNode(mq), false);
    }

    if (findBrokerResult != null) {
        int sysFlagInner = sysFlag;

        if (findBrokerResult.isSlave()) {
            sysFlagInner = PullSysFlag.clearCommitOffsetFlag(sysFlagInner);
        }

        PullMessageRequestHeader requestHeader = new PullMessageRequestHeader();
        requestHeader.setConsumerGroup(this.consumerGroup);
        requestHeader.setTopic(mq.getTopic());
        requestHeader.setQueueId(mq.getQueueId());
        requestHeader.setQueueOffset(offset);
        requestHeader.setMaxMsgNums(maxNums);
        requestHeader.setSysFlag(sysFlagInner);
        requestHeader.setCommitOffset(commitOffset);
        requestHeader.setSuspendTimeoutMillis(brokerSuspendMaxTimeMillis);
        requestHeader.setSubscription(subExpression);
        requestHeader.setSubVersion(subVersion);

        String brokerAddr = findBrokerResult.getBrokerAddr();
        if (PullSysFlag.hasClassFilterFlag(sysFlagInner)) {
            brokerAddr = computPullFromWhichFilterServer(mq.getTopic(), brokerAddr);
        }

        PullResult pullResult = this.mQClientFactory.getMQClientAPIImpl().pullMessage(//
                brokerAddr,//
                requestHeader,//
                timeoutMillis,//
                communicationMode,//这里传的是ASYNC
                pullCallback);

        return pullResult;
    }

    throw new MQClientException("The broker[" + mq.getBrokerName() + "] not exist", null);
}
```

接下来调到底层通信层NettyRemotingClient.invokeAsyncImpl:

```
private void pullMessageAsync(//
                              final String addr, // 1
                              final RemotingCommand request, //
                              final long timeoutMillis, //
                              final PullCallback pullCallback//
) throws RemotingException, InterruptedException {
    this.remotingClient.invokeAsync(addr, request, timeoutMillis, new InvokeCallback() {
        @Override
        public void operationComplete(ResponseFuture responseFuture) {
            RemotingCommand response = responseFuture.getResponseCommand();
            if (response != null) {
                try {
                    PullResult pullResult = MQClientAPIImpl.this.processPullResponse(response);
                    assert pullResult != null;
                    pullCallback.onSuccess(pullResult);
                } catch (Exception e) {
                    pullCallback.onException(e);
                }
            } else {
                if (!responseFuture.isSendRequestOK()) {
                    pullCallback.onException(new MQClientException("send request failed", responseFuture.getCause()));
                } else if (responseFuture.isTimeout()) {
                    pullCallback.onException(new MQClientException("wait response timeout " + responseFuture.getTimeoutMillis() + "ms",
                            responseFuture.getCause()));
                } else {
                    pullCallback.onException(new MQClientException("unknow reseaon", responseFuture.getCause()));
                }
            }
        }
    });
}
    
public void invokeAsyncImpl(final Channel channel, final RemotingCommand request, final long timeoutMillis,final InvokeCallback invokeCallback)
        throws InterruptedException, RemotingTooMuchRequestException, RemotingTimeoutException, RemotingSendRequestException {
    final int opaque = request.getOpaque();
    boolean acquired = this.semaphoreAsync.tryAcquire(timeoutMillis, TimeUnit.MILLISECONDS);
    if (acquired) {
        final SemaphoreReleaseOnlyOnce once = new SemaphoreReleaseOnlyOnce(this.semaphoreAsync);

        final ResponseFuture responseFuture = new ResponseFuture(opaque, timeoutMillis, invokeCallback, once);
        this.responseTable.put(opaque, responseFuture);
        try {
            channel.writeAndFlush(request).addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture f) throws Exception {
                    if (f.isSuccess()) {
                        responseFuture.setSendRequestOK(true);
                        return;
                    } else {
                        responseFuture.setSendRequestOK(false);
                    }

                    responseFuture.putResponse(null);
                    responseTable.remove(opaque);
                    try {
                    	 //执行回调函数InvokeCallback
                        responseFuture.executeInvokeCallback();
                    } catch (Throwable e) {

		....
}
```

可以看出是采用的异步模式，注册监听器，收到响应结果后调用回调函数InvokeCallback。而在InvokeCallback中调用了DefaultMQPushConsumerImpl中的PullCallback回调函数的onSuccess、onException，分别就返回成功结果和失败结果执行对应操作：

```
PullCallback核心代码：  
@Override
public void onSuccess(PullResult pullResult) {
	switch (pullResult.getPullStatus()) {
	case FOUND:
		...
		//将拉取到的消息放入processQueue
	    boolean dispathToConsume = processQueue.putMessage(pullResult.getMsgFoundList());
	    //向消息服务consumeMessageService提交消息消费请求
	    DefaultMQPushConsumerImpl.this.consumeMessageService.submitConsumeRequest(//
	            pullResult.getMsgFoundList(), //
	            processQueue, //
	            pullRequest.getMessageQueue(), //
	            dispathToConsume);
...
	
	
@Override
public void onException(Throwable e) {
    if (!pullRequest.getMessageQueue().getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
        log.warn("execute the pull request exception", e);
    }
	//提交到线程池，稍后执行
    DefaultMQPushConsumerImpl.this.executePullRequestLater(pullRequest, PullTimeDelayMillsWhenException);
}
```


## 3.2 消息消费服务-ConsumeMessageService  
Pull消息拉取服务拉取消息成功之后，向ConsumeMessageService提交消息消费请求，由后者完成具体的消费过程。具体逻辑在下一篇文章中讲解。 
