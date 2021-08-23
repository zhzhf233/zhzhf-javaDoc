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

## **二、eureka server启动**

**1.eureka server项目预览**

**1）项目依赖：build.gradle**

![项目依赖](3-Eureka源码.assets/项目依赖.png)

1. eureka client：eureka server也是一个eureka client，因为后面我们讲到eureka server集群模式的时候，eureka server也要扮演eureka client的角色，往其他的eureka server上去注册。
2. eureka core：扮演了核心的注册中心的角色，接收别人的服务注册请求，提供服务发现的功能，保持心跳（续约请求），摘除故障服务实例。eureka server依赖eureka core的，基于eureka core的功能对外暴露接口，提供注册中心的功能
3. jersey框架：eureka server依赖jersey框架，你可以认为jersey框架类比于spring web mvc框架，支持mvc模式，支持restful http请求。jersey在国内几乎没什么公司使用，很少很少，在国外有一些公司用，netflix就会用，jersey去开发eureka。eureka里面，服务通信，都是基于http请求的，restful接口来通信的。eureka client和eureka server之间进行通信，都是基于jersey框架实现http restful接口请求和调用的。eureka-client-jersey2，eureka-core-jersey2，猜测一下，就知道，其实这两个工程，就是eureka为了方便自己，对jersey框架的一个封装，提供更多的功能，方便自己使用。
4. mocktio：mock测试框架，之前项目阶段一里面，我们大量用了mockito框架，来写单元测试。在eureka框架里面，他每个工程都是有src/test/java的哦，里面都写了针对自己本工程的单元测试，必须得写。mock就是用的mockito框架。
5. jetty：方便你测试的，测试的时候，是会基于jetty直接将eureka server作为一个web应用给跑起来，jetty作为web容器，跑起来eureka server这个web应用。跑起来之后，eureka server不就活在jetty web容器里面了，jersey对外暴露了一些restful接口，然后测试类里，就可以基于jersey的客户端，发送http请求，调用eureka server暴露的restful接口，测试比如说：服务注册、心跳、服务实例摘除，等等功能。

![打包](3-Eureka源码.assets/打包.png)

1. 把这个eureka-server工程，打成一个war包，会怎么样，会将eureka-resources下面的那些jsp、js、css给搞到这个war包里面去，然后就可以跑起来，提供一个index页面。我们之前启动了eureka-server之后，第一件事儿就是访问他的控制台，可以去看看有谁来注册了。控制台的代码，就在jsp里面，就是eureka-resources的jsp提供的，人家还有对应的css和js呢。
2. 人家eureka-server的web.xml里，设置好了welcome-file-list，就是status.jsp页面，这个页面就会在打war包的时候，从eureka-resources里面给打到eureka-server里的，eureka-server里就会包含jsp目录，里面就有这个status.jsp。我们之前看到的那个eureka控制台的页面，就是这个status.jsp，里面一堆jsp代码，把你注册的服务的信息，全部都提出来，显示到页面上去。

**2）eureka server项目本质**

1. eureka-server的本质，就是一个web应用，跟你用spring mvc+spring+mybatis+jsp写出来的这个web应用没两样。你看着这个东西有点陌生，只不过是因为人家用了gralde来构建，国内很少用gradle来构建的。
2. 分析一下eureka-server的web.xml，你呢已经知道eureka-server就是个web应用的本质，无论是用tomcat还是jetty都可以启动，如果用tomcat启动，你就用gradle把这个eureka-server打成一个war包，让tomcat的webapps里面，启动tomcat就ok了。如果是用jetty的话，人家是在测试代码里用jetty去启动这个web应用的。

**3）web.xml**

![listener](3-Eureka源码.assets/listener.png)

1. 最最重要的就是listener，listener是在web应用启动的时候就会执行的，负责对这个web应用进行初始化的事儿，我们如果自己写个web应用，也经常会写一个listener，在里面搞一堆初始化的代码，比如说，启动一些后台线程，加载个配置文件。com.netflix.eureka.EurekaBootStrap

![filter1](3-Eureka源码.assets/filter1.png)

1. 又连着4个Filter，任何一个请求都会经过这些filter，这些filter会对每个请求都进行处理，这个4个filter都在eureka-core里面

   （1）StatusFilter：负责状态相关的处理逻辑

   （2）ServerRequestAuthFilter：一看就是，对请求进行授权认证的处理的

   （3）RateLimitingFilter：负责限流相关的逻辑的（很有可能成为eureka-server里面的一个技术亮点，看看人家eureka-server作为一个注册中心，是怎么做限流的，先留意算法是什么，留到后面去看）

   （4）GzipEncodingEnforcingFilter：gzip，压缩相关的；encoding，编码相关的

![filter2](3-Eureka源码.assets/filter2.png)

1. jersey框架的一个ServletContainer的一个filter，我可以告诉大家，类似于每个mvc框架，比如说struts2和spring web mvc，都会搞一个自己的核心filter，或者是核心servlet，配置在web.xml里，用了框架之后，相当于就是将web请求的处理入口交给框架了，框架会根据你的配置，自动帮你干很多事儿，最后调用你的一些处理逻辑。
2. jersey也是一样的，这里的这个ServletContainer就是一个核心filter，接收所有的请求，作为请求的入口，处理之后来调用你写的代码逻辑。

![filter-mapping](3-Eureka源码.assets/filter-mapping.png)

1. filter-mapping是配置这些filter在哪些目录下生效。
   1. StatusFilter和RequestAuthFilter，一看就是通用的处理逻辑，是对所有的请求都开放的
   2. RateLimitingFilter，默认是不开启的，如果你要打开eureka-server内置的限流功能，你需要自己把RateLimitingFilter的的注释打开，让这个filter生效
   3. GzipEncodingEnforcingFilter，/v2/apps相关的请求，会走这里，仅仅对部分特殊的请求生效
   4. jersey核心filter，是拦截所有的请求的

![welcome-file-list](3-Eureka源码.assets/welcome-file-list.png)

1. welcome-file-list，是配置了status.jsp是欢迎页面，首页，eureka-server的控制台页面，展示注册服务的信息