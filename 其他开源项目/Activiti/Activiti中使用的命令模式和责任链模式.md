

# TaskQueryImpl 类说明



Activiti 中主要是有几个 Service 接口来操作对应的表，下面我们以 TaskQueryImpl 这个类来举例说明。

## TaskQueryImpl 类的继承体系

![](https://gitee.com/GWei11/picture/raw/master/20210419221224.png)

可以看到该类实现了 Command 接口

```java
package org.activiti.engine.impl.interceptor;

public interface Command<T> {
    T execute(CommandContext var1);
}
```

所以 TaskQueryImpl 就是 Command 接口的一个具体的实现类，在命令模式中有以下几个角色

* Command 命令接口
* Command 命令接口的具体实现类
* Invoker （请求命令）
* Receiver 命令接收端，该角色不是必须

对应到到上面的例子中 Command 命令接口和该命令接口的实现类 TaskQueryImpl 都已经有了，而 Invoker 角色有一个 CommandInvoker

```java
public class CommandInvoker extends AbstractCommandInterceptor {
    private static final Logger logger = LoggerFactory.getLogger(CommandInvoker.class);

    public CommandInvoker() {
    }

    public <T> T execute(CommandConfig config, final Command<T> command) {
        final CommandContext commandContext = Context.getCommandContext();
        commandContext.getAgenda().planOperation(new Runnable() {
            public void run() {
                commandContext.setResult(command.execute(commandContext));
            }
        });
        this.executeOperations(commandContext);
        if (commandContext.hasInvolvedExecutions()) {
            Context.getAgenda().planExecuteInactiveBehaviorsOperation();
            this.executeOperations(commandContext);
        }

        return commandContext.getResult();
    }
    .........
}
```

当然，Invoker 实际上不仅仅是这一个类，而是一系列的类组成的，这里的 Invoker 角色使用的是责任链模式。可以看到 CommandInvoker 类实现了 CommandInterceptor 接口。

![](https://gitee.com/GWei11/picture/raw/master/20210419221917.png)



看一个具体的代码示例：

```java
@Test
public void testTaskQuery () {
    ProcessEngine engine = ProcessEngines.getDefaultProcessEngine();
    TaskQuery taskQuery = engine.getTaskService().createTaskQuery();
    List<Task> tasks = taskQuery.processInstanceId("流程实例id").list();
}
```

继续看  list() 方法的代码

`AbstractQuery` 类

```java
public List<U> list() {
        this.resultType = AbstractQuery.ResultType.LIST;
        return this.commandExecutor != null ? (List)this.commandExecutor.execute(this) : this.executeList(Context.getCommandContext(), (Page)null);
    }
```

这里使用 CommandExecutor 接口里面的方法，我们看一下该接口的唯一实现 CommandExecutorImpl

```java
public class CommandExecutorImpl implements CommandExecutor {
    protected CommandConfig defaultConfig;
    protected CommandInterceptor first;

    public CommandExecutorImpl(CommandConfig defaultConfig, CommandInterceptor first) {
        this.defaultConfig = defaultConfig;
        this.first = first;
    }

    public CommandInterceptor getFirst() {
        return this.first;
    }

    public void setFirst(CommandInterceptor commandInterceptor) {
        this.first = commandInterceptor;
    }

    public CommandConfig getDefaultConfig() {
        return this.defaultConfig;
    }

    public <T> T execute(Command<T> command) {
        return this.execute(this.defaultConfig, command);
    }

    public <T> T execute(CommandConfig config, Command<T> command) {
        return this.first.execute(config, command);
    }
}
```

该类里面就有 CommandInterceptor ，刚上面已经说了，CommandInvoker 其实就是一个 CommandInterceptor，因为 CommandInvoker 实现了 CommandInterceptor接口，所以 CommandInvoker 是这个责任链中的一环。

那这些责任链中的所有的类以及类之间的关系是在哪里初始化的呢？

其实是在 ProcessEngineConfigurationImpl 这个类里面的 initCommandExecutors() 方法里面进行初始化的。

`ProcessEngineConfigurationImpl`  类

```java
public void initCommandExecutors() {
    this.initDefaultCommandConfig();
    this.initSchemaCommandConfig();
    this.initCommandInvoker();  // 初始化 CommandInvoker
    this.initCommandInterceptors(); // 初始化 CommandInterceptors
    this.initCommandExecutor(); // 初始化 CommandExecutor
}
```

先看 initCommandInvoker 方法

```java
public void initCommandInvoker() {
    if (this.commandInvoker == null) {
        if (this.enableVerboseExecutionTreeLogging) {
            this.commandInvoker = new DebugCommandInvoker();
        } else {
            this.commandInvoker = new CommandInvoker();
        }
    }
}
```

可以看到，最开始的时候，给 this.commandInvoker 属性赋值的对象就是 CommandInvoker

再看 initCommandInterceptors 方法

```java
public void initCommandInterceptors() {
    if (this.commandInterceptors == null) {
        this.commandInterceptors = new ArrayList();
        if (this.customPreCommandInterceptors != null) {
            this.commandInterceptors.addAll(this.customPreCommandInterceptors); // 添加自定义的拦截器
        }
        this.commandInterceptors.addAll(this.getDefaultCommandInterceptors()); // 添加默认的拦截器
        if (this.customPostCommandInterceptors != null) {
            this.commandInterceptors.addAll(this.customPostCommandInterceptors);
        }
        this.commandInterceptors.add(this.commandInvoker); // this.commandInvoker就是 在 this.initCommandInvoker方法里面初始化的，值就是CommandInvoker对象
    }
}
```

* 上面代码中最开始是添加自定义的拦截器，如果自己没有添加，那么就是没有
* 然后就是添加默认的拦截器，进去看一下添加默认的拦截器的方法

this.getDefaultCommandInterceptors() 方法的代码

```java
public Collection<? extends CommandInterceptor> getDefaultCommandInterceptors() {
    List<CommandInterceptor> interceptors = new ArrayList();
    interceptors.add(new LogInterceptor()); // 第一个就是添加了一个日志拦截器
    CommandInterceptor transactionInterceptor = this.createTransactionInterceptor(); // 事务拦截器
    if (transactionInterceptor != null) {
        interceptors.add(transactionInterceptor);
    }

    if (this.commandContextFactory != null) {
        interceptors.add(new CommandContextInterceptor(this.commandContextFactory, this));
    }

    if (this.transactionContextFactory != null) {
        interceptors.add(new TransactionContextInterceptor(this.transactionContextFactory));
    }

    return interceptors;
}
```



现在再回到 最开始的 this.initCommandExecutor() 方法

```java
public void initCommandExecutor() {
    if (this.commandExecutor == null) {
        CommandInterceptor first = this.initInterceptorChain(this.commandInterceptors);
        this.commandExecutor = new CommandExecutorImpl(this.getDefaultCommandConfig(), first);
    }
}
```

可以看到，this.commandExecutor 属性就是使用 CommandExecutorImpl 这个类初始化的，同时还传入了一个 CommandInterceptor【也就是 first 参数】，其实如果我们没有添加自定义的 拦截器的话，这里的 first 就是 LogInterceptor，而且从 initCommandInterceptors() 方法中也不难看出，其实 CommandInvoker 就是这个责任链条上面的最后一个类

```java
public void initCommandInterceptors() {
    if (this.commandInterceptors == null) {
        this.commandInterceptors = new ArrayList();
        if (this.customPreCommandInterceptors != null) {
            this.commandInterceptors.addAll(this.customPreCommandInterceptors); // 添加自定义的拦截器
        }
        this.commandInterceptors.addAll(this.getDefaultCommandInterceptors()); // 添加默认的拦截器
        if (this.customPostCommandInterceptors != null) {
            this.commandInterceptors.addAll(this.customPostCommandInterceptors);
        }
        this.commandInterceptors.add(this.commandInvoker); // commandInvoker 是责任链上的最后一环。
    }
}
```

所以在 CommandInvoker 中的 setNext(CommandInterceptor next) 方法会抛出异常。

CommandInvoker  类。

```java
public void setNext(CommandInterceptor next) {
    throw new UnsupportedOperationException("CommandInvoker must be the last interceptor in the chain");
}
```

