---
title: 寻找 Spark executor 日志
date: 2018-10-24 22:21:28
tags:
  - Spark
  - BigData
---
## 引言

spark on yarn 应用在运行时和完成后日志的存放位置是不同的，一般运行时是存放在各个运行节点，完成后会归集到 hdfs。无论哪种情况，都可以通过 spark 的页面跳转找到 executor 的日志，但是在大多数的生产环境中，对端口的开放是有严格的限制，也就是说根本无法正常跳转到日志页面进行查看的，这种情况下，就需要通过后台查询。

## 运行时

spark on yarn 模式下一个 executor 对应 yarn 的一个 container，所以在 executor 的节点运行`ps -ef|grep spark.yarn.app.container.log.dir`，如果这个节点上可能运行多个 application，那么再通过 application id 进一步过滤。上面的命令会查到 executor 的进程信息，并且包含了日志路径，例如

```
 -Djava.io.tmpdir=/data1/hadoop/yarn/local/usercache/ocdp/appcache/application_1521424748238_0051/container_e07_1521424748238_0051_01_000002/tmp '
-Dspark.history.ui.port=18080' '-Dspark.driver.port=59555' 
-Dspark.yarn.app.container.log.dir=/data1/hadoop/yarn/log/application_1521424748238_0051/container_e07_1521424748238_0051_01_000002 

```

也就是说这个 executor 的日志就在`/data1/hadoop/yarn/log/application_1521424748238_0051/container_e07_1521424748238_0051_01_000002`目录里。至此，我们就找到了运行时的 executor 日志。

## 完成后

当这个 application 正常或者由于某种原因异常结束后，yarn 默认会将所有日志归集到 hdfs 上，所以 yarn 也提供了一个查询已结束 application 日志的方法，即
 `yarn logs -applicationId application_1521424748238_0057`，结果里面会包含所有 executor 的日志，可能会比较多，建议将结果重定向到一个文件再详细查看。

## 总结

无论对于 spark 应用程序的开发者还是运维人员，日志对于排查问题是至关重要的，所以本文介绍了找到日志的方法。

## 画外:log4j 配置 - spark on yarn client mode
spark streaming 的程序如果运行方式是 yarn **_client_** mode，那么如何指定 driver 和 executor 的 log4j 配置文件？

### Driver

添加参数`--driver-java-options`

```
 spark-submit --driver-java-options "-Dlog4j.configuration=file:/data1/conf/log4j-driver.properties"

```

### Executor

由于 executor 是运行在 yarn 的集群中的，所以先要将配置文件通过`--files`上传

```
spark-submit --files /data1/conf/log4j.properties --conf spark.executor.extraJavaOptions="-Dlog4j.configuration=log4j.properties"

```

在 log4j.properties 中要注意配置`spark.yarn.app.container.log.dir`例如

```
log4j.rootLogger=INFO, file
log4j.appender.file=org.apache.log4j.RollingFileAppender
log4j.appender.file.append=true
log4j.appender.file.file=${spark.yarn.app.container.log.dir}/stdout
log4j.appender.file.MaxFileSize=256MB
log4j.appender.file.MaxBackupIndex=20

log4j.appender.file.layout=org.apache.log4j.PatternLayout
log4j.appender.file.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss,SSS} %p [%t] %c{1}:%L - %m%n

# Settings to quiet third party logs that are too verbose
log4j.logger.org.spark-project.jetty=WARN
log4j.logger.org.spark-project.jetty.util.component.AbstractLifeCycle=ERROR
log4j.logger.org.apache.spark.repl.SparkIMain$exprTyper=INFO
log4j.logger.org.apache.spark.repl.SparkILoop$SparkILoopInterpreter=INFO

```

这样就可以在 spark 的 Web UI 中直接查看日志:executor tab 下的 Logs.

## 其他

如果是通过`java -cp`命令运行自己的 jar 包，可以通过下面的方式添加 log4j 的配置

```
java -cp -Dlog4j.configuration=file:${APP_HOME}/conf/log4j.properties
```

作者：Woople, 链接：https://www.jianshu.com/p/06a630618f19
