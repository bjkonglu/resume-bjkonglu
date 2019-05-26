## TaskManager执行task

### TaskManager的基本组件
TaskManager是Flink中资源管理的基本组件，是所有执行任务的基本容器，提供了内存管理、IO管理、网络管理等一系列功能，下面对各个模块具体介绍：
  - 内存管理（MemoryManager）Flink并没有把所有内存的管理都委托给JVM，因为JVM普遍存在存储对象密度过低、大内存时GC对系统性能影响较大的问题。所以Flink自己抽象了一套内存管理机制，将所有对象系列化后存放在自己的MemorySegment上进行管理。
  - IO管理（IOMemoryManager）Flink通过IOManager管理磁盘IO的过程，提供了同步和异步两个写模式，又进一步区分了block、buffer和bulk三种读写方式。IOManager提供了两种方式枚举磁盘文件，一种是直接遍历文件夹下所有文件，另一种是计数器方式，对每个文件名以递增顺序访问。
  - 网络网络（NetworkEnvironment）NetworkEnvironment是TaskManager的网络IO组件，包括了最终中间结果和数据交换数据结构。它的构造函数会统一将配置的内存先分配出来，抽象成NetworkBufferPool统一管理内存的申请和释放。举个例子，在输入和输出数据时，不管是保留在本地内存，等待chain在一起的下一个算子进行处理，还是通过网络把本算子的计算结果发送出去，都被抽象成NetworkBufferPool。
  
### TaskManager执行task
对于TaskManager来说，执行task就是将接收到的TaskDeploymentDescriptor对象转换成一个task并执行的过程。TaskDeploymentDescriptor这个类包含了task执行所需要的所有内容，例如序列化的算子，输入的InputGates和输出的producedPartitions，该task需要作为几个subtask执行等信息。

TaskManager从JobManager那里接收到执行的任务信息，然后交给submitTask(..)处理
```scala
TaskManager line 1120

  private def submitTask(tdd: TaskDeploymentDescriptor): Unit = {
    try {
      // grab some handles and sanity check on the fly
      val jobManagerActor = currentJobManager match {
        case Some(jm) => jm
        case None =>
          throw new IllegalStateException("TaskManager is not associated with a JobManager.")
      }
      val libCache = libraryCacheManager match {
        case Some(manager) => manager
        case None => throw new IllegalStateException("There is no valid library cache manager.")
      }
      val blobCache = this.blobCache match {
        case Some(manager) => manager
        case None => throw new IllegalStateException("There is no valid BLOB cache.")
      }

      val fileCache = this.fileCache match {
        case Some(manager) => manager
        case None => throw new IllegalStateException("There is no valid file cache.")
      }

      val slot = tdd.getTargetSlotNumber
      if (slot < 0 || slot >= numberOfSlots) {
        throw new IllegalArgumentException(s"Target slot $slot does not exist on TaskManager.")
      }

      val (checkpointResponder,
        partitionStateChecker,
        resultPartitionConsumableNotifier,
        taskManagerConnection) = connectionUtils match {
        case Some(x) => x
        case None => throw new IllegalStateException("The connection utils have not been " +
                                                       "initialized.")
      }

      // create the task. this does not grab any TaskManager resources or download
      // any libraries except for offloaded TaskDeploymentDescriptor data which
      // was too big for the RPC - the operation may only block for the latter

      val jobManagerGateway = new AkkaActorGateway(jobManagerActor, leaderSessionID.orNull)

      try {
        tdd.loadBigData(blobCache.getPermanentBlobService);
      } catch {
        case e @ (_: IOException | _: ClassNotFoundException) =>
          throw new IOException("Could not deserialize the job information.", e)
      }

      val jobInformation = try {
        tdd.getSerializedJobInformation.deserializeValue(getClass.getClassLoader)
      } catch {
        case e @ (_: IOException | _: ClassNotFoundException) =>
          throw new IOException("Could not deserialize the job information.", e)
      }
      if (tdd.getJobId != jobInformation.getJobId) {
        throw new IOException(
          "Inconsistent job ID information inside TaskDeploymentDescriptor (" +
          tdd.getJobId + " vs. " + jobInformation.getJobId + ")")
      }

      val taskInformation = try {
        tdd.getSerializedTaskInformation.deserializeValue(getClass.getClassLoader)
      } catch {
        case e@(_: IOException | _: ClassNotFoundException) =>
          throw new IOException("Could not deserialize the job vertex information.", e)
      }

      val taskMetricGroup = taskManagerMetricGroup.addTaskForJob(
        jobInformation.getJobId,
        jobInformation.getJobName,
        taskInformation.getJobVertexId,
        tdd.getExecutionAttemptId,
        taskInformation.getTaskName,
        tdd.getSubtaskIndex,
        tdd.getAttemptNumber)

      val inputSplitProvider = new TaskInputSplitProvider(
        jobManagerGateway,
        jobInformation.getJobId,
        taskInformation.getJobVertexId,
        tdd.getExecutionAttemptId,
        new FiniteDuration(
          config.getTimeout().getSize(),
          config.getTimeout().getUnit()))

      val jobID = jobInformation.getJobId

      // Allocation ids do not work properly in legacy mode, so we just fake one, based on the jid.
      val fakeAllocationID = new AllocationID(jobID.getLowerPart, jobID.getUpperPart)

      val taskLocalStateStore = taskManagerLocalStateStoresManager.localStateStoreForSubtask(
        jobID,
        fakeAllocationID,
        taskInformation.getJobVertexId,
        tdd.getSubtaskIndex)

      val taskStateManager = new TaskStateManagerImpl(
        jobID,
        tdd.getExecutionAttemptId,
        taskLocalStateStore,
        tdd.getTaskRestore,
        checkpointResponder)
      // 真正要执行的任务
      val task = new Task(
        jobInformation,
        taskInformation,
        tdd.getExecutionAttemptId,
        tdd.getAllocationId,
        tdd.getSubtaskIndex,
        tdd.getAttemptNumber,
        tdd.getProducedPartitions,
        tdd.getInputGates,
        tdd.getTargetSlotNumber,
        memoryManager,
        ioManager,
        network,
        bcVarManager,
        taskStateManager,
        taskManagerConnection,
        inputSplitProvider,
        checkpointResponder,
        blobCache,
        libCache,
        fileCache,
        config,
        taskMetricGroup,
        resultPartitionConsumableNotifier,
        partitionStateChecker,
        context.dispatcher)

      log.info(s"Received task ${task.getTaskInfo.getTaskNameWithSubtasks()}")

      val execId = tdd.getExecutionAttemptId
      // add the task to the map
      val prevTask = runningTasks.put(execId, task)
      if (prevTask != null) {
        // already have a task for that ID, put if back and report an error
        runningTasks.put(execId, prevTask)
        throw new IllegalStateException("TaskManager already contains a task for id " + execId)
      }

      // all good, we kick off the task, which performs its own initialization
      // 开始执行任务
      task.startTaskThread()

      sender ! decorateMessage(Acknowledge.get())
    }
    catch {
      case t: Throwable =>
        log.error("SubmitTask failed", t)
        sender ! decorateMessage(Status.Failure(t))
    }
  }
```
总结起来submitTask(..)方法的逻辑主要是：
  - 确定资源，例如确定与JobManager的连接，依赖的库文件信息以及当前TaskManager是否有任务槽位资源等
  - 从序列化的job信息里反序列化出job信息类(JobInformation)
  - 从序列化的task信息里反序列化出task信息类(TaskInformation)
  - 获取当前task的输入信息(inputSplitProvider)
  - 最后收集上述信息，并创建一个新的Task对象，代表将要执行的任务
  - 开始执行Task，并向JobManager上报已经启动Task了

