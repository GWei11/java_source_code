# 基本概念

* 流程定义
* 流程部署
* 流程实例





# 表结构

* 所有的 activiti 的表都是以 ACT_ 开头的
* 表的用途可以根据第二部分的两个字母可以看出来，用途也是和服务的 API 相对应的
  * **ACT_RE：**RE 表示 repository， 这个前缀的表包含了流程定义和流程静态资源（图片，规则等等）
  * **ACT_RU：**RU 表示 runtime，这些哦运行时的表，包含流程实例，任务，变量，异步任务等运行中的数据，activiti 只在流程实例执行过程中保存这些数据，当流程结束的时候就会删除这些记录，这样运行时表的数据就会一直很小。
  * **ACT_HI：**HI 表示 history，这些表包含历史数据，比如历史流程记录，变量，任务等等
  * **ACT_GE：**GE 表示 general，通用数据。

## 具体表介绍

| **表分类**   | **表名**              | **解释**                                           |
| ------------ | --------------------- | -------------------------------------------------- |
| 一般数据     |                       |                                                    |
|              | [ACT_GE_BYTEARRAY]    | 通用的流程定义和流程资源                           |
|              | [ACT_GE_PROPERTY]     | 系统相关属性                                       |
| 流程历史记录 |                       |                                                    |
|              | [ACT_HI_ACTINST]      | 历史的流程实例                                     |
|              | [ACT_HI_ATTACHMENT]   | 历史的流程附件                                     |
|              | [ACT_HI_COMMENT]      | 历史的说明性信息                                   |
|              | [ACT_HI_DETAIL]       | 历史的流程运行中的细节信息                         |
|              | [ACT_HI_IDENTITYLINK] | 历史的流程运行过程中用户关系                       |
|              | [ACT_HI_PROCINST]     | 历史的流程实例                                     |
|              | [ACT_HI_TASKINST]     | 历史的任务实例                                     |
|              | [ACT_HI_VARINST]      | 历史的流程运行中的变量信息                         |
| 流程定义表   |                       |                                                    |
|              | [ACT_RE_DEPLOYMENT]   | 部署单元信息                                       |
|              | [ACT_RE_MODEL]        | 模型信息                                           |
|              | [ACT_RE_PROCDEF]      | 已部署的流程定义                                   |
| 运行实例表   |                       |                                                    |
|              | [ACT_RU_EVENT_SUBSCR] | 运行时事件                                         |
|              | [ACT_RU_EXECUTION]    | 运行时流程执行实例                                 |
|              | [ACT_RU_IDENTITYLINK] | 运行时用户关系信息，存储任务节点与参与者的相关信息 |
|              | [ACT_RU_JOB]          | 运行时作业                                         |
|              | [ACT_RU_TASK]         | 运行时任务                                         |
|              | [ACT_RU_VARIABLE]     | 运行时变量表                                       |



# activit类关系图

![](https://gitee.com/GWei11/picture/raw/master/20210414213254.png)

## ProcessEngineConfiguration  接口

* ProcessEngineConfiguration 接口： 该类是一个配置类，主要存储的是 activiti.cfg.xml 文件解析之后的内容，当然也可以不使用这个默认的配置文件，该类提供了几个静态方法，来创建 ProcessEngineConfiguration对象。

```java
public static ProcessEngineConfiguration createProcessEngineConfigurationFromResourceDefault() {
        return createProcessEngineConfigurationFromResource("activiti.cfg.xml", "processEngineConfiguration");
    }

    public static ProcessEngineConfiguration createProcessEngineConfigurationFromResource(String resource) {
        return createProcessEngineConfigurationFromResource(resource, "processEngineConfiguration");
    }

    public static ProcessEngineConfiguration createProcessEngineConfigurationFromResource(String resource, String beanName) {
        return BeansConfigurationHelper.parseProcessEngineConfigurationFromResource(resource, beanName);
    }

    public static ProcessEngineConfiguration createProcessEngineConfigurationFromInputStream(InputStream inputStream) {
        return createProcessEngineConfigurationFromInputStream(inputStream, "processEngineConfiguration");
    }

    public static ProcessEngineConfiguration createProcessEngineConfigurationFromInputStream(InputStream inputStream, String beanName) {
        return BeansConfigurationHelper.parseProcessEngineConfigurationFromInputStream(inputStream, beanName);
    }

    public static ProcessEngineConfiguration createStandaloneProcessEngineConfiguration() {
        return new StandaloneProcessEngineConfiguration();
    }

    public static ProcessEngineConfiguration createStandaloneInMemProcessEngineConfiguration() {
        return new StandaloneInMemProcessEngineConfiguration();
    }
```

## ProcessEngine 接口

* ProcessEngine： 工作流引擎，该类就是一个门面接口，通过 ProcessEngineConfiguration 创建ProcessEngine 之后，就可以通过 ProcessEngine 来创建各个 Service 接口。如下面的代码所示：

第一种方式：直接使用默认的方式来创建

```java
//直接使用工具类 ProcessEngines，使用classpath下的activiti.cfg.xml中的配置创建processEngine
ProcessEngine processEngine = ProcessEngines.getDefaultProcessEngine();
System.out.println(processEngine);
```

​	这里表面上是使用 ProcessEngines 类来创建 ProcessEngine，但是继续看代码的话就可以看到实际上还是通过 ProcessEngineConfiguration 的 createProcessEngineConfigurationFromInputStream 方法来创建的 ProcessEngine 对象。

第二种方式：一般的创建方式，这里就可以不使用默认的配置文件了

```java
//先构建ProcessEngineConfiguration
ProcessEngineConfiguration configuration = ProcessEngineConfiguration.createProcessEngineConfigurationFromResource("activiti.cfg.xml");
//通过ProcessEngineConfiguration创建ProcessEngine，此时会创建数据库
ProcessEngine processEngine = configuration.buildProcessEngine();
```



## RepositoryService 接口

* RepositoryService 接口是 activiti 的资源管理类，提供了管理和控制流程发布包和流程定义的操作，我们使用工作流建模工具设计的业务流程图需要使用这个 service 将流程定义文件的内容部署到计算机。

部署流程示例代码：

```java
//1、创建ProcessEngine
ProcessEngine processEngine = ProcessEngines.getDefaultProcessEngine();
//2、获取RepositoryServcie
RepositoryService repositoryService = processEngine.getRepositoryService();
//3、使用service进行流程的部署，定义一个流程的名字，把bpmn和png部署到数据中
Deployment deploy = repositoryService.createDeployment()
    .name("流程名称")
    .addClasspathResource("bpmn格式的流程图")
    .addClasspathResource("图片格式的流程图") // 这个资源实际上是bpmn格式导出的，主要是为了便于查看流程
    .deploy();
```



## RuntimeService 接口

* 流程运行管理类，比如流程的启动就是该接口进行操作的



## TaskService 接口

* 任务管理类，比如你现在需要请假，那创建一个请假单就是一个任务，你把创建好的请假单发送给领导审批，对于领导来说，审批这个动作也是一个任务，所以任务的审批就是在 TaskService 接口来实现的。



## HistoryService 接口

* 历史管理类， 可以查询历史信息，执行流程时，引擎会保存很多数据（根据配置），比如流程实例启动时间，任务的参与者， 完成任务的时间，每个流程实例的执行路径，等等。 这个服务主要通过查询功能来获得这些数据。

## ManagementService 接口

* 这个接口主要是用于 activiti 系统的日常维护

