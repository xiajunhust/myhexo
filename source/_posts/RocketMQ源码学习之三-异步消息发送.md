---
title: RocketMQ源码学习之三-异步消息发送
date: 2016-11-14 22:05:07
tags: RocketMQ
---

本文主要介绍RocketMQ异步消息发送的原理。

# 1 异步消息发送Demo  
首先看一段RocketMQ异步消息发送的Demo代码：  
消息生产者：  

```
import com.alibaba.rocketmq.client.exception.MQClientException;
import com.alibaba.rocketmq.client.producer.DefaultMQProducer;
import com.alibaba.rocketmq.common.message.Message;
import com.SpringTest.rocketmq.callback.MySendCallback;

public class Producer3 {
  public static void main(String[] args) throws MQClientException, InterruptedException {
    DefaultMQProducer producer = new DefaultMQProducer("ProducerGroupName");
    producer.setNamesrvAddr("192.168.2.104:9876");
    producer.setInstanceName("Producer3");
    producer.start();
    for (int i = 0; i < 1; i++)
      try {
        {
          Message msg = new Message("TopicTest",// topic
                                    "TagA",// tag
                                    "OrderID188",// key
                                    ("Hello MetaQ").getBytes());// body

          producer.send(msg, new MySendCallback());
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

回调接口实现：

```
public class  MySendCallback implements SendCallback {

  public void onSuccess(final SendResult sendResult) {
    System.out.println("SendCallback-发送消息成功! sendResult=" + sendResult.toString());
  }


  public void onException(final Throwable e) {
    System.out.println("SendCallback-发送消息失败! 异常=" + e.toString());
  }
}
```

消息消费者和前一篇消息同步发送博文中示例相同。运行结果：  

```
生产者：
SendCallback-发送消息成功! sendResult=SendResult [sendStatus=SEND_OK, msgId=C0A802684CCA1218025C454271C80000,offsetMsgId=C0A8026800002A9F0000000000001AC9, messageQueue=MessageQueue [topic=TopicTest, brokerName=xiajuns-MacBook-Pro.local, queueId=2], queueOffset=0]

消费者：
ConsumeMessageThread_2 Receive New Messages: [MessageExt [queueId=2, storeSize=189, queueOffset=0, sysFlag=0, bornTimestamp=1479134782410, bornHost=/192.168.2.104:60966, storeTimestamp=1479134782449, storeHost=/192.168.2.104:10911, msgId=C0A8026800002A9F0000000000001AC9, commitLogOffset=6857, bodyCRC=1751783629, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message [topic=TopicTest, flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=1, KEYS=OrderID188, CONSUME_START_TIME=1479134782503, UNIQ_KEY=C0A802684CCA1218025C454271C80000, WAIT=true, TAGS=TagA}, body=11]]]

```

# 2 异步消息发送原理以及应用场景    
异步消息发送是指消息生产者发送了一条消息之后，不用等待此条消息的发送结果，而继续发送下一条消息。需要使用方自行实现回调接口SendCallback。发送端无需等待服务端响应结果，通过回调接口对服务器的响应进行处理。

![异步消息发送原理示意图](http://o8sltkx20.bkt.clouddn.com/rocketmq-sourcecode-3-1.png)

使用场景：  
异步发送一般用于链路耗时较长，对 RT 响应时间较为敏感的业务场景，例如用户视频上传后通知启动转码服务，转码完成后通知推送转码结果等。

# 3 异步消息发送源码分析  
异步消息发送的整体发送流程与[RocketMQ同步消息发送](http://xiajunhust.github.io/2016/11/12/RocketMQ%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0%E4%B9%8B%E4%BA%8C-%E5%90%8C%E6%AD%A5%E6%B6%88%E6%81%AF%E5%8F%91%E9%80%81/)流程相同。不同之处在于，DefaultMQProducer.send(Message msg)调到DefaultMQProducerImpl的这个方法传递的CommunicationMode参数是ASYNC。最后会调到底层的Netty通信逻辑NettyRemotingClient:   

```
public void invokeAsyncImpl(final Channel channel, final RemotingCommand request, final long timeoutMillis,
                            final InvokeCallback invokeCallback)
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
                    	 //执行回调接口
                        responseFuture.executeInvokeCallback();
                    } catch (Throwable e) {
                        plog.warn("excute callback in writeAndFlush addListener, and callback throw", e);
                    } finally {
                        responseFuture.release();
                    }

                    plog.warn("send a request command to channel <{}> failed.", RemotingHelper.parseChannelRemoteAddr(channel));
                }
            });
        } catch (Exception e) {
            responseFuture.release();
            plog.warn("send a request command to channel <" + RemotingHelper.parseChannelRemoteAddr(channel) + "> Exception", e);
            throw new RemotingSendRequestException(RemotingHelper.parseChannelRemoteAddr(channel), e);
        }
    } else {
        String info =
                String.format("invokeAsyncImpl tryAcquire semaphore timeout, %dms, waiting thread nums: %d semaphoreAsyncValue: %d", //
                        timeoutMillis, //
                        this.semaphoreAsync.getQueueLength(), //
                        this.semaphoreAsync.availablePermits()//
                );
        plog.warn(info);
        throw new RemotingTooMuchRequestException(info);
    }
}
```

这里同样是注册了监听器ChannelFutureListener，在监听器中设置消息发送结果，与同步发送不同的是会执行用户自己实现的回调接口，完成消息发送后的处理逻辑.  
并且没有同步发送消息的awaiting逻辑，整个发送消息发送立即返回。

# 4 参考资料  
- [消息队列三种发送方式](http://jm.taobao.org/2016/07/21/3-ways-to-send-message/)



