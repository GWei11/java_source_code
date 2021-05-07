# 依赖 jar 包

eureka 服务端是需要如下依赖

```xml
<dependencies>
    <!--Eureka server依赖-->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
</dependencies>
```

所以源头还是在这个 jar 中，该 jar 包下有一个文件， `META-INF\spring.factories` 文件内容如下：

```java
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  org.springframework.cloud.netflix.eureka.server.EurekaServerAutoConfiguration
```

这里是充分利用了Spring Boot 的自动装配功能，所以接下来就是看 EurekaServerAutoConfiguration 这个配置类的内容

# EurekaServerAutoConfiguration 配置类

```java
@Configuration
@Import({EurekaServerInitializerConfiguration.class})
@ConditionalOnBean({Marker.class})
@EnableConfigurationProperties({EurekaDashboardProperties.class, InstanceRegistryProperties.class})
@PropertySource({"classpath:/eureka/server.properties"})
public class EurekaServerAutoConfiguration extends WebMvcConfigurerAdapter {
	……
}
```

对于类里面的内容暂且先放一下，先看一下这个配置类的一些注解

![](https://gitee.com/GWei11/picture/raw/master/20210507063448.png)

## @ConditionalOnBean({Marker.class})

看 @ConditionalOnBean 这个注解知道，如果想要让当前这个配置类生效 ，那么在 ioc 容器中就需要有 Marker 这个类，这个类是哪里来的呢？

Eureka Server 其实也是一个 Spring Boot 项目，只是项目中添加了上述 的jar 包，所以 这个项目也会有 启动类，如下面代码所示：

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

* 除了 SpringBootApplication 注解之外还有一个注解 EnableEurekaServer

![](https://gitee.com/GWei11/picture/raw/master/20210507064321.png)



## EurekaServerAutoConfiguration 类自身内容

EurekaServerAutoConfiguration 这个类既然是一个配置类，那么肯定会在这个类里面创建一些 bean，现在来重点看一下该类创建的 bean 

### EurekaController

```java
@Bean
@ConditionalOnProperty(
    prefix = "eureka.dashboard",
    name = {"enabled"},
    matchIfMissing = true
)
public EurekaController eurekaController() {
    return new EurekaController(this.applicationInfoManager);
}
```

* 当启动 Eureka 服务端项目之后，我们访问 http://localhost:8761 能看到的界面就是因为在这里 创建了这个 Controller
* 可以通过配置项 eureka.dashboard.enabled=true 来控制显示（默认为true），如果不想要这个功能，可以给这个 key 设置为 false

### PeerAwareInstanceRegistry

```java
@Bean
public PeerAwareInstanceRegistry peerAwareInstanceRegistry(ServerCodecs serverCodecs) {
    this.eurekaClient.getApplications();
    return new InstanceRegistry(this.eurekaServerConfig, this.eurekaClientConfig, serverCodecs, this.eurekaClient, this.instanceRegistryProperties.getExpectedNumberOfClientsSendingRenews(), this.instanceRegistryProperties.getDefaultOpenForTrafficCount());
}
```

* 这个 bean 是在集群环境中使用，在集群环境中，当前节点相对其余节点来说就是一个客户端，需要将自己注册到其他服务（获取其他节点的数据），此时就需要这个类进行注册
* EurekaServer 集群中各个节点都是对等的，没有主从之分

### PeerEurekaNodes

```java
@Bean
@ConditionalOnMissingBean
public PeerEurekaNodes peerEurekaNodes(PeerAwareInstanceRegistry registry, ServerCodecs serverCodecs) {
    return new EurekaServerAutoConfiguration.RefreshablePeerEurekaNodes(registry, this.eurekaServerConfig, this.eurekaClientConfig, serverCodecs, this.applicationInfoManager);
}
```

* 上面的 PeerAwareInstanceRegistry 是用来注册节点，PeerEurekaNodes 则是在注册之后用来操作节点的，比如更新节点信息

* 在 PeerEurekaNodes 类中有一个 start 方法，在这个方法里面来更新节点信息

```java
public void start() {
    // 这里创建了一个线程池（在哪里启动的呢？后面的解析中会体现出来， 在后面的 EurekaServerContext 类中调用）
    this.taskExecutor = Executors.newSingleThreadScheduledExecutor(new ThreadFactory() {
        public Thread newThread(Runnable r) {
            Thread thread = new Thread(r, "Eureka-PeerNodesUpdater");
            thread.setDaemon(true);
            return thread;
        }
    });

    try {
        // 重点是这个代码，更新节点信息
        this.updatePeerEurekaNodes(this.resolvePeerUrls());
        Runnable peersUpdateTask = new Runnable() {
            public void run() {
                try {
                    // 重点是这个代码，更新节点信息
                    PeerEurekaNodes.this.updatePeerEurekaNodes(PeerEurekaNodes.this.resolvePeerUrls());
                } catch (Throwable var2) {
                    PeerEurekaNodes.logger.error("Cannot update the replica Nodes", var2);
                }

            }
        };
        this.taskExecutor.scheduleWithFixedDelay(peersUpdateTask, (long)this.serverConfig.getPeerEurekaNodesUpdateIntervalMs(), (long)this.serverConfig.getPeerEurekaNodesUpdateIntervalMs(), TimeUnit.MILLISECONDS);
    } catch (Exception var3) {
        throw new IllegalStateException(var3);
    }

    Iterator var4 = this.peerEurekaNodes.iterator();

    while(var4.hasNext()) {
        PeerEurekaNode node = (PeerEurekaNode)var4.next();
        logger.info("Replica node URL:  {}", node.getServiceUrl());
    }
}
```

### EurekaServerContext

```java
@Bean
public EurekaServerContext eurekaServerContext(ServerCodecs serverCodecs, PeerAwareInstanceRegistry registry, PeerEurekaNodes peerEurekaNodes) {
    return new DefaultEurekaServerContext(this.eurekaServerConfig, serverCodecs, registry, peerEurekaNodes, this.applicationInfoManager);
}
```

* 这个类是用来构建 EurekaServerContext ，默认实现是 DefaultEurekaServerContext 

#### DefaultEurekaServerContext 

在 DefaultEurekaServerContext  类中有一个 initialize 方法，该方法添加了 @PostConstruct 注解，所以是在该类构造方法执行之后执行

```java
@PostConstruct
public void initialize() {
    logger.info("Initializing ...");
    // 在前面的 PeerEurekaNodes 中已经看了start 方法，就是在这里进行调用的
    this.peerEurekaNodes.start();

    try {
        this.registry.init(this.peerEurekaNodes);
    } catch (Exception var2) {
        throw new RuntimeException(var2);
    }

    logger.info("Initialized");
}
```

### EurekaServerBootstrap

```java
@Bean
public EurekaServerBootstrap eurekaServerBootstrap(PeerAwareInstanceRegistry registry, EurekaServerContext serverContext) {
    return new EurekaServerBootstrap(this.applicationInfoManager, this.eurekaClientConfig, this.eurekaServerConfig, registry, serverContext);
}
```

* 注册的 EurekaServerBootstrap 类，在后面的启动过程中会使用到这个类。

## @Import(EurekaServerInitializerConfiguration.class)

@Import 注解引入了一个类，所以现在重点看一下这个类里面的内容

### EurekaServerInitializerConfiguration

```java
public class EurekaServerInitializerConfiguration implements ServletContextAware, SmartLifecycle, Ordered {
	……
}
```

* 该类实现了 SmartLifecycle 接口， 而 SmartLifecycle 接口继承了 LifeCycle 接口， 下面是 LifeCycle 接口中的方法

```java
public interface Lifecycle {
    void start();
    void stop();
    boolean isRunning();
}
```

* start 方法是在 Spring 容器的 Bean 创建完成之后执行的

* 看一下 EurekaServerInitializerConfiguration 类中的 start 方法

```java
public void start() {
    (new Thread(new Runnable() {
        public void run() {
            try {
                // 在这这里面初始化 EurekaServerContext，重点看这里面
                EurekaServerInitializerConfiguration.this.eurekaServerBootstrap.contextInitialized(EurekaServerInitializerConfiguration.this.servletContext);
                EurekaServerInitializerConfiguration.log.info("Started Eureka Server");
                // 发布事件
                EurekaServerInitializerConfiguration.this.publish(new EurekaRegistryAvailableEvent(EurekaServerInitializerConfiguration.this.getEurekaServerConfig()));
                EurekaServerInitializerConfiguration.this.running = true;
                EurekaServerInitializerConfiguration.this.publish(new EurekaServerStartedEvent(EurekaServerInitializerConfiguration.this.getEurekaServerConfig()));
            } catch (Exception var2) {
                EurekaServerInitializerConfiguration.log.error("Could not initialize Eureka servlet context", var2);
            }

        }
    })).start();
}
```

* 重点是调用了 EurekaServerBootstrap 里面 的 contextInitialized 方法
* EurekaServerBootstrap 已经在EurekaServerAutoConfiguration 中进行了初始化

#### EurekaServerBootstrap中contextInitialized 方法

* org.springframework.cloud.netflix.eureka.server.EurekaServerBootstrap#contextInitialized

```java
public void contextInitialized(ServletContext context) {
    try {
        // 初始化环境信息
        this.initEurekaEnvironment();
        // 初始化 context 细节，重点看这里
        this.initEurekaServerContext();
        context.setAttribute(EurekaServerContext.class.getName(), this.serverContext);
    } catch (Throwable var3) {
        log.error("Cannot bootstrap eureka server :", var3);
        throw new RuntimeException("Cannot bootstrap eureka server :", var3);
    }
}
```

* org.springframework.cloud.netflix.eureka.server.EurekaServerBootstrap#initEurekaServerContext

```java
protected void initEurekaServerContext() throws Exception {
    JsonXStream.getInstance().registerConverter(new V1AwareInstanceInfoConverter(), 10000);
    XmlXStream.getInstance().registerConverter(new V1AwareInstanceInfoConverter(), 10000);
    if (this.isAws(this.applicationInfoManager.getInfo())) {
        this.awsBinder = new AwsBinderDelegate(this.eurekaServerConfig, this.eurekaClientConfig, this.registry, this.applicationInfoManager);
        this.awsBinder.start();
    }
	// 为非ioc容器提供获取serverContext 对象的接口
    EurekaServerContextHolder.initialize(this.serverContext);
    log.info("Initialized server context");
    // 下面两行代码是重点
    // 某一个server 实例启动的时候，从集群中其他的server 拷贝注册信息过来（同步），每一个server相对于其他server来说就是一个客户端
    int registryCount = this.registry.syncUp();
    // 更改实例状态为up，对外提供服务
    this.registry.openForTraffic(this.applicationInfoManager, registryCount);
    EurekaMonitors.registerAllStats();
}
```

* com.netflix.eureka.registry.PeerAwareInstanceRegistryImpl#syncUp 

```java
public int syncUp() {
    int count = 0;
    // getRegistrySyncRetries 方法表示如果没有连接上就要重试
    for(int i = 0; i < this.serverConfig.getRegistrySyncRetries() && count == 0; ++i) {
        if (i > 0) {
            try {
                Thread.sleep(this.serverConfig.getRegistrySyncRetryWaitMs());
            } catch (InterruptedException var10) {
                logger.warn("Interrupted during registry transfer..");
                break;
            }
        }
		// 获取到其他server 的注册表信息
        Applications apps = this.eurekaClient.getApplications();
        Iterator var4 = apps.getRegisteredApplications().iterator();
        while(var4.hasNext()) {
            Application app = (Application)var4.next();
            Iterator var6 = app.getInstances().iterator();
            while(var6.hasNext()) {
                InstanceInfo instance = (InstanceInfo)var6.next();
                try {
                    if (this.isRegisterable(instance)) {
                        // 把从远程获取过来的注册信息注册到自己的注册表中（这里的注册表实际上就是一个map）
                        this.register(instance, instance.getLeaseInfo().getDurationInSecs(), true);
                        ++count;
                    }
                } catch (Throwable var9) {
                    logger.error("During DS init copy", var9);
                }
            }
        }
    }
    return count;
}
```

* 上面 22行说到 注册表实际上是一个 map， 可以到 register 方法中去查看

```java
public abstract class AbstractInstanceRegistry implements InstanceRegistry {
    ……
    // 注册信息存储在这个map中
    private final ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>> registry = new ConcurrentHashMap();
	……
}
```

