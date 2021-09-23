



## **一、核心组件**

**1.SpringCloud Eureka 和Netflix Eureka的关系**

spring cloud eureka server和client是对netflix的eureka进行了封装，加了一些注解，对spring boot进行支持

https://github.com/spring-cloud/spring-cloud-netflix

https://github.com/Netflix/eureka

spring cloud Edgware.SR3对应的是netflix eureka的1.7.2的版本

**2.核心组件**

（1）eureka-client：这个就是指的eureka的客户端，注册到eureka上面去的一个服务，就是一个eureka client，无论是你要注册，还是要发现别的服务，无论是服务提供者还是服务消费者，都是一个eureka客户端。

（2）eureka-core：这个就是指的eureka的服务端，其实就是eureka的注册中心

（3）eureka-resources：这个是基于jsp开发的eureka控制台，web页面，上面你可以看到各种注册服务

（4）eureka-server：这是把eureka-client、eureka-core、eureka-resources打包成了一个war包，也就是说eureka-server自己本身也是一个eureka-client，同时也是注册中心，同时也提供eureka控制台。真正的使用的注册中心

（5）eureka-examples：eureka使用的例子

（6）eureka-test-utils：eureka的单元测试工具类

**3.eurka源码**

1. eureka server的启动，相当于是注册中心的启动 -> 启动的过程搞清楚，初始化了哪些东西
2. eureka client的启动，相当于是服务的启动，初始化了哪些东西
3. eureka运行的核心的流程
   1. eureka client往eureka server注册的过程，服务注册
   2. eureka client从eureka server获取注册表的过程,服务发现
   3. eureka client定时往eureka server发送续约通知（心跳）,服务心跳
   4. 服务实例摘除
   5. 通信
   6. 限流
   7. 自我保护
   8. server集群