下面在具体介绍一下Task的构建和Task的运行

#### Task的构建
在构建Task时，第一步是把构造函数里面的变量赋值给当前task的成员变量，接下来是初始化producedPartition和inputGates。这两个变量描述了当前task的输出数据和输入数据。
```scala
Task line 277

public Task(
		JobInformation jobInformation,
		TaskInformation taskInformation,
		ExecutionAttemptID executionAttemptID,
		AllocationID slotAllocationId,
		int subtaskIndex,
		int attemptNumber,
		Collection<ResultPartitionDeploymentDescriptor> resultPartitionDeploymentDescriptors,
		Collection<InputGateDeploymentDescriptor> inputGateDeploymentDescriptors,
		int targetSlotNumber,
		MemoryManager memManager,
		IOManager ioManager,
		NetworkEnvironment networkEnvironment,
		BroadcastVariableManager bcVarManager,
		TaskStateManager taskStateManager,
		TaskManagerActions taskManagerActions,
		InputSplitProvider inputSplitProvider,
		CheckpointResponder checkpointResponder,
		BlobCacheService blobService,
		LibraryCacheManager libraryCache,
		FileCache fileCache,
		TaskManagerRuntimeInfo taskManagerConfig,
		@Nonnull TaskMetricGroup metricGroup,
		ResultPartitionConsumableNotifier resultPartitionConsumableNotifier,
		PartitionProducerStateChecker partitionProducerStateChecker,
		Executor executor) {

		...
		// Produced intermediate result partitions
		this.producedPartitions = new ResultPartition[resultPartitionDeploymentDescriptors.size()];

		int counter = 0;

		for (ResultPartitionDeploymentDescriptor desc: resultPartitionDeploymentDescriptors) {
			ResultPartitionID partitionId = new ResultPartitionID(desc.getPartitionId(), executionId);

			this.producedPartitions[counter] = new ResultPartition(
				taskNameWithSubtaskAndId,
				this,
				jobId,
				partitionId,
				desc.getPartitionType(),
				desc.getNumberOfSubpartitions(),
				desc.getMaxParallelism(),
				networkEnvironment.getResultPartitionManager(),
				resultPartitionConsumableNotifier,
				ioManager,
				desc.sendScheduleOrUpdateConsumersMessage());

			++counter;
		}

		// Consumed intermediate result partitions
		this.inputGates = new SingleInputGate[inputGateDeploymentDescriptors.size()];
		this.inputGatesById = new HashMap<>();

		counter = 0;

		for (InputGateDeploymentDescriptor inputGateDeploymentDescriptor: inputGateDeploymentDescriptors) {
			SingleInputGate gate = SingleInputGate.create(
				taskNameWithSubtaskAndId,
				jobId,
				executionId,
				inputGateDeploymentDescriptor,
				networkEnvironment,
				this,
				metricGroup.getIOMetricGroup());

			inputGates[counter] = gate;
			inputGatesById.put(gate.getConsumedResultId(), gate);

			++counter;
		}

		invokableHasBeenCanceled = new AtomicBoolean(false);

		// finally, create the executing thread, but do not start it
    // 最后，创建一个包含自己的线程
		executingThread = new Thread(TASK_THREADS_GROUP, this, taskNameWithSubtask);
	}
```
最后，创建一个Thread对象，并把自己放入到该线程对象里，这样在执行时，Task就有了自身的线程的引用。
#### Task的运行

Task类实现了Runnable接口，因此在其run()方法里

