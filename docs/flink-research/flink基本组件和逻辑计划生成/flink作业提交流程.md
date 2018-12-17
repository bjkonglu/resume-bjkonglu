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


