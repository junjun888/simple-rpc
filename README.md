#  手写简单 rpc 框架

[TOC]

## 0 缘起

> 一直对 rpc 的实现原理很好奇， 看了下别人实现的 rpc 框架， 感觉实现太复杂了，本文参考了 黄勇实现的 轻量级 rpc 框架，专注 rpc 核心功能。


## 1 技术选型
- 序列化协议 :Protostuff , 它基于 Protobuf 序列化框架，面向 POJO，无需编写 .proto 文件
- IOC框架：Spring, 后续考虑自己写一个
- 底层通信框架：netty,它使 NIO 编程更加容易，屏蔽了 Java 底层的 NIO 细节。
- 注册中心：ZooKeeper 提供服务注册与发现功能，开发分布式系统的必备选择，同时它也具备天生的集群能力
 

## 2 框架思路

该框架流程：

- 1 服务端启动, 和服务发布


```
 1 根据 配置的 zookeeper 信息， 初始化 zookeeper 的连接，
 
 2 获取 容器中 带有 RpcService 注解的 bean
 
 3 将 服务名 和 服务实体 放入 handlerMap
 
 4 初始化 netty 监听在指定端口
 
 5 遍历 handlerMap ， 向 zookeeper 中注册 服务节点信息
```


- 2 客户端调用
 

```
 1 初始化 serviceDiscovery, 用于在 zookeeper 中发现服务
 
 2 初始化 rpcProxy ， rpc 代理类， 通过这个代理类，可以创建指定服务的代理对象。
 
 3 调用 rpcProxy 的 create 方法，指定服务类， 初始化代理对象
 
 4 执行 生成代理对象的方法的时候， 会被拦截，通过上面指定的 服务类，去注册中心查询指定服务， 获取 服务的 ip 和 host
 
 5 封装请求，通过 netty 初始化一个连接 发送 该请求
 
 6 netty 接收到 请求以后，通过反射执行，并且返回结果

```

## 3 调用方式


### 3.1 定义服务

```
@RpcService(HelloService.class)
public class HelloServiceImpl implements HelloService {

    @Override
    public String hello(String name) {
        return "Hello! " + name;
    }
}
```

### 3.2 定义 bean


```
 <!-- 在 server 端 启用注册中心 -->
    <bean id="serviceRegistry" class="org.hwj.rpc.registry.center.zookeeper.ZooKeeperServiceRegistry">
        <constructor-arg name="zkAddress" value="${rpc.registry_address}"/>
    </bean>

    <bean id="rpcServer" class="org.hwj.rpc.server.RpcServer">
        <constructor-arg name="serviceAddress" value="${rpc.service_address}"/>
        <constructor-arg name="serviceRegistry" ref="serviceRegistry"/>
    </bean>
```

### 3.3 启动服务


```
new ClassPathXmlApplicationContext("spring.xml");
System.out.println("start server...");
```

### 3.4 服务调用


```
 ApplicationContext context = new ClassPathXmlApplicationContext("spring.xml");
 RpcProxy rpcProxy = context.getBean(RpcProxy.class);

 HelloService helloService = rpcProxy.create(HelloService.class);
 String result = helloService.hello("World");
 System.out.println(result);

```

## 4 完整代码

> https://github.com/junjun888/simple-rpc


## 5 参考

> https://my.oschina.net/huangyong/blog/361751

