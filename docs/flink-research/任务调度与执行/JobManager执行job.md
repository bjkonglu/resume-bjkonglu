## JobManager执行job

JobManager的实现注释是这样描述它的，Job manager负责接收Flink的job，调度任务（task），收集job状态和管理TaskManagers。

### JobManager的组件
  - BlobServer是用来管理二进制大文件服务的
  - BlobLibraryCacheManager是用来为job下载它依赖的库文件（例如Jar文件）
  - InstanceManager是用来管理当前存活的TaskManagers的组件，记录了TaskManager的心跳信息等
  - CompletedCheckpointStore用来保存已完成的checkpoint相关信息，持久化到内存中或者zookeeper上
  - SubmittedJobGraphStore保存已经提交到JobManager的JobGraph信息
  
### JobManager的启动
先看一下JobManager启动的核心代码，如下： 
```scala
JobManager line 1998

  def runJobManager(
      configuration: Configuration,
      executionMode: JobManagerMode,
      listeningAddress: String,
      listeningPort: Int)
    : Unit = {

    val numberProcessors = Hardware.getNumberCPUCores()

    val futureExecutor = Executors.newScheduledThreadPool(
      numberProcessors,
      new ExecutorThreadFactory("jobmanager-future"))

    val ioExecutor = Executors.newFixedThreadPool(
      numberProcessors,
      new ExecutorThreadFactory("jobmanager-io"))

    val timeout = AkkaUtils.getTimeout(configuration)

    // we have to first start the JobManager ActorSystem because this determines the port if 0
    // was chosen before. The method startActorSystem will update the configuration correspondingly.
    val jobManagerSystem = startActorSystem(
      configuration,
      listeningAddress,
      listeningPort)

    val highAvailabilityServices = HighAvailabilityServicesUtils.createHighAvailabilityServices(
      configuration,
      ioExecutor,
      AddressResolution.NO_ADDRESS_RESOLUTION)

    val metricRegistry = new MetricRegistryImpl(
      MetricRegistryConfiguration.fromConfiguration(configuration))

    metricRegistry.startQueryService(jobManagerSystem, null)

    val (_, _, webMonitorOption, _) = try {
      startJobManagerActors(
        jobManagerSystem,
        configuration,
        executionMode,
        listeningAddress,
        futureExecutor,
        ioExecutor,
        highAvailabilityServices,
        metricRegistry,
        classOf[JobManager],
        classOf[MemoryArchivist],
        Option(classOf[StandaloneResourceManager])
      )
    } catch {
      case t: Throwable =>
        futureExecutor.shutdownNow()
        ioExecutor.shutdownNow()

        throw t
    }

    // block until everything is shut down
    jobManagerSystem.awaitTermination()
    ...
  }
```
上述启动JobManager核心代码可以总结如下步骤：
  - 启动jobManagerSystem，它是一个ActorSystem
  - 启动HA和metric服务
  - 在startJobManagerActor(..)方法中启动JobManagerActors，以及webserver，TaskManagerActor，ResourceManagerActor等等
  - 阻塞等待终止
  - 集群通过LeaderService等选举出JobManager的leader
  
### JobManager调度Task

JobManager是一个Actor，主要工作是接收消息并相应消息请求
```scala
JobManager line 256

override def handleMessage: Receive = {

    case GrantLeadership(newLeaderSessionID) =>

    case RevokeLeadership =>
      
    case msg: RegisterResourceManager =>

    case msg: ReconnectResourceManager =>

    case msg @ RegisterTaskManager(
          resourceId,
          connectionInfo,
          hardwareInformation,
          numberOfSlots) =>

    case msg: ResourceRemoved =>

    case RequestNumberRegisteredTaskManager =>

    case RequestTotalNumberOfSlots =>

    // 处理提交JobGraph
    case SubmitJob(jobGraph, listeningBehaviour) =>
      val client = sender()

      val jobInfo = new JobInfo(client, listeningBehaviour, System.currentTimeMillis(),
        jobGraph.getSessionTimeout)

      submitJob(jobGraph, jobInfo)

    case RegisterJobClient(jobID, listeningBehaviour) =>

    case RecoverSubmittedJob(submittedJobGraph) =>

    case RecoverJob(jobId) =>

    case RecoverAllJobs =>
      
    case CancelJob(jobID) =>

    case CancelJobWithSavepoint(jobId, savepointDirectory) =>

    case StopJob(jobID) =>

    case UpdateTaskExecutionState(taskExecutionState) =>

    case RequestNextInputSplit(jobID, vertexID, executionAttempt) =>

    case checkpointMessage : AbstractCheckpointMessage =>

    case kvStateMsg : KvStateMessage =>

    case TriggerSavepoint(jobId, savepointDirectory) =>

    case DisposeSavepoint(savepointPath) =>

    ....
}
```
在这么多类型的消息里，有几类比较重要的消息：
  - GrantLeadership获取leader授权，将自身被分发到的session id写到zookeeper，并恢复所有jobs
  - RevokeLeadership剥夺leader授权，打断清空所有job信息，但是保留作业缓存，注销所有TaskManagers
  - RegisterTaskManager注册TaskManager，如果之前已经注册过。则只给对应Instance发送消息，否则启动注册逻辑：在InstanceManager中注册Instance
信息
  - SubmitJob提交jobGraph
  
