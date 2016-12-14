---
title: RocketMQ源码学习之一-MAC单机环境搭建
date: 2016-11-12 10:48:59
tags: RocketMQ
---

本文讲述如何在单机（MAC）上搭建RocketMQ环境，实现正常收发消息。

# 1 安装JDK、maven等工具  
忽略。

# 2 部署RocketMQ服务端  
从[https://github.com/alibaba/RocketMQ](https://github.com/alibaba/RocketMQ)下载压缩包，并解压。

<!-- more -->

执行如下命令安装：  
sh install.sh  
![](http://o8sltkx20.bkt.clouddn.com/rocketmq-install-1.png)

进入devenv/bin目录修改相关文件的执行权限：  
cd /root/RocketMQ-3.4.6/devenv/bin  
chmod +x mqadmin mqbroker mqfiltersrv mqshutdown mqnamesrv

设置nameserver地址的环境变量：  
export NAMESRV_ADDR=192.168.2.101:9876(改为自己的机器的IP)  

然后接下来启动nameserver以及broker（注意一定要先启动nameserver，然后再启动broker）：  
nohup ./mqnamesrv >/var/log/mqname.log &  
nohup ./mqbroker >/var/log/mqbroker.log &  

相应的关闭命令如下：  
sh mqshutdown broker  
sh mqshutdown namesrv

观察/var/log/下的启动日志mqname.log、mqbroker.log发现有如下异常：  
```
ERROR: Please set the JAVA_HOME variable in your environment, We need Java(x64)! !!
```

打开启动脚本runserver.sh以及runbroker.sh文件，发现有如下三行： 
 
```
[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=$HOME/jdk/java

[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=/opt/taobao/java

[ ! -e "$JAVA_HOME/bin/java" ] && error_exit "Please set the JAVA_HOME variable in your environment, We need java(x64)!"

```

其中第二行是阿里巴巴集团内部服务器上的java目录，将这一行注释掉。然后第一行的JAVA_HOME的值改为自己机器的Java安装目录。然后再次起送mqnameserver以及mqbroker，观察日志发现启动成功：  
![](http://o8sltkx20.bkt.clouddn.com/rocketmq-install-2.png)

可以通过jps查看进程运行状态：  
![](http://o8sltkx20.bkt.clouddn.com/rocketmq-install-3.png)

# 3 运行Producer发送消息  
新建java maven工程，pom.xml需要增加对RocketMQ的如下依赖:  

```
      <dependency>
          <groupId>com.alibaba.rocketmq</groupId>
          <artifactId>rocketmq-client</artifactId>
          <version>3.5.9</version>
      </dependency>

      <dependency>
          <groupId>com.alibaba.rocketmq</groupId>
          <artifactId>rocketmq-srvutil</artifactId>
          <version>3.2.6</version>
      </dependency>
      
```

Producer.java:  

```
public class Producer {
  public static void main(String[] args) throws MQClientException, InterruptedException {
    DefaultMQProducer producer = new DefaultMQProducer("ProducerGroupName");
    producer.setNamesrvAddr("192.168.2.101:9876");
    producer.setInstanceName("Producer");
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

运行消息生产者，报错找不到对应topic的路由信息：  
![](http://o8sltkx20.bkt.clouddn.com/rocketmq-install-4.png)

原因因为需要先正确设置nameserver，然后启动mqnamesrv，最后启动mqbroker。  

之后再次运行消息producer，依然报错：消息发送失败，原因是连接失败。  
![](http://o8sltkx20.bkt.clouddn.com/rocketmq-install-5.png)  

发现连接的地址是ifonfig中的这个地址：  
![](http://o8sltkx20.bkt.clouddn.com/rocketmq-install-6.png)  

查看pid，发现是VPN应用进程：   
![](http://o8sltkx20.bkt.clouddn.com/rocketmq-install-7.png)  

杀掉进程。

然后重启mqsrv以及mqbroker，发现恢复正常：  
![](http://o8sltkx20.bkt.clouddn.com/rocketmq-install-8.png)  


再次运行Producer，消息发送成功：  
![](http://o8sltkx20.bkt.clouddn.com/rocketmq-install-9.png)

# 4 运行Consumer消费消息  
consumer.java:  

```
public class Consumer {

  public static void main(String[] args) throws InterruptedException, MQClientException {
    DefaultMQPushConsumer
        consumer = new DefaultMQPushConsumer("please_rename_unique_group_name_4");
    consumer.setVipChannelEnabled(false);
    consumer.setNamesrvAddr("192.168.2.101:9876");

    consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);

    consumer.subscribe("TopicTest", "*");

    consumer.registerMessageListener(new MessageListenerConcurrently() {

      public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs,
                                                      ConsumeConcurrentlyContext context) {
        System.out.println(Thread.currentThread().getName() + " Receive New Messages: " + msgs);
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
      }
    });

    consumer.start();

    System.out.println("Consumer Started.");
  }
```

运行consumer，消费消息成功：  
![](http://o8sltkx20.bkt.clouddn.com/rocketmq-install-10.jpeg)


# 5 一些有用的运维命令

- 查看集群情况  
/usr/local/alibaba-rocketmq/bin/mqadmin clusterList -n 127.0.0.1:9876
- 查看broker状态  
/usr/local/alibaba-rocketmq/bin/mqadmin brokerStatus -n 127.0.0.1:9876 -b 192.168.162.10:10911  
- 查看topic列表  
/usr/local/alibaba-rocketmq/bin/mqadmin topicList -n 127.0.0.1:9876
- 查看topic状态  
/usr/local/alibaba-rocketmq/bin/mqadmin topicStatus -n 127.0.0.1:9876 -t PushTopic
- 查看topic路由  
/usr/local/alibaba-rocketmq/bin/mqadmin topicRoute  -n 127.0.0.1:9876 -t PushTopic

# 6 参考资料

- [https://my.oschina.net/lenglingx/blog/734558](https://my.oschina.net/lenglingx/blog/734558)
- [https://github.com/alibaba/RocketMQ/issues/44](https://github.com/alibaba/RocketMQ/issues/44)
- [http://blog.chinaunix.net/uid-26111972-id-5750397.html](http://blog.chinaunix.net/uid-26111972-id-5750397.html)
- [http://www.tuicool.com/articles/mEBVneB](http://www.tuicool.com/articles/mEBVneB)
  
