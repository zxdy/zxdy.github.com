今天终于有时间把spark的源代码看了下，因为之前一直对spark的任务调度流程比较模糊，而网上关于spark的设计实现文档又不太完整，所以这篇文章主要用于梳理这方面的逻辑关系，并不会对代码的细节实现作过多的讨论，我对scala来说，也只能算个新手，许多复杂的语法和实现，无法太深入解读，只能比较宏观地介绍一下其中的逻辑和目的，如果有错漏之处，请多指教。顺便吐槽一下看源代码真是望山跑死马啊，短短一句代码隐藏N多细节。

本文基于**spark 1.3.0**

## Job 提交
当我们向spark提交一个application的时候，首先都会调用程序里的val sc = new SparkContext(sparkConf)语句，这一句创建了一个SparkContext实例，确立整个程序作为driver的地位。

我们知道数据在spark中的处理主要分为[Transformations][1]和[action][2]两种类型。

> * Transformations：属于懒加载，他会先建立一系列的RDD，每个RDD的compute() 定义如何根据上游数据计算当前RDD的结果。每个RDD的getDependencies()定义RDD之间的数据依赖关系。
> * action：是数据最后的reduce过程，只有当程序运行到action的时候才会真正触发生成一个job，即application中有几个action就会提交几个job。
    
从SparkContext的实例来出发，原始的RDD经过一连串的transformation操作，转换成为其它类型的RDD，直到遇到action，触发调用SparkContext的runJob方法   
    
1. SparkContext

    ```java
    createTaskScheduler
            scheduler = new TaskSchedulerImpl
            backend=?
                LocalBackend
                SparkDeploySchedulerBackend
                CoarseGrainedSchedulerBackend
    SparkContext.runjob
        dagScheduler.runJob
            submitJob
                eventProcessLoop.post(JobSubmitted())
                onReceive
                    dagScheduler.handleJobSubmitted
    ```

    SparkContext在初始化的时候会新建TaskSchedulerImpl和LocalBackend（本文以local模式为例，如果是standalone或是yarn则分别为SparkDeploySchedulerBackend和CoarseGrainedSchedulerBackend）实例。这两个实例会在之后task执行时用到。

    SparkContext.runjob是后面一系列反应的起点，它最终调用的是dagScheduler.handleJobSubmitted方法。

2. DAGScheduler

    ```java
    dagScheduler.handleJobSubmitted
        finalStage = newStage()
        submitStage(finalStage)
            getMissingParentStages
            submitMissingTasks
                tasks:Seq[]=new ShuffleMapTask or ResultTask
                taskScheduler.submitTasks
    ```
    DAGScheduler在spark中是非常重要的一个组件，spark任务所谓的有向无环图就是通过这个组件生成。
    *  **finalStage = newStage()** 将整个job根据宽依赖和窄依赖进行stage划分（总体的思想是从最后的finallRDD出发反向递归逻辑执行图，每遇到宽依赖就断开，把之前沿途的窄依赖都加入同一个stage）。同时，将每个stage中的最后一个RDD通过mapOutputTracker.registerShuffle注册到MapOutputTrackerMaster，用于指示ShuffleMapTask最后输出数据的位置。
    
    * **submitStage** 首先调用getMissingParentStages(),确定有没有parentStage，如果有的话，先递归提交parentStage，并将自己加入到 waitingStages 里，直到当前stage没有parentStage，此时stage 可以立即执行，调用submitMissingTasks，根据当前stage的类型（ShuffleMapStage或ResultStage）生成数量跟当前stage最后一个RDD的partition数一样的Tasks（ShuffleMapTasks或ResultTasks）。打包Tasks交给taskScheduler处理。
    
3.  TaskSchedulerImpl
    ```java
    TaskSchedulerImpl.submitTasks
         backend.reviveOffers()
    ```
    TaskSchedulerImpl实现了taskScheduler的接口，这个TaskSchedulerImpl就是之前在第一步产生的TaskSchedulerImpl实例。最后将Tasks交给backend（同样是第一步产生的实例，本文中为了方便使用LocalBackend）处理。


4.  LocalBackend
    ```java
    receiveWithLogging
        reviveOffers
            executor.launchTask(..., task.serializedTask)
    ```
    Backend 接收到 taskSet 后,将序列化后之后的 task 分发到调度器指定的 worker node 上执行

## Job 接收 
1.  Executor

    ```java
    executor.launchTask()
        new TaskRunner()
            run()
                task = ser.deserialize
                value = task.run()
                serializedResult=?
                    IndirectTaskResult
                        write to mem&disk
                    serializedDirectResult
                execBackend.statusUpdate(taskId, TaskState.FINISHED, serializedResult)
    ```
    Executor 收到序列化后的task，首先进行反序列化，然后运行 task 得到执行结果 directResult。序列化directResult后，得到其大小，如果大于 spark.driver.maxResultSize 或者akkaFrameSize - AkkaUtils.reservedSizeBytes，将结果写入内存或磁盘（根据conf配置），由 blockManager 管理，只返回存储位置信息的IndirectTaskResult。否则就将结果serializedDirectResult直接返回给driver。task结束调用execBackend.statusUpdate()。

2.  LocalBackend
    ```java
    receiveWithLogging
        scheduler.statusUpdate
    ```
    
3.  TaskSchedulerImpl
    ```java
    statusUpdate
        //IF TaskState.FINISHED
        taskResultGetter.enqueueSuccessfulTask()
            serializedTaskResult=blockManager.getRemoteBytes()
            scheduler.handleSuccessfulTask
        taskSetManager.handleSuccessfulTask
             sched.dagScheduler.taskEnded()
    ```
    通知TaskSchedulerImpl task已经执行完，最后result如果是IndirectTaskResult，则还需调用 blockManager.getRemoteBytes() 去拿到实际的 result。通知dagScheduler Task执行结束。

4.  dagScheduler
    ```java
    dagScheduler.handleTaskCompletion
        //IF TASK IS ResultTask
            // If the whole job has finished, remove it
            markStageAsFinished
            listenerBus.post(SparkListenerJobEnd)
            job.listener.taskSucceeded
        //IF TASK IS ShuffleMapTask
            //IF ALL TASKS DONE IN CURRENT STAGE
            stage.addOutputLoc(smt.partitionId, status)
            mapOutputTracker.registerMapOutputs()
            submitMissingTasks
    ```
    dagScheduler 判断当前结束的task类型，假如是ResultTask，继续判断是不是job已经执行完毕。假如是task类型是ShuffleMapTask，判断当前stage的所有task是不是都已经运行完毕，如果是的话将当前stage的执行结果注册到mapOutputTrackerMaster给下一个被依赖的stage使用，并继续提交等待中的stage。
    
## 结束
本文主要介绍了spark job的一个完整生命周期（抱歉没有画图，有空补），下一篇准备结合一个具体的例子来继续讲一下。目前对stage之间的shuffle连接没有做深入研究，shuffle write在task结束后，但是没有发现task开始时的shuffle read，不知是不是被封装在各个RDD的compute里了。



  [1]: http://spark.apache.org/docs/latest/programming-guide.html#transformations
  [2]: http://spark.apache.org/docs/latest/programming-guide.html#actions