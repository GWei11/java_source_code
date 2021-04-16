# 基础版案例（请假申请流程）

## 使用java程序创建表

```java
@Test
public void testCreateDbTable(){
    //        需要使用avtiviti提供的工具类 ProcessEngines ,使用方法getDefaultProcessEngine
    //        getDefaultProcessEngine会默认从resources下读取名字为actviti.cfg.xml的文件
    //        创建processEngine时，就会创建mysql的表

    //        第一种方式，使用默认方式
    //        ProcessEngine processEngine = ProcessEngines.getDefaultProcessEngine();

    //        第二种创建表的方式，使用自定义方式
    //        配置文件的名字可以自定义,bean的名字也可以自定义
    ProcessEngineConfiguration processEngineConfiguration = ProcessEngineConfiguration.
        createProcessEngineConfigurationFromResource("activiti.cfg.xml",
                                                     "processEngineConfiguration");
    //        获取流程引擎对象
    ProcessEngine processEngine = processEngineConfiguration.buildProcessEngine();
}
```



## 需求讲解

![](https://gitee.com/GWei11/picture/raw/master/20210415064935.png)

如图所示，现在出差申请的流程由**申请人创建单据->经理审批->总经理审批->财务审批**这几步，每一个步骤对于 activiti 来说其实就是一个 task ，每一个 task 的处理人在当前这个案例中先固定（实际操作中是动态的）。

## 画流程图（bpmn图）

* 先在 idea 中 plugins 里面搜索 actiBPM 插件进行安装
* 在 resources 目录下 创建 bpmn 目录， 然后在bpmn 目录里面创建 evection.bpmn 文件，然后打开，按照下图所示的结构来绘制
  * 比如要画开始节点，就用把右侧的 StartEvent 拖到中间去就好， 同样的把下面的几个 UserTask 和 EndEvent 都拖进来。
  * 如何将这几个节点连起来呢？将鼠标点击一个节点，比如拖进来的 StartEvent，将鼠标移到这个节点的正中间的位置，鼠标的形状会改变，此时就可以从这个节点拖线出来去连接其他的节点了。
  * 最左侧的是节点属性设置，我们先点击某一个节点，比如图中就是点击了第一个用户节点，然后就会出现最左侧的面板，需要填写的就是 name 属性和 Assignee 属性。

![](https://gitee.com/GWei11/picture/raw/master/20210415065715.png)

* 画好了流程图之后，可以先将 evection.bpmn 文件修改成 evection.xml 文件（注意：这里的修改是为了导出图片，后续还是要将 evection.xml 修改回 evection.bpmn）

![](https://gitee.com/GWei11/picture/raw/master/20210415070602.png)

按照上图这种方式操作打开之后可以看到一张图片，然后导出为 png格式，命名为evection.png

![image-20210415070726941](https://gitee.com/GWei11/picture/raw/master/20210415070727.png)

**【注意】：导出为图片格式之后，需要将 evection.xml 文件名修改成 evection.bpmn**



查询画好的 bpmn 文件（文件名evection.bpmn）后可以使用文本文件查看软件打开看一下，实际上 bpmn 文件就是一个 xml 格式的文件，我们找到 <process></process> 标签部分内容看一下

```xml
<process id="myEvection" isClosed="false" isExecutable="true" name="出差申请" processType="None">
    <startEvent id="_2" name="StartEvent"/>
    <userTask activiti:assignee="zhangsan" activiti:exclusive="true" id="_3" name="创建出差申请"/>
    <userTask activiti:assignee="jerry" activiti:exclusive="true" id="_4" name="经理审批"/>
    <userTask activiti:assignee="jack" activiti:exclusive="true" id="_5" name="总经理审批"/>
    <userTask activiti:assignee="rose" activiti:exclusive="true" id="_6" name="财务审批"/>
    <endEvent id="_7" name="EndEvent"/>
    <sequenceFlow id="_8" sourceRef="_2" targetRef="_3"/>
    <sequenceFlow id="_9" sourceRef="_3" targetRef="_4"/>
    <sequenceFlow id="_10" sourceRef="_4" targetRef="_5"/>
    <sequenceFlow id="_11" sourceRef="_5" targetRef="_6"/>
    <sequenceFlow id="_12" sourceRef="_6" targetRef="_7"/>
  </process>
```

* 在 process 标签上面有一个 id，这个 id 就是用来唯一标识这个请假流程的。
* 在 userTask 标签里面有 assignee属性（分配责任人），同时还有 id 属性， id 属性的作用是被 sequenceFlow 标签进行绑定

## 部署流程

```java
@Test
public void testDeployment() {
    // 1、创建ProcessEngine
    ProcessEngine processEngine = ProcessEngines.getDefaultProcessEngine();
    // 2、获取RepositoryServcie
    RepositoryService repositoryService = processEngine.getRepositoryService();
    // 3、使用service进行流程的部署，定义一个流程的名字，把bpmn和png部署到数据中
    Deployment deploy = repositoryService.createDeployment()
        .name("出差申请流程")
        .addClasspathResource("bpmn/evection.bpmn")
        .addClasspathResource("bpmn/evection.png")
        .deploy();
}
```

* 由于流程是一个资源， 所以需要使用 RepositoryService 接口来进行部署
* 部署流程之后，操作的有三张表， 分别是
  * act_re_procdef ：流程定义表，部署每一个新的流程定义都会在这个表中增加一个记录
  * act_re_deployment：流程定义部署表，每部署一次增加一条记录
  * act_ge_bytearray：流程资源表，比如存储我们的流程图

![](https://gitee.com/GWei11/picture/raw/master/20210415055353.png)

现在如果我们**重新执行一次**上面的部署的代码，发现表中数据如下：

**ACT_RE_DEPLOYMENT 表的数据**

![](https://gitee.com/GWei11/picture/raw/master/20210415060037.png)

**ACT_RE_PROCDEF 表的数据**

![](https://gitee.com/GWei11/picture/raw/master/20210415055959.png)

**ACT_GE_BYTEARRAY 表的数据**

![](https://gitee.com/GWei11/picture/raw/master/20210415055714.png)

* 当流程图中的 process 标签里面的 id 相同的时候，如果多次部署，就会被认为是同一个流程，只是会增加相应的版本，这个在 ACT_RE_PROCDEF 流程定义表中可以看出来。



## 启动流程

启动一个流程表示发起一个新的出差申请单，这就相当于java类与java对象的关系，类定义好后需要new创建一个对象使用，当然可以new多个对象。所以如果执行下面的代码多次，就表示zhangsan发起了多次请假申请（为什么这里是zhangsan这个人呢，其实这是因为我们在流程定义里面把 assignee 写死成 zhangsan 了）

```java
@Test
public void testStartProcess() {
    //1、创建ProcessEngine
    ProcessEngine processEngine = ProcessEngines.getDefaultProcessEngine();
    //2、获取RunTimeService
    RuntimeService runtimeService = processEngine.getRuntimeService();
    //3、根据流程定义的id启动流程
    ProcessInstance instance = runtimeService.startProcessInstanceByKey("myEvection");
}
```

* 启动流程需要根据流程id 进行启动，RuntimeService 里面有很多的重载的可以启动流程的方法，这里使用的是最简单的一个。

### 启动流程操作的表

* act_hi_actinst     流程实例执行历史，我们的每一次对任务的操作都能在这个表查询到（包括开始事件和结束事件都可以在这张表查询到）

* act_hi_identitylink  流程的参与用户历史信息

* act_hi_procinst      流程实例历史信息

* act_hi_taskinst       流程任务历史信息，这里面主要记录的是任务信息，查询不到开始事件，结束事件等信息

* act_ru_execution   流程执行信息

* act_ru_identitylink  流程的参与用户信息

* act_ru_task    运行时任务信息，也就是当前需要执行的任务信息记录在这里，比如按照上面的流程图，刚启动一个流程的时候，zhangsan 需要创建出差申请单，这是 zhangsan 的一个任务，所以在这个表里面会记录这个任务，当 zhangsan 完成当前任务后，就到了经理审批节点，zhangsan 的那个任务就不会在这个表里面。

### 执行任务

我们先看一下 evection.bpmn 文件里面 process 标签的内容

```xml
<process id="myEvection" isClosed="false" isExecutable="true" name="出差申请" processType="None">
    <startEvent id="_2" name="StartEvent"/>
    <userTask activiti:assignee="zhangsan" activiti:exclusive="true" id="_3" name="创建出差申请"/>
    <userTask activiti:assignee="jerry" activiti:exclusive="true" id="_4" name="经理审批"/>
    <userTask activiti:assignee="jack" activiti:exclusive="true" id="_5" name="总经理审批"/>
    <userTask activiti:assignee="rose" activiti:exclusive="true" id="_6" name="财务审批"/>
    <endEvent id="_7" name="EndEvent"/>
    <sequenceFlow id="_8" sourceRef="_2" targetRef="_3"/>
    <sequenceFlow id="_9" sourceRef="_3" targetRef="_4"/>
    <sequenceFlow id="_10" sourceRef="_4" targetRef="_5"/>
    <sequenceFlow id="_11" sourceRef="_5" targetRef="_6"/>
    <sequenceFlow id="_12" sourceRef="_6" targetRef="_7"/>
  </process>
```

之所以要看这个，是因为基础版案例里面的审批人都是写死的，比如创建出差申请单的就是 zhangsan, 经理审批的就是 jerry等等，所以现在完成任务的时候需要将人信息写出来带进去。

流程启动之后，第一个任务就是 zhangsan 这个人要创建出差申请单，所以是 zhangsan 来完成当前任务

```java
@Test
public void completTask() {
    //获取引擎
    ProcessEngine processEngine = ProcessEngines.getDefaultProcessEngine();
    //获取操作任务的服务 TaskService
    TaskService taskService = processEngine.getTaskService();
    Task task = taskService.createTaskQuery()
        .processDefinitionKey("myEvection")
        .taskAssignee("zhangsan")
        .singleResult();
    taskService.complete(task.getId());
}
```

当然，第二次审批的时候就是经理审批，是jerry 审批，将上面的代码中的 taskAssign() 方法里面的 zhangsan 替换成 jerry 即可， 同样的套路，jerry 完成任务后，就换 jack 了，直到最后一个人审批完成，这个流程就完成了。







