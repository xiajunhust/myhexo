---
title: RocketMQ源码学习之二-同步消息发送
date: 2016-11-12 20:52:32
tags: RocketMQ
---

本文讲述RocketMQ同步发送消息的原理。

# 1 消息生产者demo  
先看一段同步发送消息的demo代码：  

```
import com.alibaba.rocketmq.client.exception.MQClientException;
import com.alibaba.rocketmq.client.producer.DefaultMQProducer;
import com.alibaba.rocketmq.client.producer.SendResult;
import com.alibaba.rocketmq.common.message.Message;

public class Producer1 {
  public static void main(String[] args) throws MQClientException, InterruptedException {
    DefaultMQProducer producer = new DefaultMQProducer("ProducerGroupName");
    producer.setNamesrvAddr("192.168.2.101:9876");
    producer.setInstanceName("Producer1");
    producer.start();
    for (int i = 0; i < 1; i++)
      try {
        {
          Message msg = new Message("TopicTest",// topic
                                    "TagA",// tag
                                    "OrderID188",// key
                                    ("Hello MetaQ").getBytes());// body
          SendResult sendResult = producer.send(msg);
          System.out.println(sendResult);
        }

      }
      catch (Exception e) {
        e.printStackTrace();
      }

    producer.shutdown();
  }
}

```

<!-- more -->

运行结果：

```
log4j:WARN No appenders could be found for logger (com.alibaba.rocketmq.shade.io.netty.util.internal.logging.InternalLoggerFactory).
log4j:WARN Please initialize the log4j system properly.
log4j:WARN See http://logging.apache.org/log4j/1.2/faq.html#noconfig for more info.
SendResult [sendStatus=SEND_OK, msgId=C0A8026507741218025C3A983E8F0000,offsetMsgId=C0A8026500002A9F0000000000001A0C, messageQueue=MessageQueue [topic=TopicTest, brokerName=xiajuns-MacBook-Pro.local, queueId=1], queueOffset=0]

Process finished with exit code 0
```

