---
title: 提交 Spark 应用
date: 2018-10-26 16:21:56
hidden: true
tags:
  - Spark
  - BigData
---
提纲:如果用 shell 提交 Spark 应用, 需要指定 `HADOOP_CONF_DIR`, 然后 Spark 会帮你加到自己的 classpath 里去, 因为 Hadoop 的 Configuration 会从 classpath 中去找各种 site.xml. 如果你自己启动 spark 程序提交 Yarn 任务的话, 得自己的手动用代码加 classpath.

1. Spark 如何使用 `HADOOP_CONF_DIR`(加到 classpath)
2. Hadoop 如何加载 Configuration.