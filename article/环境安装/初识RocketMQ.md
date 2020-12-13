# 初识RocketMQ

### RocketMQ简介


RocketMQ是一个纯 Java、分布式、队列模型的开源消息中间件，前身是MetaQ，是阿里参考 Kafka 特点研发的一个队列模型的消息中间件，后开源给 apache 基金会成为了 apache 的顶级开源项目，具有高性能、高可靠、高实时、分布式特点。


废话不多说，先搞一个 demo 玩玩。


### SpringBoot使用RocketMQ


#### 安装RocketMQ


> 下载地址：[http://rocketmq.apache.org/dowloading/releases/](http://rocketmq.apache.org/dowloading/releases/)



```bash
# 将下载好的压缩包拷贝到Liunx服务器上

# 解压
unzip rocketmq-all-4.7.1-bin-release.zip

# 进入rocketmq工作目录
cd /opt/rocketmq-all-4.7.1-bin-release/bin/

# 首先启动Name Server
nohup sh mqnamesrv &

# 验证Name Server 是否启动成功
tail -f ~/logs/rocketmqlogs/namesrv.log
The Name Server boot success...

# 然后启动Broker
cd ../
nohup sh bin/mqbroker -n localhost:9876 &

# 验证Name Server 是否启动成功,例如Broker的IP为：192.168.223.137,且名称为localhost.localdomain
tail -f ~/logs/rocketmqlogs/Broker.log 
The broker[localhost.localdomain, 192.168.223.137:10911] boot success...
```


#### 新建SpringBoot项目测试


1. pom.xml添加依赖
```xml
<dependency>
	<groupId>org.apache.rocketmq</groupId>
	<artifactId>rocketmq-client</artifactId>
	<version>4.3.0</version>
</dependency>
```

2. 编写生产者类RocketMqProducerController
```java
package com.cxhello.demo.controller;

import org.apache.rocketmq.client.producer.DefaultMQProducer;
import org.apache.rocketmq.client.producer.SendResult;
import org.apache.rocketmq.common.message.Message;
import org.apache.rocketmq.remoting.common.RemotingHelper;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.ResponseBody;

/**
 * @author cxhello
 * @create 2020/11/14 21:38
 */
@Controller
public class RocketMqProducerController {

    @GetMapping("/send")
    @ResponseBody
    public String send() throws Exception {
        // 实例化消息生产者Producer
        DefaultMQProducer producer = new DefaultMQProducer("producer_test_group");
        // 设置NameServer的地址
        producer.setNamesrvAddr("192.168.223.137:9876");
        // 启动Producer实例
        producer.start();
        for (int i = 0; i < 5; i++) {
            // 创建消息，并指定Topic，Tag和消息体
            Message msg = new Message("TopicTest" /* Topic */,
                    "TagA" /* Tag */,
                    ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET) /* Message body */
            );
            // 发送消息到一个Broker
            SendResult sendResult = producer.send(msg);
            // 通过sendResult返回消息是否成功送达
            System.out.printf("%s%n", sendResult);
        }
        // 如果不再发送消息，关闭Producer实例。
        return "success";
    }
}
```
> 访问连接测试生产消息：[http://localhost:8080/send](http://localhost:8080/send)

![image.png](https://cdn.nlark.com/yuque/0/2020/png/2584604/1605366920815-48c1abed-4a51-4ccd-8083-474fd9ae44cf.png#align=left&display=inline&height=183&margin=%5Bobject%20Object%5D&name=image.png&originHeight=183&originWidth=1801&size=55880&status=done&style=none&width=1801)

3. 编写消费者类
```java
package com.cxhello.demo;

import org.apache.rocketmq.client.consumer.DefaultMQPushConsumer;
import org.apache.rocketmq.client.consumer.listener.ConsumeConcurrentlyContext;
import org.apache.rocketmq.client.consumer.listener.ConsumeConcurrentlyStatus;
import org.apache.rocketmq.client.consumer.listener.MessageListenerConcurrently;
import org.apache.rocketmq.client.exception.MQClientException;
import org.apache.rocketmq.common.message.MessageExt;
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;

import java.util.List;

/**
 * @author cxhello
 * @create 2020/11/14 21:59
 */
@Component
public class RocketMqConsumer implements CommandLineRunner {

    public void messageListener() throws MQClientException {
        // 实例化消费者
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("consumer_test_group");

        // 设置NameServer的地址
        consumer.setNamesrvAddr("192.168.223.137:9876");

        // 订阅一个或者多个Topic，以及Tag来过滤需要消费的消息
        consumer.subscribe("TopicTest", "*");
        // 注册回调实现类来处理从broker拉取回来的消息
        consumer.registerMessageListener(new MessageListenerConcurrently() {
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
                System.out.printf("%s Receive New Messages: %s %n", Thread.currentThread().getName(), msgs);
                // 标记该消息已经被成功消费
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
        // 启动消费者实例
        consumer.start();
        System.out.printf("Consumer Started.%n");
    }

    @Override
    public void run(String... args) throws Exception {
        this.messageListener();
    }
}
```
启动项目即可
![image.png](https://cdn.nlark.com/yuque/0/2020/png/2584604/1605367068454-8d05ef27-8d1d-4725-bd13-5b6154dacf7d.png#align=left&display=inline&height=131&margin=%5Bobject%20Object%5D&name=image.png&originHeight=131&originWidth=1811&size=40311&status=done&style=none&width=1811)


### RocketMQ监控平台搭建


```bash
# 下载并编译打包
git clone https://github.com/apache/rocketmq-externals
cd rocketmq-console
mvn clean package -Dmaven.test.skip=true

# 注意：打包前在rocketmq-console中配置namesrv集群地址
rocketmq.config.namesrvAddr=192.168.223.137:9876

# 启动rocketmq-console
nohup java -jar rocketmq-console-ng-1.0.0.jar &
```
> 访问连接：[http://192.168.223.137:8080/](http://192.168.223.137:8080/)

![image.png](https://cdn.nlark.com/yuque/0/2020/png/2584604/1605368969386-560c17f9-eca9-443d-9f16-f0cf4a6aa441.png#align=left&display=inline&height=935&margin=%5Bobject%20Object%5D&name=image.png&originHeight=935&originWidth=1900&size=97650&status=done&style=none&width=1900)
![image.png](https://cdn.nlark.com/yuque/0/2020/png/2584604/1605369139180-2c53a88e-93f6-4950-b871-f5e4688bf5c5.png#align=left&display=inline&height=252&margin=%5Bobject%20Object%5D&name=image.png&originHeight=252&originWidth=1908&size=39346&status=done&style=none&width=1908)
![image.png](https://cdn.nlark.com/yuque/0/2020/png/2584604/1605369124022-fe281dd6-6c1b-4e79-ad6e-b7e87570d3f9.png#align=left&display=inline&height=420&margin=%5Bobject%20Object%5D&name=image.png&originHeight=420&originWidth=1910&size=47231&status=done&style=none&width=1910)
### 踩坑记录


#### 启动 Broken 报内存不足的错误


```bash
修改MQ安装目录bin下面的runserver.sh,将内存调到你机器能够承受的内存大小
JAVA_OPT="${JAVA_OPT} -server -Xms512m -Xmx512m -Xmn256m"
```
#### 
#### 使用 RocketMQ 发送消息时报错 No route info of this topic，详细错误信息如下：


```bash
org.apache.rocketmq.client.exception.MQClientException: No route info of this topic, TopicTest
See http://rocketmq.apache.org/docs/faq/ for further details.
	at org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl.sendDefaultImpl(DefaultMQProducerImpl.java:610) ~[rocketmq-client-4.3.0.jar:4.3.0]
	at org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl.send(DefaultMQProducerImpl.java:1223) ~[rocketmq-client-4.3.0.jar:4.3.0]
	at org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl.send(DefaultMQProducerImpl.java:1173) ~[rocketmq-client-4.3.0.jar:4.3.0]
	at org.apache.rocketmq.client.producer.DefaultMQProducer.send(DefaultMQProducer.java:214) ~[rocketmq-client-4.3.0.jar:4.3.0]
	at com.cxhello.demo.controller.RocketMqProducerController.send(RocketMqProducerController.java:34) ~[classes/:na]
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[na:1.8.0_131]
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62) ~[na:1.8.0_131]
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:1.8.0_131]
	at java.lang.reflect.Method.invoke(Method.java:498) ~[na:1.8.0_131]
	at org.springframework.web.method.support.InvocableHandlerMethod.doInvoke(InvocableHandlerMethod.java:197) ~[spring-web-5.3.1.jar:5.3.1]
	at org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(InvocableHandlerMethod.java:141) ~[spring-web-5.3.1.jar:5.3.1]
	at org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:106) ~[spring-webmvc-5.3.1.jar:5.3.1]
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandlerMethod(RequestMappingHandlerAdapter.java:893) ~[spring-webmvc-5.3.1.jar:5.3.1]
	at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.handleInternal(RequestMappingHandlerAdapter.java:807) ~[spring-webmvc-5.3.1.jar:5.3.1]
	at org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(AbstractHandlerMethodAdapter.java:87) ~[spring-webmvc-5.3.1.jar:5.3.1]
	at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:1061) ~[spring-webmvc-5.3.1.jar:5.3.1]
	at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:961) ~[spring-webmvc-5.3.1.jar:5.3.1]
	at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:1006) ~[spring-webmvc-5.3.1.jar:5.3.1]
	at org.springframework.web.servlet.FrameworkServlet.doGet(FrameworkServlet.java:898) ~[spring-webmvc-5.3.1.jar:5.3.1]
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:626) ~[tomcat-embed-core-9.0.39.jar:4.0.FR]
	at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:883) ~[spring-webmvc-5.3.1.jar:5.3.1]
	at javax.servlet.http.HttpServlet.service(HttpServlet.java:733) ~[tomcat-embed-core-9.0.39.jar:4.0.FR]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:231) ~[tomcat-embed-core-9.0.39.jar:9.0.39]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166) ~[tomcat-embed-core-9.0.39.jar:9.0.39]
	at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:53) ~[tomcat-embed-websocket-9.0.39.jar:9.0.39]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193) ~[tomcat-embed-core-9.0.39.jar:9.0.39]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166) ~[tomcat-embed-core-9.0.39.jar:9.0.39]
	at org.springframework.web.filter.RequestContextFilter.doFilterInternal(RequestContextFilter.java:100) ~[spring-web-5.3.1.jar:5.3.1]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:119) ~[spring-web-5.3.1.jar:5.3.1]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193) ~[tomcat-embed-core-9.0.39.jar:9.0.39]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166) ~[tomcat-embed-core-9.0.39.jar:9.0.39]
	at org.springframework.web.filter.FormContentFilter.doFilterInternal(FormContentFilter.java:93) ~[spring-web-5.3.1.jar:5.3.1]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:119) ~[spring-web-5.3.1.jar:5.3.1]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193) ~[tomcat-embed-core-9.0.39.jar:9.0.39]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166) ~[tomcat-embed-core-9.0.39.jar:9.0.39]
	at org.springframework.web.filter.CharacterEncodingFilter.doFilterInternal(CharacterEncodingFilter.java:201) ~[spring-web-5.3.1.jar:5.3.1]
	at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:119) ~[spring-web-5.3.1.jar:5.3.1]
	at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193) ~[tomcat-embed-core-9.0.39.jar:9.0.39]
	at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166) ~[tomcat-embed-core-9.0.39.jar:9.0.39]
	at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:202) ~[tomcat-embed-core-9.0.39.jar:9.0.39]
	at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:97) [tomcat-embed-core-9.0.39.jar:9.0.39]
	at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:542) [tomcat-embed-core-9.0.39.jar:9.0.39]
	at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:143) [tomcat-embed-core-9.0.39.jar:9.0.39]
	at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:92) [tomcat-embed-core-9.0.39.jar:9.0.39]
	at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:78) [tomcat-embed-core-9.0.39.jar:9.0.39]
	at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:343) [tomcat-embed-core-9.0.39.jar:9.0.39]
	at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:374) [tomcat-embed-core-9.0.39.jar:9.0.39]
	at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:65) [tomcat-embed-core-9.0.39.jar:9.0.39]
	at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:868) [tomcat-embed-core-9.0.39.jar:9.0.39]
	at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1590) [tomcat-embed-core-9.0.39.jar:9.0.39]
	at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49) [tomcat-embed-core-9.0.39.jar:9.0.39]
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142) [na:1.8.0_131]
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617) [na:1.8.0_131]
	at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61) [tomcat-embed-core-9.0.39.jar:9.0.39]
	at java.lang.Thread.run(Thread.java:748) [na:1.8.0_131]
```
出现上面的错误，主要有三种情况


1. Broker模块不支持自动创建 topic，并且 topic 没有被手动创建过；
1. Broker模块没有正确连接到 nameserver；
1. 发送者没有连接到 namerserver；



第一种情况是 topic 确实不存在，可以查看 broker 的日志确定 topic 是否真的存在
```bash
# 登录MQ所在的服务器,执行如下命令（把TopicTest换成你的topic名称）
cat ~/logs/rocketmqlogs/broker.log  | grep topicName=TopicTest

# 如果没有出现,说明topic确实不存在,可以通过自动创建topic或手动创建topic解决

# 自动创建topic:启动broker时加上自动创建topic的参数,如下,其中autoCreateTopicEnable=true表示自动创建topic,但我使用了这个命令还是不行,后续再搞吧,我最后用的是手动创建topic的方法解决的
nohup sh bin/mqbroker -n localhost:9876 autoCreateTopicEnable=true &

# 进入rocketmq的安装目录,手动创建topic
sh mqadmin updateTopic -b localhost:10911 -t TopicTest
```
第二种情况出现的概率较低，可以采用以下两种方式确认
```bash
# 查看broker的日志,出现如下内容,说明连接成功
cat ~/logs/rocketmqlogs/broker.log | grep register

2019-12-04 09:40:16 INFO brokerOutApi_thread_1 - register broker to name server localhost:9876 OK 2019-12-04 09:40:46 INFO brokerOutApi_thread_2 - register broker to name server localhost:9876 OK

# 在rocketmq的安装目录执行如下命令
sh mqadmin clusterList -n localhost:9876
```
出现如下内容说明连接成功,没有问题
![image.png](https://cdn.nlark.com/yuque/0/2020/png/2584604/1605368567653-3fb73a4a-4962-4e56-962b-75fe078d75e6.png#align=left&display=inline&height=105&margin=%5Bobject%20Object%5D&name=image.png&originHeight=105&originWidth=1256&size=19686&status=done&style=none&width=1256)
第三种情况出现的最大可能是发送者和 rocketmq 服务器之间的网络或端口不通，一定要记得关闭 rocketmq 服务器的防火墙
```bash
systemctl stop firewalld.service
```


### 参考链接


> [https://zhuanlan.zhihu.com/p/103391787](https://zhuanlan.zhihu.com/p/103391787)
> [https://www.cnblogs.com/lijiayong/p/13171822.html](https://www.cnblogs.com/lijiayong/p/13171822.html)
> [https://juejin.im/post/6854573214988222472](https://juejin.im/post/6854573214988222472)

