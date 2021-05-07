# Spring Cloud 核心组件

|                | 第一代 Spring Cloud        | 第二代Spring Cloud（Spring Cloud Alibaba） |
| -------------- | -------------------------- | ------------------------------------------ |
| 注册中心       | Netflix Eureka（闭源了）   | Nacos                                      |
| 客户端负载均衡 | Netflix Ribbon             | Dubbo LB、Spring Cloud Loadbalancer        |
| 熔断器         | Netflix Hystrix            | Sentinel                                   |
| 网关           | Netflix Zuul（不再使用）   | Spring Cloud Gateway                       |
| 配置中心       | Spring Cloud Config        | Nacos、携程 Apollo                         |
| 服务调用       | Netflix Feign              | Dubbo RPC                                  |
| 消息驱动       | Spring Cloud Stream        |                                            |
| 链路追踪       | Spring Cloud Sleuth/Zipkin |                                            |
|                |                            | 阿里巴巴 seata 分布式事务方案              |

# 主流服务注册组件的对比

| 组件名    | 开发语言 | CAP           | 对外暴露接口 |
| --------- | -------- | ------------- | ------------ |
| Eureka    | Java     | AP            | HTTP         |
| Consul    | Go       | CP            | HTTP/DNS     |
| Zookeeper | Java     | CP            | 客户端       |
| Nacos     | Java     | 支持AP/CP切换 | HTTP         |

* P: 分区容错性
* C: 数据一致性
* A: 高可用

# 搭建 Eureka 集群

## 项目依赖 

可以引入一个父工程用来规范 Maven 依赖，两个 Eureka 服务作为子项目

```java
parent-project
	eureka-server-one
    eureka-server-two
```

在 parent-project 项目的 pom.xml 中添加如下依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>parent-project</artifactId>
    <version>1.0-SNAPSHOT</version>
    <modules>
        <module>eureka-server-one</module>
        <module>eureka-server-two</module>
    </modules>
    <!--父工程打包方式为pom-->
    <packaging>pom</packaging>
    <!--spring boot 父启动器依赖-->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.6.RELEASE</version>
    </parent>
    <dependencyManagement>
        <dependencies>
            <!--spring cloud依赖管理，引入了Spring Cloud的版本-->
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Greenwich.RELEASE</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <!--web依赖-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <!--日志依赖-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </dependency>
        <!--测试依赖-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <!--lombok工具-->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.4</version>
            <scope>provided</scope>
        </dependency>
        <!-- Actuator可以帮助你监控和管理Spring Boot应用-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <!--热部署-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <optional>true</optional>
        </dependency>

        <!--eureka server 需要引入Jaxb，开始-->
        <dependency>
            <groupId>com.sun.xml.bind</groupId>
            <artifactId>jaxb-core</artifactId>
            <version>2.2.11</version>
        </dependency>
        <dependency>
            <groupId>javax.xml.bind</groupId>
            <artifactId>jaxb-api</artifactId>
        </dependency>
        <dependency>
            <groupId>com.sun.xml.bind</groupId>
            <artifactId>jaxb-impl</artifactId>
            <version>2.2.11</version>
        </dependency>
        <dependency>
            <groupId>org.glassfish.jaxb</groupId>
            <artifactId>jaxb-runtime</artifactId>
            <version>2.2.10-b140310.1920</version>
        </dependency>
        <dependency>
            <groupId>javax.activation</groupId>
            <artifactId>activation</artifactId>
            <version>1.1.1</version>
        </dependency>
    </dependencies>
</project>
```

然后在 子工程的 pom.xml 中添加 Eureka 的依赖即可。 两个子工程的依赖都是如下代码所示

```xml
<parent>
    <artifactId>parent-project</artifactId>
    <groupId>com.example</groupId>
    <version>1.0-SNAPSHOT</version>
</parent>
<dependencies>
    <!--Eureka server依赖, 这里不用添加版本号是因为在父工程中已经添加了 Spring cloud 的版本号-->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
