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
  
跟任务调度相关的消息类型就是SubmitJob，从客户端接收到JobGraph，并在JobManager上执行ExecuteGraph的构建过程，而构建ExecuteGraph过程具体实现如下所示：

```scala
JobManager line 1216

 private def submitJob(jobGraph: JobGraph, jobInfo: JobInfo, isRecovery: Boolean = false): Unit = {
  ...
  // 根据JobGraph构建ExecuteGraph
  executionGraph = ExecutionGraphBuilder.buildGraph(
          executionGraph,
          jobGraph,
          flinkConfiguration,
          futureExecutor,
          ioExecutor,
          scheduler,
          userCodeLoader,
          checkpointRecoveryFactory,
          Time.of(timeout.length, timeout.unit),
          restartStrategy,
          jobMetrics,
          numSlots,
          blobServer,
          Time.milliseconds(allocationTimeout),
          log.logger)
  ...
  
  if (leaderSessionID.isDefined &&
            leaderElectionService.hasLeadership(leaderSessionID.get)) {
            // There is a small chance that multiple job managers schedule the same job after if
            // they try to recover at the same time. This will eventually be noticed, but can not be
            // ruled out from the beginning.

            // NOTE: Scheduling the job for execution is a separate action from the job submission.
            // The success of submitting the job must be independent from the success of scheduling
            // the job.
            log.info(s"Scheduling job $jobId ($jobName).")

            // 调度任务
            executionGraph.scheduleForExecution()
          } else {
            // Remove the job graph. Otherwise it will be lingering around and possibly removed from
            // ZooKeeper by this JM.
            self ! decorateMessage(RemoveJob(jobId, removeJobFromStateBackend = false))

            log.warn(s"Submitted job $jobId, but not leader. The other leader needs to recover " +
              "this. I am not scheduling the job for execution.")
          }
     ...     
          
```
上述代码逻辑总结起来是：首先做一些准备工作，然后根据JobGraph构建ExecuteGraph，判断是否是恢复的job，然后将job保持下来，并且通知客户端端本地以及提交成功，最后判断当前JobManager是否是leader，如果是则执行executionGraph.scheduleForExecution()方法，这个方法经过一系列调用，把每个ExecutionVertex传递给Execute类的deploy()方法

```scala
ExecuteGraph line 932

	private CompletableFuture<Void> scheduleEager(SlotProvider slotProvider, final Time timeout) {
		checkState(state == JobStatus.RUNNING, "job is not running currently");
		final boolean queued = allowQueuedScheduling;

		final ArrayList<CompletableFuture<Execution>> allAllocationFutures = new ArrayList<>(getNumberOfExecutionJobVertices());

		// allocate the slots (obtain all their futures
		for (ExecutionJobVertex ejv : getVerticesTopologically()) {
			// these calls are not blocking, they only return futures
			Collection<CompletableFuture<Execution>> allocationFutures = ejv.allocateResourcesForAll(
				slotProvider,
				queued,
				LocationPreferenceConstraint.ALL,
				allocationTimeout);

			allAllocationFutures.addAll(allocationFutures);
		}

		// this future is complete once all slot futures are complete.
		// the future fails once one slot future fails.
		final ConjunctFuture<Collection<Execution>> allAllocationsFuture = FutureUtils.combineAll(allAllocationFutures);

		final CompletableFuture<Void> currentSchedulingFuture = allAllocationsFuture
			.thenAccept(
				(Collection<Execution> executionsToDeploy) -> {
					for (Execution execution : executionsToDeploy) {
						try {
              // 开始执行task
							execution.deploy();
						} catch (Throwable t) {
							throw new CompletionException(
								new FlinkException(
									String.format("Could not deploy execution %s.", execution),
									t));
						}
					}
				})
			...

		return currentSchedulingFuture;
	}
```

```scala
Execution line 536

public void deploy() throws JobException {
		...

		try {
			...	
			// task描述符
			final TaskDeploymentDescriptor deployment = vertex.createDeploymentDescriptor(
				attemptId,
				slot,
				taskRestore,
				attemptNumber);

			// null taskRestore to let it be GC'ed
			taskRestore = null;

			final TaskManagerGateway taskManagerGateway = slot.getTaskManagerGateway();

			// 提交task任务到TM上
			final CompletableFuture<Acknowledge> submitResultFuture = taskManagerGateway.submitTask(deployment, rpcTimeout);

			submitResultFuture.whenCompleteAsync(
				(ack, failure) -> {
					// only respond to the failure case
					if (failure != null) {
						if (failure instanceof TimeoutException) {
							String taskname = vertex.getTaskNameWithSubtaskIndex() + " (" + attemptId + ')';

							markFailed(new Exception(
								"Cannot deploy task " + taskname + " - TaskManager (" + getAssignedResourceLocation()
									+ ") not responding after a rpcTimeout of " + rpcTimeout, failure));
						} else {
							markFailed(failure);
						}
					}
				},
				executor);
		}
		catch (Throwable t) {
			markFailed(t);
			ExceptionUtils.rethrow(t);
		}
	}
```
首先我们创建一个TaskDeploymentDescriptor，然后通过TaskManager的网关提交任务（taskManagerGateway.submitTask(..)），下面任务就被发送到TaskManager里面了。

注意：TaskDeploymentDescriptor的组件包含JobId，JobInformation(序列化之后的)，TaskInformation(序列化之后的)，槽位信息等等

