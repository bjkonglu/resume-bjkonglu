## Flink Command-line interface源码详解
> 文章主要从Flink的命令行接口入手，分析Flink整体解析命令行参数的过程

### Flink的提交任务脚本
我们可以在$FLINK_HOME/bin目录下找到Flink提交任务的脚本，即*flink*。*flink*这个脚本主要完成的功能如下：
  - 获取flink的配置文件
  - 构建flink的classpath
  - 配置flink任务的日志配置
  - 最后便是通过*java*命令启动主类*org.apache.flink.client.cli.CliFrontend*
当*CliFrontend*被运行时，程序就进如它的主函数(main(String[] args))，如下：
```java
    EnvironmentInformation.logEnvironmentInfo(LOG, "Command Line Client", args);

		// 1. find the configuration directory
		final String configurationDirectory = getConfigurationDirectoryFromEnv();

		// 2. load the global configuration
		final Configuration configuration = GlobalConfiguration.loadConfiguration(configurationDirectory);

		// 3. load the custom command lines
		final List<CustomCommandLine<?>> customCommandLines = loadCustomCommandLines(
			configuration,
			configurationDirectory);

		try {
			final CliFrontend cli = new CliFrontend(
				configuration,
				customCommandLines);

			SecurityUtils.install(new SecurityConfiguration(cli.configuration));
			int retCode = SecurityUtils.getInstalledContext()
					.runSecured(() -> cli.parseParameters(args));
			System.exit(retCode);
		}
		catch (Throwable t) {
			LOG.error("Fatal error while running command line interface.", t);
			t.printStackTrace();
			System.exit(31);
		}
```

简而言之，CliFrontend主要的功能是：
  - 打印用户通过脚本传入的参数
  - 获取flink的配置文件路径，并加载配置文件
  - 加载自定义的命令行接口FlinkYarnSessionCli和默认的命令行接口DefaultCLI，注意DefaultCLI必须添加到列表的最后面，因为在下面的逻辑会通过   getActiveCustomCommandLine(..)顺序获取处于活动状态的自定义命令行接口而DefaultCLI一直处于活动状态。
  - 用configuration和customCommandLines实例化CliFrontend
  - 用实例出来的CliFrontend来解析用户传入的参数

### CliFrontend解析用户传入的参数
首先获取参数数组的第一个，因为第一个是flink的动作参数，例如run、info、list、stop、cancel、savepoint和modify。针对不同的动作触发不同的逻辑，如下：
```java
	// do action
	switch (action) {
		case ACTION_RUN:
			run(params);
			return 0;
		case ACTION_LIST:
			list(params);
			return 0;
		case ACTION_INFO:
			info(params);
			return 0;
		case ACTION_CANCEL:
			cancel(params);
			return 0;
		case ACTION_STOP:
			stop(params);
			return 0;
		case ACTION_SAVEPOINT:
			savepoint(params);
			return 0;
		case ACTION_MODIFY:
			modify(params);
			return 0;
		case "-h":
		case "--help":
			CliFrontendParser.printHelp(customCommandLines);
			return 0;
		case "-v":
		case "--version":
			String version = EnvironmentInformation.getVersion();
			String commitID = EnvironmentInformation.getRevisionInformation().commitId;
			System.out.print("Version: " + version);
			System.out.println(commitID.equals(EnvironmentInformation.UNKNOWN) ? "" : ", Commit ID: " + commitID);
			return 0;
		default:
			System.out.printf("\"%s\" is not a valid action.\n", action);
			System.out.println();
			System.out.println("Valid actions are \"run\", \"list\", \"info\", \"savepoint\", \"stop\", or \"cancel\".");
			System.out.println();
			System.out.println("Specify the version option (-v or --version) to print Flink version.");
			System.out.println();
			System.out.println("Specify the help option (-h or --help) to get help on the command.");
			return 1;
	}
```

下面我们以运行任务的动作为例来接着探索，进入*run(String[] args)* 部分开始解析启动参数。Flink采用Apache Commons CLI来进行入参的解析，这里简单介绍Apache Commons CLI这个工具，它主要采用先注册后解析的思想，具体来说就是我们需要哪些参数，我们就将这些参数注册到Options容器，并且Option可以设定参数的必要性，然后遍历参数数组，并通过Options判断是不是我们需要的参数，详细了解可以参考[Apache Commons CLI](http://commons.apache.org/proper/commons-cli/)。

```java
   	1.
	final Options commandOptions = CliFrontendParser.getRunCommandOptions();

	final Options commandLineOptions = CliFrontendParser.mergeOptions(commandOptions, customCommandLineOptions);

	final CommandLine commandLine = CliFrontendParser.parse(commandLineOptions, args, true);

	final RunOptions runOptions = new RunOptions(commandLine);
	2.
	program = buildProgram(runOptions);
	3.
	final CustomCommandLine<?> customCommandLine = getActiveCustomCommandLine(commandLine);
	4.
	runProgram(customCommandLine, commandLine, runOptions, program);
```
主要的功能是：
  - 获取runCommand的选项，并将runCommandOptions选项和用户自定义的customCommandLineOptions进行合并
  - 然后采用Apache Commons CLI工具来进行解析获取解析后的参数CommandLine
  - 接着从JAR文件build program，即jarFile、classpath、entryPointClass以及args封装成PackagedProgram
  - 并获取处于活动状态的用户自定义命令行接口(FlinkYarnSessionCli)
  - 最后开始运行program
  
  ### 集群的部署/获取以及用户代码启动
  在上一小节，已经完成入参的解析进而我们获取到用户传入的各种参数以及flink的默认参数。在生产环境下，使用比较的模式是On YARN，所以下面开始介绍Flink On YARN的运行。在*runProgram(..)* 里面，首先创建集群的描述符，主要设定flinkJar、shipPath、queue、name、zookeeperNamespace等YARN的配置信息；然后，判断入参里面有没有指定集群地址，根据有没指定*yid*，分成几种情况来处理：
  - 没有指定yid并且处理detached模式，这种情况使用yarn session提交任务，一个任务一个集群，但是客户端提交完任务就不与任务就行交互了
  - 没有指定yid但不处理detached模式，这种情况使用yarn session提交任务，一个任务一个集群，（使用比较多）
  - 指定了yid，使用yarn session提交任务到指定集群里

这里针对使用场景比较多的情况进行分析，即第二种情况。如下：
```java
	1.
	// also in job mode we have to deploy a session cluster because the job
	// might consist of multiple parts (e.g. when using collect)
	final ClusterSpecification clusterSpecification = customCommandLine.getClusterSpecification(commandLine);
	client = clusterDescriptor.deploySessionCluster(clusterSpecification);
	// if not running in detached mode, add a shutdown hook to shut down cluster if client exits
	// there's a race-condition here if cli is killed before shutdown hook is installed
	if (!runOptions.getDetachedMode() && runOptions.isShutdownOnAttachedExit()) {
		shutdownHook = ShutdownHookUtil.addShutdownHook(client::shutDownCluster, client.getClass().getSimpleName(), LOG);
	} else {
		shutdownHook = null;
	}
	2.
	executeProgram(program, client, userParallelism);
```
从上述代码中，可以发现使用集群描述符开始部署Flink集群，并返回该Flink集群的客户端。并开始执行program。在*executeProgram(..)* 里面主要的逻辑就是通过Java放射机制调用用户主类，启动用户的代码。
```java
	// invoke main method
	prog.invokeInteractiveModeForExecution();
```
