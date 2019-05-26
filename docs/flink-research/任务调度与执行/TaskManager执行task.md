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

Task类实现了Runnable接口，因此在其run()方法里定义了运行逻辑。

```scala
Task line 528

public void run() {

		// ----------------------------
		//  Initial State transition
		// ----------------------------
		while (true) {
			ExecutionState current = this.executionState;
			if (current == ExecutionState.CREATED) {
				if (transitionState(ExecutionState.CREATED, ExecutionState.DEPLOYING)) {
					// success, we can start our work
					break;
				}
			}
			else if (current == ExecutionState.FAILED) {
				// we were immediately failed. tell the TaskManager that we reached our final state
				notifyFinalState();
				if (metrics != null) {
					metrics.close();
				}
				return;
			}
			else if (current == ExecutionState.CANCELING) {
				if (transitionState(ExecutionState.CANCELING, ExecutionState.CANCELED)) {
					// we were immediately canceled. tell the TaskManager that we reached our final state
					notifyFinalState();
					if (metrics != null) {
						metrics.close();
					}
					return;
				}
			}
			else {
				if (metrics != null) {
					metrics.close();
				}
				throw new IllegalStateException("Invalid state for beginning of operation of task " + this + '.');
			}
		}

		// all resource acquisitions and registrations from here on
		// need to be undone in the end
		Map<String, Future<Path>> distributedCacheEntries = new HashMap<>();
		AbstractInvokable invokable = null;

		try {
			// ----------------------------
			//  Task Bootstrap - We periodically
			//  check for canceling as a shortcut
			// ----------------------------

			// activate safety net for task thread
			LOG.info("Creating FileSystem stream leak safety net for task {}", this);
			FileSystemSafetyNet.initializeSafetyNetForThread();

			blobService.getPermanentBlobService().registerJob(jobId);

			// first of all, get a user-code classloader
			// this may involve downloading the job's JAR files and/or classes
			LOG.info("Loading JAR files for task {}.", this);

			userCodeClassLoader = createUserCodeClassloader();
			final ExecutionConfig executionConfig = serializedExecutionConfig.deserializeValue(userCodeClassLoader);

			if (executionConfig.getTaskCancellationInterval() >= 0) {
				// override task cancellation interval from Flink config if set in ExecutionConfig
				taskCancellationInterval = executionConfig.getTaskCancellationInterval();
			}

			if (executionConfig.getTaskCancellationTimeout() >= 0) {
				// override task cancellation timeout from Flink config if set in ExecutionConfig
				taskCancellationTimeout = executionConfig.getTaskCancellationTimeout();
			}

			if (isCanceledOrFailed()) {
				throw new CancelTaskException();
			}

			// ----------------------------------------------------------------
			// register the task with the network stack
			// this operation may fail if the system does not have enough
			// memory to run the necessary data exchanges
			// the registration must also strictly be undone
			// ----------------------------------------------------------------

			LOG.info("Registering task at network: {}.", this);

			network.registerTask(this);

			// add metrics for buffers
			this.metrics.getIOMetricGroup().initializeBufferMetrics(this);

			// register detailed network metrics, if configured
			if (taskManagerConfig.getConfiguration().getBoolean(TaskManagerOptions.NETWORK_DETAILED_METRICS)) {
				// similar to MetricUtils.instantiateNetworkMetrics() but inside this IOMetricGroup
				MetricGroup networkGroup = this.metrics.getIOMetricGroup().addGroup("Network");
				MetricGroup outputGroup = networkGroup.addGroup("Output");
				MetricGroup inputGroup = networkGroup.addGroup("Input");

				// output metrics
				for (int i = 0; i < producedPartitions.length; i++) {
					ResultPartitionMetrics.registerQueueLengthMetrics(
						outputGroup.addGroup(i), producedPartitions[i]);
				}

				for (int i = 0; i < inputGates.length; i++) {
					InputGateMetrics.registerQueueLengthMetrics(
						inputGroup.addGroup(i), inputGates[i]);
				}
			}

			// next, kick off the background copying of files for the distributed cache
			try {
				for (Map.Entry<String, DistributedCache.DistributedCacheEntry> entry :
						DistributedCache.readFileInfoFromConfig(jobConfiguration)) {
					LOG.info("Obtaining local cache file for '{}'.", entry.getKey());
					Future<Path> cp = fileCache.createTmpFile(entry.getKey(), entry.getValue(), jobId, executionId);
					distributedCacheEntries.put(entry.getKey(), cp);
				}
			}
			catch (Exception e) {
				throw new Exception(
					String.format("Exception while adding files to distributed cache of task %s (%s).", taskNameWithSubtask, executionId), e);
			}

			if (isCanceledOrFailed()) {
				throw new CancelTaskException();
			}

			// ----------------------------------------------------------------
			//  call the user code initialization methods
			// ----------------------------------------------------------------

			TaskKvStateRegistry kvStateRegistry = network.createKvStateTaskRegistry(jobId, getJobVertexId());

			Environment env = new RuntimeEnvironment(
				jobId,
				vertexId,
				executionId,
				executionConfig,
				taskInfo,
				jobConfiguration,
				taskConfiguration,
				userCodeClassLoader,
				memoryManager,
				ioManager,
				broadcastVariableManager,
				taskStateManager,
				accumulatorRegistry,
				kvStateRegistry,
				inputSplitProvider,
				distributedCacheEntries,
				producedPartitions,
				inputGates,
				network.getTaskEventDispatcher(),
				checkpointResponder,
				taskManagerConfig,
				metrics,
				this);

			// now load and instantiate the task's invokable code
			invokable = loadAndInstantiateInvokable(userCodeClassLoader, nameOfInvokableClass, env);

			// ----------------------------------------------------------------
			//  actual task core work
			// ----------------------------------------------------------------

			// we must make strictly sure that the invokable is accessible to the cancel() call
			// by the time we switched to running.
			this.invokable = invokable;

			// switch to the RUNNING state, if that fails, we have been canceled/failed in the meantime
			if (!transitionState(ExecutionState.DEPLOYING, ExecutionState.RUNNING)) {
				throw new CancelTaskException();
			}

			// notify everyone that we switched to running
			notifyObservers(ExecutionState.RUNNING, null);
			taskManagerActions.updateTaskExecutionState(new TaskExecutionState(jobId, executionId, ExecutionState.RUNNING));

			// make sure the user code classloader is accessible thread-locally
			executingThread.setContextClassLoader(userCodeClassLoader);

			// run the invokable
			// 执行用户代码
			invokable.invoke();

			// make sure, we enter the catch block if the task leaves the invoke() method due
			// to the fact that it has been canceled
			if (isCanceledOrFailed()) {
				throw new CancelTaskException();
			}

			// ----------------------------------------------------------------
			//  finalization of a successful execution
			// ----------------------------------------------------------------

			// finish the produced partitions. if this fails, we consider the execution failed.
			for (ResultPartition partition : producedPartitions) {
				if (partition != null) {
					partition.finish();
				}
			}

			// try to mark the task as finished
			// if that fails, the task was canceled/failed in the meantime
			if (transitionState(ExecutionState.RUNNING, ExecutionState.FINISHED)) {
				notifyObservers(ExecutionState.FINISHED, null);
			}
			else {
				throw new CancelTaskException();
			}
		}
		...	
	}
```

