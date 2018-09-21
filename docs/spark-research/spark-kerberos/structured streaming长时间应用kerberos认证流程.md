# structured streaming长时间应用kerberos认证流程

## 背景
在开启Kerberos认证集群上运行Spark Streaming或者Structured Streaming这种长时间应用时，会遇到因为HADOOP_DELEGATION_TOKEN过期的问题而导致应用而异常退出。在Spark的官网上提供了一个解决方案如下图，总结来说就是在我们通过spark-submit脚本提交应用时，通过增加参数--principal和--keytab来解决这个问题。但是Spark内部是如何做到的呢？这就是下一小结的内容。
![screen shot 2018-09-21 at 11 15 56 am](https://user-images.githubusercontent.com/26513242/45858419-e123e880-bd8f-11e8-9040-b0975cbc5592.png)

## structured streaming on yarn 实现kerberos认证流程
这次就要介绍cluster模式下的kerberos认证流程。cluster模式下分为两部分的认证，一部分是ApplicationMaster(Driver)端的实现，另一部分是Executor端的实现。下面分别介绍

### ApplicationMaster(Driver)端的kerberos认证流程
当客户端提交应用后，会向YARN申请一个容器，并在容器里面使用command启动ApplicationMaster，如图所示：
```scala
ApplicationMaster
1.
def main(args: Array[String]): Unit = {
    SignalUtils.registerLogger(log)
    val amArgs = new ApplicationMasterArguments(args)
    master = new ApplicationMaster(amArgs)
    System.exit(master.run())
  }
2.
  final def run(): Int = {
    doAsUser {
      runImpl()
    }
    exitCode
  }
3.
  private def doAsUser[T](fn: => T): T = {
    ugi.doAs(new PrivilegedExceptionAction[T]() {
      override def run: T = fn
    })
  }
4.
  private val ugi = {
    val original = UserGroupInformation.getCurrentUser()

    // If a principal and keytab were provided, log in to kerberos, and set up a thread to
    // renew the kerberos ticket when needed. Because the UGI API does not expose the TTL
    // of the TGT, use a configuration to define how often to check that a relogin is necessary.
    // checkTGTAndReloginFromKeytab() is a no-op if the relogin is not yet needed.
    val principal = sparkConf.get(PRINCIPAL).orNull
    val keytab = sparkConf.get(KEYTAB).orNull

    if (principal != null && keytab != null) {

      //FIXME 使用principal和keytab文件登入kerberos
      UserGroupInformation.loginUserFromKeytab(principal, keytab)

      //FIXME 启动am-kerberos-renewer线程定时重新登入kerberos
      val renewer = new Thread() {
        override def run(): Unit = Utils.tryLogNonFatalError {
          while (true) {
            TimeUnit.SECONDS.sleep(sparkConf.get(KERBEROS_RELOGIN_PERIOD))
            UserGroupInformation.getCurrentUser().checkTGTAndReloginFromKeytab()
          }
        }
      }
      renewer.setName("am-kerberos-renewer")
      renewer.setDaemon(true)
      renewer.start()

      // Transfer the original user's tokens to the new user, since that's needed to connect to
      // YARN. It also copies over any delegation tokens that might have been created by the
      // client, which will then be transferred over when starting executors (until new ones
      // are created by the periodic task).
      val newUser = UserGroupInformation.getCurrentUser()
      SparkHadoopUtil.get.transferCredentials(original, newUser)
      newUser
    } else {
      SparkHadoopUtil.get.createSparkUser()
    }
  }
5.
  private def runImpl(): Unit = {
      val appAttemptId = client.getAttemptId()
      ...
      // If the credentials file config is present, we must periodically renew tokens. So create
      // a new AMDelegationTokenRenewer
      if (sparkConf.contains(CREDENTIALS_FILE_PATH)) {
        // Start a short-lived thread for AMCredentialRenewer, the only purpose is to set the
        // classloader so that main jar and secondary jars could be used by AMCredentialRenewer.
        val credentialRenewerThread = new Thread {
          setName("AMCredentialRenewerStarter")
          setContextClassLoader(userClassLoader)

          override def run(): Unit = {
            val credentialManager = new YARNHadoopDelegationTokenManager(
              sparkConf,
              yarnConf,
              conf => YarnSparkHadoopUtil.hadoopFSsToAccess(sparkConf, conf))
            val credentialRenewer =
              new AMCredentialRenewer(sparkConf, yarnConf, credentialManager)
            // FIXME 启动定期线程更新证书，并将新的证书写入HDFS
            credentialRenewer.scheduleLoginFromKeytab()
          }
        }
        credentialRenewerThread.start()
        credentialRenewerThread.join()
      }
       if (isClusterMode) {
          runDriver()
        } else {
          runExecutorLauncher()
        }
      ...
      }
```

说明：

1. 在步骤4里面，我们可以发现spark在ApplicationMaster代码里面启动一个名叫*am-kerberos-renewer*的线程周期性（默认1m）去重新去登入kerberos

1. 在步骤5里面，先运行用户的代码。如果spark配置里面包含参数*spark.yarn.credentials.file*，则会启动证书刷新器（AMCredentialRenewer）。并且会证书刷新器里面会启动定时（dfs.namenode.delegation.token.renew-interval值的75%）任务，定时将获取的证书写到HDFS（path->spark.yarn.stagingDir）下。至此，ApplicationMaster中kerberos认证的工作就完成了。

### Executor端kerberos认证流程
上个小节中，AM已经会周期性的登入kerberos，并会周期性得去更新刷新证书，将刷新后的证书会被持久化到HDFS上，然后Executor端会周期性得从该HDFS目录下拉取证书，如下：
```scala
CoarseGrainedExecutorBackend
1.
    if (driverConf.contains("spark.yarn.credentials.file")) {
        logInfo("Will periodically update credentials from: " +
          driverConf.get("spark.yarn.credentials.file"))
        //FIXME 在Executor上执行获取HDFS上token的动作
        Utils.classForName("org.apache.spark.deploy.yarn.YarnSparkHadoopUtil")
          .getMethod("startCredentialUpdater", classOf[SparkConf])
          .invoke(null, driverConf)
      }
YarnSparkHadoopUtil
2.
  def startCredentialUpdater(sparkConf: SparkConf): Unit = {
    val hadoopConf = SparkHadoopUtil.get.newConfiguration(sparkConf)
    val credentialManager = new YARNHadoopDelegationTokenManager(
      sparkConf,
      hadoopConf,
      conf => YarnSparkHadoopUtil.hadoopFSsToAccess(sparkConf, conf))
    credentialUpdater = new CredentialUpdater(sparkConf, hadoopConf, credentialManager)
    credentialUpdater.start()
  }
CredentialUpdater
3.
  private def updateCredentialsIfRequired(): Unit = {
    val timeToNextUpdate = try {
      val credentialsFilePath = new Path(credentialsFile)
      val remoteFs = FileSystem.get(freshHadoopConf)
      SparkHadoopUtil.get.listFilesSorted(
        remoteFs, credentialsFilePath.getParent,
        credentialsFilePath.getName, SparkHadoopUtil.SPARK_YARN_CREDS_TEMP_EXTENSION)
        .lastOption.map { credentialsStatus =>
          val suffix = SparkHadoopUtil.get.getSuffixForCredentialsPath(credentialsStatus.getPath)
          if (suffix > lastCredentialsFileSuffix) {
            logInfo("Reading new credentials from " + credentialsStatus.getPath)
            val newCredentials = getCredentialsFromHDFSFile(remoteFs, credentialsStatus.getPath)
            lastCredentialsFileSuffix = suffix
            UserGroupInformation.getCurrentUser.addCredentials(newCredentials)
            logInfo("Credentials updated from credentials file.")

            val remainingTime = (getTimeOfNextUpdateFromFileName(credentialsStatus.getPath)
              - System.currentTimeMillis())
            if (remainingTime <= 0) TimeUnit.MINUTES.toMillis(1) else remainingTime
          } else {
            // If current credential file is older than expected, sleep 1 hour and check again.
            TimeUnit.HOURS.toMillis(1)
          }
      }.getOrElse {
        // Wait for 1 minute to check again if there's no credential file currently
        TimeUnit.MINUTES.toMillis(1)
      }
    } catch {
      // Since the file may get deleted while we are reading it, catch the Exception and come
      // back in an hour to try again
      case NonFatal(e) =>
        logWarning("Error while trying to update credentials, will try again in 1 hour", e)
        TimeUnit.HOURS.toMillis(1)
    }

    logInfo(s"Scheduling credentials refresh from HDFS in $timeToNextUpdate ms.")
    credentialUpdater.schedule(
      credentialUpdaterRunnable, timeToNextUpdate, TimeUnit.MILLISECONDS)
  }
```
