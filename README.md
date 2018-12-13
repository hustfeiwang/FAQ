# 说明

记录一些在大数据平台上出现问题的思考与分析

所以问题详细记录在[该页面](./qa/qa.md)，可以直接进入[该页面](./qa/qa.md)进行搜索关键词查询。

# 写在前面

在生产环境中使用Spark，了解参数配置很重要。

这是[spark官方配置文档](https://spark.apache.org/docs/latest/configuration.html)，建议熟练掌握参数配置，了解参数功能。

日志的查阅也很重要，讲解一下日志的分类。（我个人理解，有误请指出）

- 通过azkaban提交任务产生的日志，打印出的日志同用yarn-client模式提交后，打印到console的日志相似。这里面的日志没有什么参考价值。`请不要只看这里面的日志，一定要看下yarn对应application_ID的日志，才能得到有效信息`
- yarn中默认的日志是ApplicationMaster日志。（待后续确认）
- 开启yarn聚合日志操作之后可以把每个container日志在yarn显示，这里面包括executor所在container的日志，如果需要看具体container的日志，可以查阅。
- 如果未开启yarn聚合日志操作，需要到每个executor所在host查看其container日志。
- hadoop的jobhistory是与spark history结合，可以看到可视化的spark执行过程，可以点击stage，storage以及environment看详细执行情况，点击stage可以具体看到每个stage的运行时间，以及在stage运行中出现的错误。
- yarn的resourceManager日志可以看具体每个container的资源申请情况。
- yarn的nodemanager日志可以得到每个executor container在每个node上的资源分配管理情况。

### 常见问题

- 使用姿势有误

  > 用户使用猛犸在线提交还好，一些常见的配置不会出现问题，比如spark.yarn.access.namenodes的配置，还有其他域集群有关的参数配置。
  >
  > > 用户在使用自己的client提交任务时，特别是针对一些新手用户，往往会出现一些与集群相关参数的配置问题，这时候就需要查看日志，观察所有参数是否有问题。

- 参数配置不当，比如内存配置太小，各种超时时间配置太小，参考[spark官方配置文档](https://spark.apache.org/docs/latest/configuration.html)

- jar包冲突，比如有多个相同功能的不同版本jar包，用户没有及时清除无用jar包，导致类加载冲突，从而程序未执行到期望的程序代码，出现错误

- 应用程序编写问题，自查

# 问题列表

## [Spark任务写入HDFS无反应](./qa/qa.md/#SLOWHDFSWRITE)

## [Filesystem closed](./qa/qa.md/#FileSystemClosed)

## [Failed to get database default, returning NoSuchObjectException](./qa/qa.md/#MetaStoreError)

## [Spark任务executor在动态分配前提下没有释放](./qa/qa.md/#ExecutorNotRelease)

##  [Spark sql 参数查看](./qa/qa.md/#sqlconf)

## [driver连接超时](./qa/qa.md/#driver-timeout)

##[租约超期org.apache.hadoop.ipc.RemoteException(org.apache.hadoop.hdfs.server.namenode.LeaseExpiredException): No lease on](./qa/qa.md/#no-lease)