代码逻辑总结起来大概是这样：
  - 根据任务的当前状态切换状态
  - 导入用户类加载器并加载用户代码
  - 然后向网络管理注册当前任务（Flink的各个算子在运行时进行数据交换需要依赖网络管理器），分配一些缓存来保存数据
  - 创建执行环境Environment
  - 最后便是invokeable.invoke()
  
 下面我们在具体介绍一下invokable.invoke()。在这个方法里，用户代码会被真正执行，例如我们在map算子里写的new MapFunction()逻辑，最终将在这里被执行。这里介绍一下invokable，这是一个抽象类，提供了可以被TaskManager执行的对象的基本抽象。这个invoable是在解析JobGraph的时候生成相关信息的，并在此处形成真正可执行的对象。
 ```scala
 Task line 687
 
 	// now load and instantiate the task's invokable code
	invokable = loadAndInstantiateInvokable(userCodeClassLoader, nameOfInvokableClass, env);
 ```
下面具体列举了Flink可被执行的Task类型：
![Task]()

接下来就是invoke方法，我们主要介绍流式计算，所以我们以StreamTask的invoke方法为例进行说明

#### StreamTask的执行逻辑

```java
StreamTask line 239

	public final void invoke() throws Exception {

		boolean disposed = false;
		try {
			// -------- Initialize ---------
			LOG.debug("Initializing {}.", getName());

			asyncOperationsThreadPool = Executors.newCachedThreadPool();

			CheckpointExceptionHandlerFactory cpExceptionHandlerFactory = createCheckpointExceptionHandlerFactory();

			synchronousCheckpointExceptionHandler = cpExceptionHandlerFactory.createCheckpointExceptionHandler(
				getExecutionConfig().isFailTaskOnCheckpointError(),
				getEnvironment());

			asynchronousCheckpointExceptionHandler = new AsyncCheckpointExceptionHandler(this);

			stateBackend = createStateBackend();
			checkpointStorage = stateBackend.createCheckpointStorage(getEnvironment().getJobID());

			// if the clock is not already set, then assign a default TimeServiceProvider
			if (timerService == null) {
				ThreadFactory timerThreadFactory = new DispatcherThreadFactory(TRIGGER_THREAD_GROUP,
					"Time Trigger for " + getName(), getUserCodeClassLoader());

				timerService = new SystemProcessingTimeService(this, getCheckpointLock(), timerThreadFactory);
			}

			operatorChain = new OperatorChain<>(this, streamRecordWriters);
			headOperator = operatorChain.getHeadOperator();

			// task specific initialization
			init();

			// save the work of reloading state, etc, if the task is already canceled
			if (canceled) {
				throw new CancelTaskException();
			}

			// -------- Invoke --------
			LOG.debug("Invoking {}", getName());

			// we need to make sure that any triggers scheduled in open() cannot be
			// executed before all operators are opened
			synchronized (lock) {

				// both the following operations are protected by the lock
				// so that we avoid race conditions in the case that initializeState()
				// registers a timer, that fires before the open() is called.
				// 初始化操作算子的状态
				initializeState();
				// 执行rich算子的open()方法
				openAllOperators();
			}

			// final check to exit early before starting to run
			if (canceled) {
				throw new CancelTaskException();
			}

			// let the task do its work
			isRunning = true;
			// 真正开始执行代码
			run();

			// if this left the run() method cleanly despite the fact that this was canceled,
			// make sure the "clean shutdown" is not attempted
			if (canceled) {
				throw new CancelTaskException();
			}

			LOG.debug("Finished task {}", getName());

			// make sure no further checkpoint and notification actions happen.
			// we make sure that no other thread is currently in the locked scope before
			// we close the operators by trying to acquire the checkpoint scope lock
			// we also need to make sure that no triggers fire concurrently with the close logic
			// at the same time, this makes sure that during any "regular" exit where still
			synchronized (lock) {
				// this is part of the main logic, so if this fails, the task is considered failed
				closeAllOperators();

				// make sure no new timers can come
				timerService.quiesce();

				// only set the StreamTask to not running after all operators have been closed!
				// See FLINK-7430
				isRunning = false;
			}

			// make sure all timers finish
			timerService.awaitPendingAfterQuiesce();

			LOG.debug("Closed operators for task {}", getName());

			// make sure all buffered data is flushed
			operatorChain.flushOutputs();

			// make an attempt to dispose the operators such that failures in the dispose call
			// still let the computation fail
			tryDisposeAllOperators();
			disposed = true;
		}
		...
	}
```

StreamTask.invoke()代码主体逻辑分为以下几块：
  - 初始化算子的状态
  - 执行rich算子的open()方法
  - 最后执行run方法，调用用户编写的代码逻辑，例如当前task是SourceStreamTask类型(具体如数据源是env.socketTextStream())，经过一些的调用最后会调用SocketTextStreamFunction类的run()方法，建立socket连接并读入文本。
  
至此，从用户编写的流式用户到Flink真正执行用户代码逻辑的流程就完成了。  