</dependencies>
```



## 配置文件

maven 依赖已经添加完成，接下来就是添加配置文件。Eureka 集群这里使用的是两个服务 ，所以 eureka-server-one 和 eureka-server-two 两个服务都是需要配置文件的。

**【注意】因为是在同一台机器上模拟集群，也就是说 ip 是相同的，所以可以在 host 文件中配置 dns 映射**

window 系统下在 host 文件中添加如下内容

```java
127.0.0.1 eurekaServer1
127.0.0.1 eurekaServer2
```

### eureka-server-one 的配置文件

```yaml
#eureka server服务端口
server:
  port: 8761
spring:
  application:
    name: eureka-server # 应用名称，应用名称会在Eureka中作为服务名称，集群中多个实例之间应用名称需要保持一致

# eureka 客户端配置, 比如集群中有两台机器 ，当前机器对于另外一台机器来说就是一个客户端（集群中当前实例对于其他实例来说就是一个客户端）
eureka:
  instance:
    hostname: eurekaServer1  # 当前eureka实例的主机名，eurekaServer1就是在host 文件中配置的
  client:
    service-url:
      # 配置客户端所交互的Eureka Server的地址（Eureka Server集群中每一个Server其实相对于其它Server来说都是Client）
      # 集群模式下，defaultZone应该指向其它Eureka Server，如果有更多其它Server实例，逗号拼接即可
      defaultZone: http://eurekaServer2:8762/eureka
    register-with-eureka: true  # 集群模式下可以改成true，表示将当前实例注册成一个 eureka 服务，如果是单机模式下就不用注册，因为自身就是服务
    fetch-registry: true
```

###  eureka-server-two 的配置文件

```yaml
#eureka server服务端口
server:
  port: 8762
spring:
  application:
    name: eureka-server # 应用名称，应用名称会在Eureka中作为服务名称，集群中多个实例之间应用名称需要保持一致

# eureka 客户端配置, 比如集群中有两台机器 ，当前机器对于另外一台机器来说就是一个客户端（集群中当前实例对于其他实例来说就是一个客户端）
eureka:
  instance:
    hostname: eurekaServer2  # 当前eureka实例的主机名，eurekaServer2就是在host 文件中配置的
  client:
    service-url: # 配置客户端所交互的Eureka Server的地址
      defaultZone: http://eurekaServer1:8761/eureka
    register-with-eureka: true # 集群模式下可以改成true，表示将当前实例注册成一个 eureka 服务，如果是单机模式下就不用注册，因为自身就是服务
    fetch-registry: true
```

## 启动类

【注意】在启动类中需要加上 @**EnableEurekaServer** 注解

### eureka-server-one 项目的启动类

```java
package com.example;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
// 声明当前项目为Eureka服务
@EnableEurekaServer
public class EurekaServerApp8761 {

    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApp8761.class,args);
    }
}
```

### euraka-server-two项目的启动类

```java
package com.lexample;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
// 声明当前项目为Eureka服务
@EnableEurekaServer
public class EurekaServerApp8762 {

    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApp8762.class,args);
    }
}
```

## 访问项目

启动两个项目后，可以在浏览器中午访问 Eureka 集群。

```http
http://eurekaserver1:8761/
http://eurekaserver2:8762/
```

访问上面的两个地址都可以访问 Eureka 集群， 访问 8761 的服务界面如下所示：

![](https://gitee.com/GWei11/picture/raw/master/20210506214656.png)

访问 8762 服务的界面如下：

![](https://gitee.com/GWei11/picture/raw/master/20210506214847.png)



# 服务消费者

## 添加 Eureka 客户端依赖

这里只是列举了一个 服务提供者，如果想要启动多个服务提供者，也是一样的，每一个服务提供者都需要引入 Eureka 客户端的依赖。

```xml
<!--eureka client 客户端依赖引入-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

## 编写配置文件

**application.yml**

这里只是列举了一个 服务提供者配置文件，如果想要启动多个服务提供者，**server.port 是需要修改的**，其余的内容可以不用修改。

