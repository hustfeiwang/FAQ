### Filesystem closed

<div id="FileSystemClosed"></div>
- 问题描述

```
    java.io.IOException: Filesystem closed
     at org.apache.hadoop.hdfs.DFSClient.checkOpen(DFSClient.java:808)
```
- 问题分析

FileSystem类有缓存策略，可以把filesystem实例进行缓存，如果打开了缓存策略。当任务提交到集群上面以后，多个datanode在getFileSystem过程中，由于conf一样，会得到同一个FileSystem。如果有一个datanode在使用完关闭连接，其它的datanode在访问就会出现上述异常。

因此解决方法就是将这个缓存策略关掉，这样就不会出现这种不一致的问题。在spark中参数spark.hadoop.fs.hdfs.impl.disable.cache设置为true，即为关掉缓存策略。

但是，这个参数，设置为true会不会有什么问题，比如hive有一个由这个参数设置为true造成的问题还需要细看下。

### Spark任务写入HDFS无反应

<div id="SLOWHDFSWRITE"></div>

- 问题描述

  Spark任务在写入HDFS时无响应，报错信息如下：

```
18/09/09 16:32:40 WARN DFSClient: DataStreamer Exception
org.apache.hadoop.ipc.RemoteException(org.apache.hadoop.hdfs.server.namenode.LeaseExpiredException): No lease on /user/**/_temporary/0/_temporary/attempt_20180909160941_0004_m_000003_0/part-00003-eeb017cb-3ba9-46e0-a1d0-b4d9f61ed7ac.snappy.parquet (inode 491680434): File does not exist. Holder DFSClient_NONMAPREDUCE_156371534_131 does not have any open files.
	at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.checkLease(FSNamesystem.java:3436)
	at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.analyzeFileState(FSNamesystem.java:3239)
	at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.getNewBlockTargets(FSNamesystem.java:3077)
	at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.getAdditionalBlock(FSNamesystem.java:3037)
	at org.apache.hadoop.hdfs.server.namenode.NameNodeRpcServer.addBlock(NameNodeRpcServer.java:725)
	at org.apache.hadoop.hdfs.protocolPB.ClientNamenodeProtocolServerSideTranslatorPB.addBlock(ClientNamenodeProtocolServerSideTranslatorPB.java:492)
	at org.apache.hadoop.hdfs.protocol.proto.ClientNamenodeProtocolProtos$ClientNamenodeProtocol$2.callBlockingMethod(ClientNamenodeProtocolProtos.java)
	at org.apache.hadoop.ipc.ProtobufRpcEngine$Server$ProtoBufRpcInvoker.call(ProtobufRpcEngine.java:616)
	at org.apache.hadoop.ipc.RPC$Server.call(RPC.java:982)
	at org.apache.hadoop.ipc.Server$Handler$1.run(Server.java:2049)
	at org.apache.hadoop.ipc.Server$Handler$1.run(Server.java:2045)
	at java.security.AccessController.doPrivileged(Native Method)
	at javax.security.auth.Subject.doAs(Subject.java:422)
	at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1698)
	at org.apache.hadoop.ipc.Server$Handler.run(Server.java:2043)

	at org.apache.hadoop.ipc.Client.call(Client.java:1475)
	at org.apache.hadoop.ipc.Client.call(Client.java:1412)
	at org.apache.hadoop.ipc.ProtobufRpcEngine$Invoker.invoke(ProtobufRpcEngine.java:229)
	at com.sun.proxy.$Proxy15.addBlock(Unknown Source)
	at org.apache.hadoop.hdfs.protocolPB.ClientNamenodeProtocolTranslatorPB.addBlock(ClientNamenodeProtocolTranslatorPB.java:418)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.apache.hadoop.io.retry.RetryInvocationHandler.invokeMethod(RetryInvocationHandler.java:191)
	at org.apache.hadoop.io.retry.RetryInvocationHandler.invoke(RetryInvocationHandler.java:102)
	at com.sun.proxy.$Proxy16.addBlock(Unknown Source)
	at org.apache.hadoop.hdfs.DFSOutputStream$DataStreamer.locateFollowingBlock(DFSOutputStream.java:1455)
	at org.apache.hadoop.hdfs.DFSOutputStream$DataStreamer.nextBlockOutputStream(DFSOutputStream.java:1251)
	at org.apache.hadoop.hdfs.DFSOutputStream$DataStreamer.run(DFSOutputStream.java:448)
18/09/09 16:32:40 ERROR DFSClient: Failed to close inode 491680434
```

- 问题分析

提示为文件操作超租期，由于多个task操作写一个文件，其中某个task完成任务后删除了临时文件引起。有两个解决方法，一种是修改spark代码减小写文件的并发度，另一种是修改hdfs参数配置，增大dfs.datanode.max.xcievers这个参数代表每个datanode任一时刻可以打开的文件数量上限。

parquet是一种列式数据存储格式，parquet在写入hdfs时要构建自己的数据结构，parquet结构内有更复杂的设计，会向同一个hdfs文件中写入大量的碎文件。

我给用户提的建议是使用coalesce或者reparation算子减小写文件并发度，看到用户的运行日志，上面显示，每个stage只有5个task，但是仍然运行缓慢。后续调研下parquet在写入hdfs时的合并优化。

### Failed to get database default, returning NoSuchObjectException

<div id="MetaStoreError"></div>
- 问题描述
```
18/09/19 15:58:11 WARN metastore.ObjectStore: Failed to get database default, returning NoSuchObjectException
18/09/19 15:58:12 WARN metastore.ObjectStore: Failed to get database global_temp, returning NoSuchObjectException
```
- 问题分析

  多半是hive配置问题，排查hive-site.xml配置是否正确

###  Spark任务executor在动态分配前提下没有释放

<div id="ExecutorNotRelease"></div>

- 问题描述

客户在开启了spark.dynamicAllocation.enabled 前提下,发现在某个stage里面，只有一个任务在运行，然而上百个已运行完毕的executor都没有释放

- 问题分析

  与spark.dynamicAllocation相关的参数有若干个。如下所示：

  [spark官方配置文档](https://spark.apache.org/docs/latest/configuration.html)



| 参数|默认值 |
| ---------------------------------------------------------- | -------------------------------------- |
| `spark.dynamicAllocation.enabled`                          | false                                  |
| `spark.dynamicAllocation.executorIdleTimeout`              | 60s                                    |
| `spark.dynamicAllocation.cachedExecutorIdleTimeout`        | infinity                               |
| `spark.dynamicAllocation.initialExecutors`                 | `spark.dynamicAllocation.minExecutors` |
| `spark.dynamicAllocation.maxExecutors`                     | infinity                               |
| `spark.dynamicAllocation.minExecutors`                     | 0                                      |
| `spark.dynamicAllocation.schedulerBacklogTimeout`          | 1s                                     |
| `spark.dynamicAllocation.sustainedSchedulerBacklogTimeout` | `schedulerBacklogTimeout`              |

spark.dynamicAllocation.cachedExecutorIdleTimeout 这个参数默认是无穷大的，也就是说如果一个executor有cache数据是不可能被动态回收的。因为用户将某个RDD cache到内存中，因此这些executor即使运行完成也不会被释放，可以根据具体使用情况设置spark.dynamicAllocation.cachedExecutorIdleTimeout的值，合理利用资源。