
@[TOC](Eureka 注册中心/服务发现框架)

# Eureka注册中心/服务发现框架

Eureka是Netflix开发的服务发现框架，本身是一个基于REST的服务，主要用于定位运行在AWS域中的中间层服务，以达到负载均衡和中间层服务故障转移的目的。SpringCloud将它集成在其子项目spring-cloud-netflix中，以实现SpringCloud的服务发现功能。
Eureka包含两个组件：Eureka Server和Eureka Client。
Eureka Server提供服务注册服务，各个节点启动后，会在Eureka Server中进行注册，这样EurekaServer中的服务注册表中将会存储所有可用服务节点的信息，服务节点的信息可以在界面中直观的看到。
Eureka Client是一个java客户端，用于简化与Eureka Server的交互，客户端同时也就是一个内置的、使用轮询(round-robin)负载算法的负载均衡器。
在应用启动后，将会向Eureka Server发送心跳,默认周期为30秒，如果Eureka Server在多个心跳周期内没有接收到某个节点的心跳，Eureka Server将会从服务注册表中把这个服务节点移除(默认90秒)。
Eureka Server之间通过复制的方式完成数据的同步，Eureka还提供了客户端缓存机制，即使所有的Eureka Server都挂掉，客户端依然可以利用缓存中的信息消费其他服务的API。综上，Eureka通过心跳检查、客户端缓存等机制，确保了系统的高可用性、灵活性和可伸缩性。

## 如何使用构建 Eureka Server ?
### 加入依赖（此处以Maven为例）
```xml
<!-- 1. 继承 spring-boot-starter-parent ，如果是聚合工程可以写到父工程中-->
<parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.4.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
</parent>
 <!-- 2.加入Eureka 服务端依赖 -->
<dependencies>
         <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>
</dependencies>
```
### 创建Eureka Server 主运行类
```java
package com.liang.cloud;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@EnableEurekaServer // 加上启用Eureka服务注解（标记其为Eureka服务）
@SpringBootApplication
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class,args);
    }
}

```
Eureka Server 会带有一个Web主页,默认访问地址：[http://localhost:8761/](http://localhost:8761/)。

Eureka服务 没有后台存储，但是注册表中的所有服务实例都必须发送心跳信号以使其注册保持最新（因此可以在内存中完成）。客户端还具有Eureka注册的内存缓存（因此，对于每个对服务的请求，它们都不必进入注册表）。

默认情况下，每个Eureka服务端也是有Eureka客户端，并且需要（至少一个）服务URL来定位。如果您不提供该服务，则该服务将不断运行，所输出的错误日志，也许对你有所干扰（如果你端口不是8761并且配置了另外的serviceUrl则会不断产生这样的错误日志，如果按默认配置只会报一次这样的错误，随后待自身启动后便可连接自身成功）。

### 单机配置
application.yml(单个Eureka服务配置），如下：
```yml
server:
  port: 8761 # 端口
spring:
  application:
    name: eureka-server # 应用名称，会在Eureka中显示
eureka:
  client:
    register-with-eureka: false # 是否注册自己的信息到EurekaServer，默认是true
    fetch-registry: false # 是否拉取其它服务的信息，默认是true
    service-url: # EurekaServer的地址，现在是自己的地址，如果是集群，需要加上其它Server的地址。
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka      
```
请注意==serviceUrl==,指向与本地实例相同的主机。

### 集群配置 
application.yml(两个Eureka服务配置），如下
```yml
---
server:
  port: 6001
spring:
  profiles: eureka6001
eureka:
  instance:
    hostname: eureka6001.com
  client:
    register-with-eureka: false # 是否注册自己的信息到EurekaServer，默认是true
    fetch-registry: false # 是否拉取其它服务的信息，默认是true
    service-url: # EurekaServer的地址，现在是自己的地址，如果是集群，需要加上其它Server的地址。
      defaultZone: http://eureka6002:6002/eureka


---
server:
  port: 6002
spring:
  profiles: eureka6002
eureka:
  instance:
    hostname: eureka6002.com
  client:
    register-with-eureka: false # 是否注册自己的信息到EurekaServer，默认是true
    fetch-registry: false # 是否拉取其它服务的信息，默认是true
    service-url: # EurekaServer的地址，现在是自己的地址，如果是集群，需要加上其它Server的地址。
      defaultZone: http://eureka6001:6001/eureka

```
在前面的示例中，我们有一个YAML文件，通过在不同的Spring配置文件中运行该服务器，可以在两个主机（==eureka6001==和==eureka6002==）上运行同一Eureka服务。您可以使用此配置通过操作/etc/hosts解析主机名来测试单个主机上的对等感知（在生产环境中这样做没有太大价值）。实际上，eureka.instance.hostname如果您在知道其主机名的计算机上运行（默认情况下，使用的是该机器的主机名）。

您可以将多个Eureka服务添加到集群，并且只要它们均通讯的连接，它们就可以在彼此之间同步注册。如果在物理上分开（在一个数据中心内或在多个数据中心之间），只要它们都直接相互连接，它们就可以在彼此之间同步注册。

### Eureka Client 连接Eureka Server 集群配置

application.yml(两个Eureka服务连接地址都需要加进来，英文逗号分隔），如下

```yml
eureka:
  client:
    serviceUrl:
      defaultZone: http://eureka6001.com/eureka/,http://eureka6002.com
```

# 示例源码下载：  
[https://github.com/mwanlyc/springcloud_study/tree/master/liang-cloud](https://github.com/mwanlyc/springcloud_study/tree/master/liang-cloud)