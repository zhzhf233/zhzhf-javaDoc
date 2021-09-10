# Eureka启动全量抓取注册表

## 一、Eureka Client

1. 在eureka client第一次启动的时候，会在初始化服务时全量抓取一次注册表。

![image-20210910072907322](7-Eureka启动全量抓取注册表.assets/image-20210910072907322.png)

2. 在fetchRegistry()方法中，首先会去获取eureka client服务本地缓存的注册表，第一次启动时肯定是没有的。

   如果时增量更新注册表被禁用了，或者第一次拉取注册表，则去获取全量的注册表。

![image-20210910073534190](7-Eureka启动全量抓取注册表.assets/image-20210910073534190.png)

3. 获取全量注册表getAndStoreFullRegistry()方法中
   1. 首先判断registryRefreshSingleVipAddress是否配置了参数，该配置表示该客户端服务只对单一的VIP注册表信息感兴趣，只获取单一的服务地址的注册表，默认为null。
   2. registryRefreshSingleVipAddress为null，则通过http请求去访问GET请求http://localhost:8080/eureka/v2/apps抓取全量的注册表。对应eureka-core包resources目录下的ApplicationsResource类下的getContainers()方法。
   3. 最后对抓取到的注册表set到本地。

![image-20210910074003394](7-Eureka启动全量抓取注册表.assets/image-20210910074003394.png)

## 二、Eureka server

1. Eureka Client抓取注册表调用的接口对应eureka-core包resources目录下的ApplicationsResource类下的getContainers()方法。在该接口方法中，会去缓存中加载注册表，返回给eureka client。

![image-20210910080708178](7-Eureka启动全量抓取注册表.assets/image-20210910080708178.png)

2. eureka server使用了多级缓存的机制来读取注册表信息。用了两个map，只读缓存的map和读写缓存的map，做了两级缓存。
   - 下面getValue()方法，就是先从只读缓存中去读，读取不到再去读写缓存中读取。useReadOnlyCache参数通过配置文件shouldUseReadOnlyResponseCache配置，默认为true。
   - getValue()方法通过两级缓存都读取不到，则会从eureka server的注册表中读取，并放入读写缓存和只读缓存中。

![image-20210910081108064](7-Eureka启动全量抓取注册表.assets/image-20210910081108064.png)

