---
layout:     post
title:      spring cloud stream 的封装与使用
subtitle:   根据自身业务封装spring cloud stream 
date:       2021-07-20
author:     zj
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - 实战经验

---

   

##### Spring Cloud Stream 

​    spring cloud stream 个人理解就是将消息中间件进行抽象为输入通道和输出通道，开发者使用输入输出通道发送接收消息，从而屏蔽或降低开发人员对不同消息中间件的认知和学习，开发人员不用关注不同消息中间件的特定api,减少业务代码对中间件依赖

![img](https://gitee.com/zjhy/PicGo/raw/master/img/20210720142250.png)



##### 封装的目的

- 代码结构统一，spring cloud stream 开发比较自由，不利于代码维护
- 明确消息的流向和来源，通常使用mq 无法明确消息的流向和来源导致无法快速定位系统之间的联系（这个消   息是那个系统发送来的？发送的这个消息是谁再使用）
- 统一配置管理
- 消息消费失败统一处理

​    

##### 封装原理

- 自定义注解【@HrmsMqListener】添加系统枚举字段区分消息来源，并将自定义注解动态绑定至spring cloud stream 原始注解【@StreamListener】上，不修改框架原始逻辑兼容自定义注解
- 利用注解【@StreamListener】支持条件过滤消息，在自定义注解【@HrmsMqListener】中添加话题字段【topic】区分同一系统不同类型消息，并且兼容原【@StreamListener】注解的消息过滤功能
- 向spring IOC 容器中动态生成消息生产者实例，减少开发人员stream发送消息方法编写，从而可以使用统一消息发送工具类发送消息
- 拦截消费者消息消费失败方法，将消费失败的消息及原因统一持久化至数据库，容错消息丢失，消息消费失败问题快速定位

##### 使用方式

- maven 依赖

  ```xml
  <dependency>
      <groupId>com.hrms</groupId>
      <artifactId>mq-spring-boot-starter</artifactId>
      <version>1.0-SNAPSHOT</version>
  </dependency>
  <dependency>
      <groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
      <scope>runtime</scope>
      <version>8.0.21</version>
  </dependency>
  <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
      <version>${spring.cloud.stream.version}</version>
  </dependency>
  <!--<dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-stream-kafka</artifactId>
      <version>${spring.cloud.stream.version}</version>
  </dependency>-->
  ```

  如果开启消息消费失败持久化功能，那么需要引入数据库驱动，需要根据情况引入所需驱动程序如mysql、oracle 等，根据不同的消息中间件引入不同的spring-starer 比如rabbit 或者kafka

- 配置文件

  ```yaml
  spring:
    cloud:
      stream:
        kafka:
          binder:
            brokers:
              - localhost:9092
        rabbit:
          host: http://localhost:15672
          username: guest
          password: guest
  
  
        bindings:
          hrms-check-order-output:
            destination: test1
            group: test
            content-type: application/json  #消息传输格式
          hrms-check-order-input:
            destination: test1
            group: test
            content-type: application/json
            consumer:
              concurrency: 1 #消费者并发数
              max-attempts: 1 #消息消费失败重试次数
              auto-bind-dlq: true # 死信队列
              dlq-ttl: 50000 #死信队列消息存活时间
              republish-to-dlq: true #该消息在进入到死信队列的时候，会在headers中加入错误信息 (版本不支持)
          hrms-check-prize-output:
            destination: test2
            group: test
            content-type: application/json  #消息传输格式
          hrms-check-prize-input:
            destination: test2
            group: test
            content-type: application/json
            consumer:
              concurrency: 1 #消费者并发数
              max-attempts: 1 #消息消费失败重试次数
              auto-bind-dlq: true # 死信队列
              dlq-ttl: 50000 #死信队列消息存活时间
              republish-to-dlq: true #该消息在进入到死信队列的时候，会在headers中加入错误信息 (版本不支持)
  
  hrms:
    frame:
      mq:
        ds:
          enable: true
          url: jdbc:mysql:xxx
          driver: com.mysql.cj.jdbc.Driver
          username: xxx
          password: xxx
          initial-size: 3
          min-idle: 30
          max-active: 100
          max-wait: 9000
  
  ```

  以上配置为示例，具体消息的配置将由配置中心统一管理，其它服务使用同一份配置即可，配置启用kafka还是rabbit 将由引入的maven spring stream starter 决定

- 代码示例

  *绑定消息通道*

  ```java
  //根据需要发送和接收mq消息的服务引入需要的通道绑定配置类
  //HrmsSink 的子接口为消息接收绑定类，HrmsSource 的子接口为消息发送绑定类
  // Processor 结尾继承HrmsSink和HrmsSource的子接口的类为消息接收和发送绑定类
  //例如订单校验服务自产自销mq消息那么引入 HrmsCheckOrderProcessor类即可（或者引入HrmsCheckOrderSink, HrmsCheckOrderSource两个绑定类）   
  @EnableBinding({HrmsCheckOrderProcessor.class, HrmsCheckPrizeProcessor.class})
  @SpringBootApplication
  public class Sender {
  
      public static void main(String[] args) {
          SpringApplication.run(Sender.class, args);
      }
  }
  ```

  *消息发送*

  ```java
  //向订单校验服务的topic=TEST 的话题发送普通字符串
          boolean s1 = HrmsMqMessageSendUtil.send(HrmsMqSysName.HRMS_CHECK_ORDER, "TEST", "Hello hrms-mq");
          //发送奖等校验服务发送topic=PRO 的话题发送自定义消息头和实体bean消息（发送实体对象时务必存在无参构造器和属性getter/setter）
          HashMap<String, Object> map = new HashMap<>();
          map.put("prizeOrder", "abcde1234");
          boolean s2 = HrmsMqMessageSendUtil.send(HrmsMqSysName.HRMS_CHECK_PRIZE, "PRO", map, new Person("郑军", 28));
  ```

  *消息接收*

  ```java
   //接收来自订单校验服务topic=TEST 的消息
      @HrmsMqListener(value = HrmsMqSysName.HRMS_CHECK_ORDER, topic = "TEST")
      public void listenerOrder(String message) {
          System.out.println("接收到订单校验系统发送的消息：[" + message + "]");
      }
  
      //接收来自计奖校验服务topic=TEST 的消息（包括消息内容，和消息头信息）
      @HrmsMqListener(value = HrmsMqSysName.HRMS_CHECK_PRIZE, topic = "TEST")
      public void listenerPrize(Person person, MessageHeaders headers) {
          System.out.println("接收到计奖校验系统发送的消息头：[" + headers + "]");
          System.out.println("接收到计奖校验系统发送的消息体：[" + person + "]");
      }
  ```



##### <font color=red>注意</font>

- 封装的starter 与原始spring cloud stream 兼容，但是推荐使用hrms 自定义注解和api 接收和发送消息，便于代码统一和维护，以及消息丢失容错处理。
- 发送和接收消息内容为javabean 实体时务必存在无参构造器和成员属性的getter/setter 方法
- 消息头中的【HRMS_TOPIC】和【HRMS_MS_ID】为封装starter 占有字段，请不要修改或者它用，详情查看HrmsMqConstant类
- 对于不支持发送mq 消息的服务，联系管理员添加对应配置，或者按照HrmsSink与HrmsSource接口中的规则进行配置升级starter
- 消息消费方法一定不要自行处理（或者吃掉）异常，发生异常及时向外抛出，否则会影响mq消息重试发送，和消息失败持久化入库功能