# 2 同步消息发送原理分析  
RocketMQ网络部署图：  
![RocketMQ网络部署图](http://o8sltkx20.bkt.clouddn.com/rocketmq-sourcecode-1-1.jpeg)

原理：同步发送是指消息发送方发出数据后，会在收到接收方发回响应之后才发下一个数据包的通讯方式。

![同步消息发送原理示意图](http://o8sltkx20.bkt.clouddn.com/rocketmq-sourcecode-2-1.png)

应用场景：此种方式应用场景非常广泛，例如重要通知邮件、报名短信通知、营销短信系统等。

RocketMQ发送同步消息的整体序列图如下：  
![RocketMQ同步消息发送整体序列图](http://o8sltkx20.bkt.clouddn.com/rocketmq-sourcecode-1-2.jpg)

下面进行具体分析。  

## 2.1 发送消息前  
同步消息发送的入口函数是DefaultMQProducer.send(Message msg).  
发送消息之前需要完成如下事情：   

- 创建DefaultMQProducer实例  
其实就是指定group名字，以及创建了DefaultMQProducerImpl实例。发送晓得具体操作都是在DefaultMQProducerImpl中完成的，DefaultMQProducer只是做了一层封装。  
- 给DefaultMQProducer实例设置nameserver地址  
- 启动生产者  
启动过程较复杂，下面详细分析。  
首先判断服务状态，如果正在运行或者启动失败、已关闭，则抛出异常。否则执行下面的启动流程。  

```
//初始化服务状态为启动失败
this.serviceState = ServiceState.START_FAILED;

//检查配置，主要是ProducerGroup
this.checkConfig();

if (!this.defaultMQProducer.getProducerGroup().equals(MixAll.CLIENT_INNER_PRODUCER_GROUP)) {
    this.defaultMQProducer.changeInstanceNameToPID();
}

//创建MQClientInstance实例（MQClientInstance是MQ客户端实例，负责执行一些定时任务比如定时更新nameserver地址、从nameserver更新topic路由信息、向broker发送心跳等等。
//MQClientManager负责创建MQClientInstance，每个clientId只创建一个，第一次创建完成之后存放在一个ConcurrentHashMap中）
this.mQClientFactory = MQClientManager.getInstance().getAndCreateMQClientInstance(this.defaultMQProducer, rpcHook);

//注册Producer。实际就是将group名、Producer键值对存到producerTable ConcurrentHashMap里
boolean registerOK = mQClientFactory.registerProducer(this.defaultMQProducer.getProducerGroup(), this);
if (!registerOK) {
    this.serviceState = ServiceState.CREATE_JUST;
    throw new MQClientException("The producer group[" + this.defaultMQProducer.getProducerGroup()
            + "] has been created before, specify another name please." + FAQUrl.suggestTodo(FAQUrl.GROUP_NAME_DUPLICATE_URL),
            null);
}

//存储TopicPublishInfo
this.topicPublishInfoTable.put(this.defaultMQProducer.getCreateTopicKey(), new TopicPublishInfo());

//启动MQClientInstance（startFactory默认是true）
if (startFactory) {
    mQClientFactory.start();
}

log.info("the producer [{}] start OK. sendMessageWithVIPChannel={}", this.defaultMQProducer.getProducerGroup(),
        this.defaultMQProducer.isSendMessageWithVIPChannel());
        
//更改服务状态
this.serviceState = ServiceState.RUNNING;
```

## 2.2 发送消息  
序列图：  
![同步消息发送具体步骤](http://o8sltkx20.bkt.clouddn.com/rocketmq-sourcecode-1-3.jpg)

单步跟进，发现DefaultMQProducer.send(Message msg)会最终调到DefaultMQProducerImpl的这个方法：

```
public SendResult send(Message msg, long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    return this.sendDefaultImpl(msg, CommunicationMode.SYNC, null, timeout);
}
```

可以看到调sendDefaultImpl传递的CommunicationMode是SYNC，是同步模式：  

```
//三种消息模式枚举类型
public enum CommunicationMode {
//同步
SYNC,
//异步
ASYNC,
//单向
ONEWAY,
}
```

接下来调到sendKernelImpl方法，下面摘取重要的部分：  

```
private SendResult sendKernelImpl(final Message msg, //
                                  final MessageQueue mq, //
                                  final CommunicationMode communicationMode, //
                                  final SendCallback sendCallback, //
                                  final TopicPublishInfo topicPublishInfo, //
                                  final long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    //获取broker地址
    String brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
    if (null == brokerAddr) {
        tryToFindTopicPublishInfo(mq.getTopic());
        brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
    }

    SendMessageContext context = null;
    if (brokerAddr != null) {
        brokerAddr = MixAll.brokerVIPChannel(this.defaultMQProducer.isSendMessageWithVIPChannel(), brokerAddr);

        byte[] prevBody = msg.getBody();
        try {
			  //消息ID生成
            MessageClientIDSetter.setUniqID(msg);

            int sysFlag = 0;
            if (this.tryToCompressMessage(msg)) {
                sysFlag |= MessageSysFlag.CompressedFlag;
            }

            final String tranMsg = msg.getProperty(MessageConst.PROPERTY_TRANSACTION_PREPARED);
            if (tranMsg != null && Boolean.parseBoolean(tranMsg)) {
                sysFlag |= MessageSysFlag.TransactionPreparedType;
            }

			  //一些钩子函数执行
            if (hasCheckForbiddenHook()) {
                ...
            }

            if (this.hasSendMessageHook()) {
                ...
            }

			  //消息发送请求体构造
            SendMessageRequestHeader requestHeader = new SendMessageRequestHeader();
            requestHeader.setProducerGroup(this.defaultMQProducer.getProducerGroup());
            requestHeader.setTopic(msg.getTopic());
            requestHeader.setDefaultTopic(this.defaultMQProducer.getCreateTopicKey());
            requestHeader.setDefaultTopicQueueNums(this.defaultMQProducer.getDefaultTopicQueueNums());
            requestHeader.setQueueId(mq.getQueueId());
            requestHeader.setSysFlag(sysFlag);
            requestHeader.setBornTimestamp(System.currentTimeMillis());
            requestHeader.setFlag(msg.getFlag());
            requestHeader.setProperties(MessageDecoder.messageProperties2String(msg.getProperties()));
            requestHeader.setReconsumeTimes(0);
            requestHeader.setUnitMode(this.isUnitMode());
            if (requestHeader.getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
                String reconsumeTimes = MessageAccessor.getReconsumeTime(msg);
                if (reconsumeTimes != null) {
                    requestHeader.setReconsumeTimes(Integer.valueOf(reconsumeTimes));
                    MessageAccessor.clearProperty(msg, MessageConst.PROPERTY_RECONSUME_TIME);
                }

                String maxReconsumeTimes = MessageAccessor.getMaxReconsumeTimes(msg);
                if (maxReconsumeTimes != null) {
                    requestHeader.setMaxReconsumeTimes(Integer.valueOf(maxReconsumeTimes));
                    MessageAccessor.clearProperty(msg, MessageConst.PROPERTY_MAX_RECONSUME_TIMES);
                }
            }

            SendResult sendResult = null;
            switch (communicationMode) {
                case ASYNC:
                    sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(//
                            brokerAddr, // 1
                            mq.getBrokerName(), // 2
                            msg, // 3
                            requestHeader, // 4
                            timeout, // 5
                            communicationMode, // 6
                            sendCallback, // 7
                            topicPublishInfo, // 8
                            this.mQClientFactory, // 9
                            this.defaultMQProducer.getRetryTimesWhenSendAsyncFailed(), // 10
                            context, //
                            this);
                    break;
                case ONEWAY:
                case SYNC://同步发送模式
                    sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(//
                            brokerAddr, // 1
                            mq.getBrokerName(), // 2
                            msg, // 3
                            requestHeader, // 4
                            timeout, // 5
                            communicationMode, // 6
                            context,//
                            this);
                    break;
                default:
                    assert false;
                    break;
            }

            if (this.hasSendMessageHook()) {
                context.setSendResult(sendResult);
                this.executeSendMessageHookAfter(context);
            }

            return sendResult;
        
        ...
}
```

同步发送模式，最终会调到MQClientAPIImpl的如下方法：  

```
private SendResult sendMessageSync(//
                                   final String addr, //
                                   final String brokerName, //
                                   final Message msg, //
                                   final long timeoutMillis, //
                                   final RemotingCommand request//
) throws RemotingException, MQBrokerException, InterruptedException {
    RemotingCommand response = this.remotingClient.invokeSync(addr, request, timeoutMillis);
    assert response != null;
    return this.processSendResponse(brokerName, msg, response);
}
```

会调用RemotingClient.invokeSync发送同步消息。此处的RemotingClient是NettyRemotingClient，封装了Netty通信逻辑。

```
public RemotingCommand invokeSync(String addr, final RemotingCommand request, long timeoutMillis)
            throws InterruptedException, RemotingConnectException, RemotingSendRequestException, RemotingTimeoutException {
    final Channel channel = this.getAndCreateChannel(addr);
    if (channel != null && channel.isActive()) {
        try {
            if (this.rpcHook != null) {
                this.rpcHook.doBeforeRequest(addr, request);
            }
            RemotingCommand response = this.invokeSyncImpl(channel, request, timeoutMillis);
            if (this.rpcHook != null) {
                this.rpcHook.doAfterResponse(RemotingHelper.parseChannelRemoteAddr(channel), request, response);
            }
            return response;
            
            ...
}

public RemotingCommand invokeSyncImpl(final Channel channel, final RemotingCommand request, final long timeoutMillis)
        throws InterruptedException, RemotingSendRequestException, RemotingTimeoutException {
    final int opaque = request.getOpaque();

    try {
        final ResponseFuture responseFuture = new ResponseFuture(opaque, timeoutMillis, null, null);
        this.responseTable.put(opaque, responseFuture);
        final SocketAddress addr = channel.remoteAddress();
        
        //添加监听器
        channel.writeAndFlush(request).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture f) throws Exception {
                if (f.isSuccess()) {
                    responseFuture.setSendRequestOK(true);
                    return;
                } else {
                    responseFuture.setSendRequestOK(false);
                }

                responseTable.remove(opaque);
                responseFuture.setCause(f.cause());
                responseFuture.putResponse(null);
                plog.warn("send a request command to channel <" + addr + "> failed.");
            }
        });

		 //线程阻塞等待，获取发送消息的结果
        RemotingCommand responseCommand = responseFuture.waitResponse(timeoutMillis);
        if (null == responseCommand) {
            if (responseFuture.isSendRequestOK()) {
                throw new RemotingTimeoutException(RemotingHelper.parseSocketAddressAddr(addr), timeoutMillis,
                        responseFuture.getCause());
            } else {
                throw new RemotingSendRequestException(RemotingHelper.parseSocketAddressAddr(addr), responseFuture.getCause());
            }
        }

        return responseCommand;
    } finally {
        this.responseTable.remove(opaque);
    }
}            

```

NettyRemotingAbstract.invokeSyncImpl这里才是真正发送消息的逻辑，底层使用Netty通信。关于netty，我们知道在Netty中所有的IO操作都是异步的，因此，你不能立刻得知消息是否被正确处理，但是我们可以过一会等它执行完成或者直接注册一个监听，具体的实现就是通过Future和ChannelFuture,他们可以注册一个监听，当操作执行成功或失败时监听会自动触发。总之，所有的操作都会返回一个ChannelFuture。

这里为channelFuture配置了ChannelFutureListener监听器，监听channel的操作结果。如果消息发送成功，则设置ResponseFuture中SendRequestOK为true，return。否则设置为false，并且设置responseFuture中的responseCommand为null。

在设置了监听器之后，线程会阻塞等待：

```
RemotingCommand responseCommand = responseFuture.waitResponse(timeoutMillis);

---

this.countDownLatch.await(timeoutMillis, TimeUnit.MILLISECONDS);
return this.responseCommand;
```

达到了最大等待时间之后，返回responseCommand。如果为null，则抛出消息发送超时或者其他异常。

# 3 参考资料  
- [消息队列三种发送方式](http://jm.taobao.org/2016/07/21/3-ways-to-send-message/)










  