```yaml
server:
  port: 8091
#注册到Eureka服务中心
eureka:
  client:
    service-url:
      # 注册到集群，就把多个Eurekaserver地址使用逗号连接起来即可；注册到单实例（非集群模式），那就写一个就ok
      defaultZone: http://eurekaServer1:8761/eureka,http://eurekaServer2:8762/eureka
spring:
  application:
    name: service-producer
```

## 编写启动类

```java
package com.example;


import com.netflix.hystrix.contrib.metrics.eventstream.HystrixMetricsStreamServlet;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.web.servlet.ServletRegistrationBean;
import org.springframework.cloud.client.circuitbreaker.EnableCircuitBreaker;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestTemplate;

@SpringBootApplication
@EnableDiscoveryClient
public class ConsumerApplication8091 {

    public static void main(String[] args) {
        SpringApplication.run(ConsumerApplication8091.class,args);
    }

    // 使用RestTemplate模板对象进行远程调用
    @Bean
    @LoadBalanced
    public RestTemplate getRestTemplate() {
        return new RestTemplate();
    }
}
```

## Controller 层

```java
package com.example.controller;

import com.netflix.hystrix.contrib.javanica.annotation.HystrixCommand;
import com.netflix.hystrix.contrib.javanica.annotation.HystrixProperty;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.client.discovery.DiscoveryClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;

import java.util.List;

@RestController
@RequestMapping("/consumer")
public class ConsumerController {
    @Autowired
    private RestTemplate restTemplate;

    @Autowired
    private DiscoveryClient discoveryClient;

    /**
     * 服务注册到Eureka之后的改造
     * @param userId
     * @return
     */
    @GetMapping("/test")
    public void testConsumer() {
        // 从Eureka Server中获取我们关注的那个服务的实例信息以及接口信息
        List<ServiceInstance> instances = discoveryClient.getInstances("provider-server");
        // 2、如果有多个实例，选择一个使用(负载均衡的过程)
        ServiceInstance serviceInstance = instances.get(0);
        // 3、从元数据信息获取host port
        String host = serviceInstance.getHost();
        int port = serviceInstance.getPort();
        // 这个路径是服务提供者端 Controller 中方法的路径
        String url = "http://" + host + ":" + port + "/provider/testProvider/";
        // httpclient封装好多内容进行远程调用
        Integer forObject = restTemplate.getForObject(url, Integer.class);
        return forObject;
    }
}

```

# 服务提供者

## 添加 Eureka 客户端依赖

```xml
<!--eureka client 客户端依赖引入-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<!--Config 客户端依赖-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-client</artifactId>
</dependency>
```

## 编写配置文件

```yaml
eureka:
  client:
    service-url:
      # 注册到集群，就把多个Eurekaserver地址使用逗号连接起来即可；注册到单实例（非集群模式），那就写一个就ok
      defaultZone: http://eurekaServer1:8761/eureka,http://eurekaServer2:8762/eureka
```

## 编写启动类

```java
package com.example;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.autoconfigure.domain.EntityScan;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.context.annotation.Bean;
import org.springframework.context.support.PropertySourcesPlaceholderConfigurer;

@SpringBootApplication
@EntityScan("com.lagou.edu.pojo")
//@EnableEurekaClient  // 开启Eureka Client（Eureka独有）
@EnableDiscoveryClient // 开启注册中心客户端 （通用型注解，比如注册到Eureka、Nacos等）
                       // 说明：从SpringCloud的Edgware版本开始，不加注解也可以
public class ProducerApplication8080 {
    public static void main(String[] args) {
        SpringApplication.run(ProducerApplication8080.class,args);
    }
}
```

## Controller 层

```java
package com.lagou.edu.controller;

import com.lagou.edu.service.ResumeService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/provider")
public class ResumeController {

    @GetMapping("/testProvider")
    public Integer testProvider() {
        System.out.println("这里是服务提供者");
        return 1;
    }
}
```

