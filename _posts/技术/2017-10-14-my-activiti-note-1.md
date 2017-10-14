---
layout: post
title: 从单元测试中学习Activiti（一）
category: 技术
tags: activiti unitTest
keywords: activiti
description: Activiti学习记录1
---

## 从[Activiti单元测试](https://www.activiti.org/userguide/index.html#apiUnitTesting)中学习Activiti（一）

重要的几个类：

![image](https://www.activiti.org/userguide/images/api.services.png)


### 一、MyBusinessProcessTest

来自[Activiti GuideBook](https://www.activiti.org/userguide/index.html#chapterApi)的单元测试例子：
 ```Java
1. public class MyBusinessProcessTest {
2. 
3.   @Rule
4.   public ActivitiRule activitiRule = new ActivitiRule();
5. 
6.   @Test
    //这里要指定自己的流程文件
7.   @Deployment(resource = {"org/activiti/test/myProcess.bpmn20.xml"})
8.   public void ruleUsageExample() {
9.      RuntimeService runtimeService = activitiRule.getRuntimeService();
10.     runtimeService.startProcessInstanceByKey("ruleUsage");
11. 
12.     TaskService taskService = activitiRule.getTaskService();
13.     Task task = taskService.createTaskQuery().singleResult();
14.     assertEquals("My Task", task.getName());
15. 
16.     taskService.complete(task.getId());
17.     assertEquals(0, runtimeService.createProcessInstanceQuery().count());
18.   }
19. }
```

第四行得到**activitiRule**对象，其apply->startingQuietly->starting方法中的`initializeProcessEngine`方法调用***TestHelper***类的getProcessEngine方法，该方法通过
```
ProcessEngineConfiguration
.createProcessEngineConfigurationFromResource(configurationResource)
.buildProcessEngine();
```
获得processEngine对象。`buildProcessEngin`具体实现在ProcessEngineConfigurationImpl.java类。

再往上一层的`createProcessEngineConfigurationFromResource`方法中的***configurationReource***参数，是ActivitiRule类中的默认参数***activiti.cfg.xml***，也可以在实例化activitiRule时加入configuration资源文件名的参数。该方法(*`createProcessEngineConfigurationFromResource`*)具体调用了***BeansConfigurationHelper***类中的***parseProcessEngineConfiguration***方法，里面实例化了***processEngineConfigurationImpl***对象。

---

### 二、[ProcessEngine](https://www.activiti.org/userguide/index.html#apiEngine)
    
```Java
// will initialize and build a process engine the first time it is called and afterwards always return the same process engine
1.  ProcessEngine processEngine = ProcessEngines.getDefaultProcessEngine();
2. 
3.  RuntimeService runtimeService = processEngine.getRuntimeService();
4.  RepositoryService repositoryService = processEngine.getRepositoryService();
5.  TaskService taskService = processEngine.getTaskService();
6.  ManagementService managementService = processEngine.getManagementService();
7.  IdentityService identityService = processEngine.getIdentityService();
8.  HistoryService historyService = processEngine.getHistoryService();
9.  FormService formService = processEngine.getFormService();
10. DynamicBpmnService dynamicBpmnService = processEngine.getDynamicBpmnService();
```


在activitiRule对象中，在**initialProcessEngine**方法实例化processEngine对象后，在***initialServices***方法中获得各service对象。


#### 1. repositoryService

> repositoryService offers operations for managing and manipulating deployments and ***process definitions***
【流程定义】
>
> a process definition is a Java counterpart of BPMN 2.0 process. It is a representation of the ***structure and behaviour*** of each of the steps of a process. 
>
> A ***deployment***【流程部署】 is the unit of packaging within the Activiti engine. A deployment can contain multiple BPMN 2.0 xml files and any other resource.

【与模型相关：activateProcessDefinition（启动）、suspendProcessDefinition（挂起）、设置分类、部署流程等】

#### 2. runTimeService

> It deals with starting new process instances of process definitions. 【如第一部分单元测试代码中的"runtimeService.startProcessInstanceByKey("ruleUsage");"】
>
> is the service which is used to retrieve and store ***process variables***. 【流程实例的变量】
also allows to query on process instances and executions.(Basically an execution is a pointer pointing to where the process instance currently is.)

【与流程实例相关：启动/删除/查询流程实例，更新变量等】

#### 3. TaskService

> Querying tasks assigned to users or groups
> 
> Creating new *standalone* tasks. These are tasks that are not related to a process instances.
> 
> Manipulating to which user a task is assigned or which users are in some way involved with the task.
> 
> ***Claiming*** and ***completing*** a task. Claiming means that someone decided to be the assignee for the task, meaning that this user will complete the task. Completing means doing the work of the tasks. Typically this is filling in a form of sorts.

#### 4. HistoryService

> The HistoryService exposes all historical data gathered by the Activiti engine. When executing processes, a lot of data can be kept by the engine (this is configurable) such as process instance start times, who did which tasks, how long it took to complete the tasks, which path was followed in each process instance, etc. This service exposes mainly query capabilities to access this data.

---

### 三、[Query相关](https://www.activiti.org/userguide/index.html#queryAPI)

> There are two ways of querying data from the engine: The ***query API*** and ***native queries***. 

#### 1. repositoryService中

1. ModelQuery
2. ProcessDefinitionQuery
3. DeploymentQuery


1. NativeModelQuery
2. NativeProcessDefinitionQuery
3. NativeDeploymentQuery
 
#### 2. runTimeService中

1. ExecutionQuery
2. ProcessInstanceQuery


1. NativeExecutionQuery
2. NativeProcessInstanceQuery

#### 3. TaskService中

- TaskQuery
- NativeTaskQuery

```Java
//using query API
List<Task> tasks = taskService.createTaskQuery()
    .taskAssignee("kermit")
    .processVariableValueEquals("orderId", "0815")
    .orderByDueDate().asc()
    .list();

//using native queries
List<Task> tasks = taskService.createNativeTaskQuery()
  .sql("SELECT count(*) FROM " + managementService.getTableName(Task.class) + " T WHERE T.NAME_ = #{taskName}")
  .parameter("taskName", "gonzoTask")
  .list();

long count = taskService.createNativeTaskQuery()
  .sql("SELECT count(*) FROM " + managementService.getTableName(Task.class) + " T1, "
    + managementService.getTableName(VariableInstanceEntity.class) + " V1 WHERE V1.TASK_ID_ = T1.ID_")
  .count();
```

#### 4. HistoryService中

1. HistoricProcessInstanceQuery
2. HistoricActivityInstanceQuery
3. HistoricTaskInstanceQuery
4. HistoricDetailQuery
5. HistoricVariableInstanceQuery
6. *ProceInstanceHistoryLogQuery*


1. NativeHistoricProcessInstanceQuery
2. NativeHistoricActivityInstanceQuery
3. NativeHistoricTaskInstanceQuery
4. NativeHistoricDetailQuery
5. NativeHistoricVariableInstanceQuery

### 四、[Variables相关](https://www.activiti.org/userguide/index.html#apiVariables)

> A process instance can have variables (called process variables), but also executions (which are specific pointers to where the process is active) and user tasks can have variables. A process instance can have any number of variables. Each variable is stored in a row in the ***ACT_RU_VARIABLE*** database table.
【流程实例可以有变量、用户任务也可以有】

runtimeService、taskService中都有很多相关的方法，针对的层级不同。

```
//runtimeService
void setVariable(String executionId, String variableName, Object value);
void setVariableLocal(String executionId, String variableName, Object value);
void setVariables(String executionId, Map<String, ? extends Object> variables);
void setVariablesLocal(String executionId, Map<String, ? extends Object> variables);
```
> Variables are often used in ***Java delegates***, ***expressions***, ***execution*** or ***tasklisteners***, ***scripts***, etc. In those constructs, the current execution or task object is available and it can be used for variable setting and/or retrieval.

```
execution.getVariables();
execution.getVariables(Collection<String> variableNames);
execution.getVariable(String variableName);

execution.setVariables(Map<String, object> variables);
execution.setVariable(String variableName, Object value);
```

有一个需要注意的点是，上面的这些方法会把该流程实例/任务所有的变量取出来：
> For historical (and backwards compatible reasons), when doing any of the calls above, behind the scenes actually ***all variables will be fetched*** from the database.
>
> Since Activiti 5.17, new methods have been introduced to give a tighter control on this, by adding in new methods that have an optional parameter that tells the engine whether or not behind the scenes all variables need to be fetched and cached:【==然而在jeesite中集成的5.21版本，并没有发现这个新的方法==】
```
Map<String, Object> getVariables(Collection<String> variableNames, boolean fetchAllVariables);
Object getVariable(String variableName, boolean fetchAllVariables);
void setVariable(String variableName, Object value, boolean fetchAllVariables);
```

用户指引中还提到了[Transient variable](https://www.activiti.org/userguide/index.html#apiTransientVariables)（临时/瞬时变量？），跟普通的process variable相比，有几点不同：

- There is no history stored at all for transient variables.【不会被保存】
- Transient variables ***can only be set*** by the ***setTransientVariable(name, value)***, but transient variables are also returned when calling *getVariable(name)* (a *getTransientVariable(name)* also exists, that only checks the transient variables). The reason for this is to make the writing of expressions easy and existing logic using variables works for both types.【set只有一个方法，普通的get方法也能获得】
- … … 

> Transient variables can be got and/or set in most places where regular variables are exposed:
> 
> - On DelegateExecution in *JavaDelegate* implementations
> - On DelegateExecution in *ExecutionListener* implementations and on DelegateTask on TaskListener implementations
> - In *script* task via the execution object
> - When starting a process instance via the runtime service
> - When completing a task
> - When calling the runtimeService.trigger method

但同样，==在jeesite集成的5.21版本中没有发现这个属性==
