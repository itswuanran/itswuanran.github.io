---
title: 任务执行引擎设计
date: 2018-11-08 15:19:28
tags:
---
## 一个简单的任务执行引擎设计（转）

## 前言
最近做的一个项目是一个数据库服务化的管控平台，用时髦一点的名词来说是一个DBaaS产品。这种面向云化的产品，呈现给最终用户的体验是提供一个管理页面，把数据库的生命周期，监控等功能通过WEB页面或者Open API暴露给用户或者第三方的程序，常见的产品类似于阿里云或者AWS的RDS。而我们的做的产品实际上是一个分布式的数据库服务平台，除了底层的存储，还有上层的proxy去完成分库分表，读写分离等操作。

对于终端用户来说，使用的是一个数据库连接，但实际上在后面，会有很多系统一同工作。例如当用户创建一个RDS的时候，会去创建底层的数据库实例（MySQL，SQL Server等），Loader Blance，Proxy等，而这些组件其实也是由其他系统通过Open API或者RPC的方式暴露给上层应用。作为Paas或者DBaaS的最上层的产品，免不了会调用其他的系统接口去申请资源。那么在代码实现上会碰到一个问题，当依赖多个系统的时候，依次调用各个系统的的过程中，如果中途出错，错误处理比较难。

## 之前状况：
之前的项目代码中，没有统一的任务执行框架，由各个开发人员自行去编码，那么经常看到这种代码

```java
try{
    resultA = call_system_a;
    resultB = call_system_b;
} catch(Exception e){
    if ( ! resultA ){
         do_some_clean_work_sysytem_a;
    } else ( result && ! reusltB ) {
　　     do_some_clean_word_system_a;
         do_some_clean_work_system_b;
    }
}        
```
为什么要这么写呢，是因为当出了异常以后，你需要判断是a系统调用失败了，还是b系统调用失败了。并且把之前调用其他系统的资源给及时释放掉。

上面这段代码只是大致演示了当流程涉及到两个系统的时候，代码是怎么样，实际上，完成一个云资源的申请，会涉及到 5 6 个系统，如果中间一步出错，需要在代码里面控制如何回滚，是非常难的。因为系统间的调用多数是通过HTTP或者RPC的方式，而不像数据库可用事物控制。

## 统一的任务执行API
为了在新的项目里面规避之前实现不合理的带来的问题，我新写了一个package，将原来树形的流程控制（即多个if else嵌套去判断是否系统调用出问题），改成了线性的流程控制，这样简化的编程模型，统一了项目组成员编写代码的风格。

对于要执行的任务，抽象出任务和步骤两个核心领域模型，对于每一次任务的执行抽象出任务执行实例这个模型，整个系统的模型如下图所示：
![](images/task/api.png)
这样，之前在一个方法里面去调用多个系统，变成多个方法的组合，各个方法只关注单个系统的调用和出错处理，简化了编程模型。

为了实现错误处理，在step中还定义了onError接口，在出错的时候，框架按照先入后出的顺序会调用各个step的onError方法。Step的接口定义如下：

```java
public interface  Step {

    String getStepName();

    void beforeExecute(TaskExecution taskExecution);

    /**
     * 该步骤主要的业务逻辑实现，如果抛出任何异常，表示改步骤执行失败，会调用该步骤和已经执行完的步骤的onError方法
     * @param taskExecution
     */
    void onExecute(TaskExecution taskExecution);

    void onComplete(TaskExecution taskExecution);

    /**
     * 该步骤出错时会调用，如果抛出异常，会被框架给忽略掉。已完成其他步骤的onError方法调用。
     * @param execution
     * @param throwable
     */
    void onError(TaskExecution execution, Throwable throwable);

}
```

之前提到了按照先入后出的顺序去回调onError方法，自然就想到用stack去保存已经执行过的步骤，任务执行引擎的主要过程和代码如下：
 ![](images/task/schedule.png)

```java
    private TaskExecution doExecute(Task Task, TaskExecution taskExecution) {
        Stack<Step> StepStack = new Stack<>();
        for (Step step : Task.getSteps()) {
            StepStack.push(step);
            StepExecution StepExecution = executeSingleStep(taskExecution, step);
            if (StepExecution.getStatus() == StepStatus.FAILED) {
                rollBack(StepStack, taskExecution);
                taskExecution = taskExecutionRepository.updateStatus(taskExecution, FAILED);
                return taskExecution;
            } else {
                taskExecution.setPercent(calculatePercent(StepStack.size(), Task.getSteps().size()));
            }
        }
        return taskExecutionRepository.updateStatus(taskExecution, COMPLETED);
    }
 ```

 其中TaskExecutionRepository主要用来做任务执行状态的持久化。

## 其他：
除去统一了任务执行的编程模型，在整个系统的领域层，也抽取出Task这个领域对象，前端通过rest接口的方式暴露一个统一的任务提交接口，返回一个TaskExecution对象之后由后端去异步执行任务。前端根据该对象的唯一ID可以查询当前任务执行的状态，耗时时间，完成百分比等信息。这类信息对于云产品的运维提供了方便。在以后新加业务功能的时候，只要新加任务类型即可。

## 参考
- https://www.cnblogs.com/javanerd/p/6412787.html