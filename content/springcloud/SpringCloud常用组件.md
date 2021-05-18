```json
{  
"date": "2021.05.19 00:01",  "tags": ["java","springcloud"],  "description":""
}
```

# 注册中心Eureka
Eureka包含两个组件：Eureka Server和Eureka Client，是Netflix提供的一款用于服务注册和服务发现的产品，springcloud最核心的组件之一。
通常，Eureka Server简称为server，Eureka Client简称为微服务。我们可以认为，springcloud中，除了Eureka server，其他所有的东西包括Zuul网关都是Eureka client。
多实例的Eureka server之间会相互同步他们保存的元信息，并最终达到一致。

元信息：Eureka client注册到server上面提供的信息，server会持有整个系统所有微服务的信息。

### Eureka Client: 

服务注册：client在启动之后，向server注册，把自身的信息告诉server 

心跳续约：client和server之间维护的定时的心跳，目的是告诉server，当前client还存活，可以认为是一个ping的过程 

下线：client关闭之后，通知server删除相关的元信息 

获取服务注册信息：获取服务注册元信息

```java
ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>>
```

Eureka使用一个嵌套的map来管理元信息，第一层K存储应用名称，每一个微服务都是一个应用；第二层的K是实例的名称，应用可以包含多个实例，value例如实例的ip地址，端口号等等

# 网关Zuul

# 微服务调用Feign（Ribbon）

# 断路器Hystrix
