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

### spark sql configuration

<div id="sqlconf"></div>

查看spark sql的参数

```
// spark is an existing SparkSession
spark.sql("SET -v").show(numRows = 200, truncate = false)
```

spark-2.3.1结果如下：

| key                                                      | defalut value                                                | description                                                  |
| -------------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| spark.sql.adaptive.enabled                               | false                                                        | When true, enable adaptive query execution.                  |
| spark.sql.adaptive.shuffle.targetPostShuffleInputSize    | 67108864b                                                    | The target post-shuffle input size in bytes of a task.       |
| spark.sql.autoBroadcastJoinThreshold                     | 10485760                                                     | Configures the maximum size in bytes for a table that will be broadcast to all worker nodes when performing a join.  By setting this value to -1 broadcasting can be disabled. Note that currently statistics are only supported for Hive Metastore tables where the command <code>ANALYZE TABLE &lt;tableName&gt; COMPUTE STATISTICS noscan</code> has been run, and file-based data source tables where the statistics are computed directly on the files of data. |
| spark.sql.broadcastTimeout                               | 300000ms                                                     | Timeout in seconds for the broadcast wait time in broadcast joins. |
| spark.sql.cbo.enabled                                    | false                                                        | Enables CBO for estimation of plan statistics when set true. |
| spark.sql.cbo.joinReorder.dp.star.filter                 | false                                                        | Applies star-join filter heuristics to cost based join enumeration. |
| spark.sql.cbo.joinReorder.dp.threshold                   | 12                                                           | The maximum number of joined nodes allowed in the dynamic programming algorithm. |
| spark.sql.cbo.joinReorder.enabled                        | false                                                        | Enables join reorder in CBO.                                 |
| spark.sql.cbo.starSchemaDetection                        | false                                                        | When true, it enables join reordering based on star schema detection. |
| spark.sql.columnNameOfCorruptRecord                      | _corrupt_record                                              | The name of internal column for storing raw/un-parsed JSON and CSV records that fail to parse. |
| spark.sql.crossJoin.enabled                              | false                                                        | When false, we will throw an error if a query contains a cartesian product without explicit CROSS JOIN syntax. |
| spark.sql.execution.arrow.enabled                        | false                                                        | When true, make use of Apache Arrow for columnar data transfers. Currently available for use with pyspark.sql.DataFrame.toPandas, and pyspark.sql.SparkSession.createDataFrame when its input is a Pandas DataFrame. The following data types are unsupported: BinaryType, MapType, ArrayType of TimestampType, and nested StructType. |
| spark.sql.execution.arrow.maxRecordsPerBatch             | 10000                                                        | When using Apache Arrow, limit the maximum number of records that can be written to a single ArrowRecordBatch in memory. If set to zero or negative there is no limit. |
| spark.sql.extensions                                     | <undefined>                                                  | Name of the class used to configure Spark Session extensions. The class should implement Function1[SparkSessionExtension, Unit], and must have a no-args constructor. |
| spark.sql.files.ignoreCorruptFiles                       | false                                                        | Whether to ignore corrupt files. If true, the Spark jobs will continue to run when encountering corrupted files and the contents that have been read will still be returned. |
| spark.sql.files.ignoreMissingFiles                       | false                                                        | Whether to ignore missing files. If true, the Spark jobs will continue to run when encountering missing files and the contents that have been read will still be returned. |
| spark.sql.files.maxPartitionBytes                        | 134217728                                                    | The maximum number of bytes to pack into a single partition when reading files. |
| spark.sql.files.maxRecordsPerFile                        | 0                                                            | Maximum number of records to write out to a single file. If this value is zero or negative, there is no limit. |
| spark.sql.function.concatBinaryAsString                  | false                                                        | When this option is set to false and all inputs are binary, `functions.concat` returns an output as binary. Otherwise, it returns as a string. |
| spark.sql.function.eltOutputAsString                     | false                                                        | When this option is set to false and all inputs are binary, `elt` returns an output as binary. Otherwise, it returns as a string. |
| spark.sql.groupByAliases                                 | true                                                         | When true, aliases in a select list can be used in group by clauses. When false, an analysis exception is thrown in the case. |
| spark.sql.groupByOrdinal                                 | true                                                         | When true, the ordinal numbers in group by clauses are treated as the position in the select list. When false, the ordinal numbers are ignored. |
| spark.sql.hive.caseSensitiveInferenceMode                | INFER_AND_SAVE                                               | Sets the action to take when a case-sensitive schema cannot be read from a Hive table's properties. Although Spark SQL itself is not case-sensitive, Hive compatible file formats such as Parquet are. Spark SQL must use a case-preserving schema when querying any table backed by files containing case-sensitive field names or queries may not return accurate results. Valid options include INFER_AND_SAVE (the default mode-- infer the case-sensitive schema from the underlying data files and write it back to the table properties), INFER_ONLY (infer the schema but don't attempt to write it to the table properties) and NEVER_INFER (fallback to using the case-insensitive metastore schema instead of inferring). |
| spark.sql.hive.convertMetastoreParquet                   | true                                                         | When set to true, the built-in Parquet reader and writer are used to process parquet tables created by using the HiveQL syntax, instead of Hive serde. |
| spark.sql.hive.convertMetastoreParquet.mergeSchema       | false                                                        | When true, also tries to merge possibly different but compatible Parquet schemas in different Parquet data files. This configuration is only effective when "spark.sql.hive.convertMetastoreParquet" is true. |
| spark.sql.hive.filesourcePartitionFileCacheSize          | 262144000                                                    | When nonzero, enable caching of partition file metadata in memory. All tables share a cache that can use up to specified num bytes for file metadata. This conf only has an effect when hive filesource partition management is enabled. |
| spark.sql.hive.manageFilesourcePartitions                | true                                                         | When true, enable metastore partition management for file source tables as well. This includes both datasource and converted Hive tables. When partition management is enabled, datasource tables store partition in the Hive metastore, and use the metastore to prune partitions during query planning. |
| spark.sql.hive.metastore.barrierPrefixes                 |                                                              | A comma separated list of class prefixes that should explicitly be reloaded for each version of Hive that Spark SQL is communicating with. For example, Hive UDFs that are declared in a prefix that typically would be shared (i.e. <code>org.apache.spark.*</code>). |
| spark.sql.hive.metastore.jars                            | builtin                                                      | Location of the jars that should be used to instantiate the HiveMetastoreClient. This property can be one of three options: " 1. "builtin" Use Hive 1.2.1, which is bundled with the Spark assembly when <code>-Phive</code> is enabled. When this option is chosen,<code>spark.sql.hive.metastore.version</code> must be either <code>1.2.1</code> or not defined. 2. "maven" Use Hive jars of specified version downloaded from Maven repositories. 3. A classpath in the standard format for both Hive and Hadoop. |
| spark.sql.hive.metastore.sharedPrefixes                  | com.mysql.jdbc,org.postgresql,com.microsoft.sqlserver,oracle.jdbc | A comma separated list of class prefixes that should be loaded using the classloader that is shared between Spark SQL and a specific version of Hive. An example of classes that should be shared is JDBC drivers that are needed to talk to the metastore. Other classes that need to be shared are those that interact with classes that are already shared. For example, custom appenders that are used by log4j. |
| spark.sql.hive.metastore.version                         | 1.2.1                                                        | Version of the Hive metastore. Available options are <code>0.12.0</code> through <code>2.1.1</code>. |
| spark.sql.hive.metastorePartitionPruning                 | true                                                         | When true, some predicates will be pushed down into the Hive metastore so that unmatching partitions can be eliminated earlier. This only affects Hive tables not converted to filesource relations (see HiveUtils.CONVERT_METASTORE_PARQUET and HiveUtils.CONVERT_METASTORE_ORC for more information). |
| spark.sql.hive.thriftServer.async                        | true                                                         | When set to true, Hive Thrift server executes SQL queries in an asynchronous way. |
| spark.sql.hive.thriftServer.singleSession                | false                                                        | When set to true, Hive Thrift server is running in a single session mode. All the JDBC/ODBC connections share the temporary views, function registries, SQL configuration and the current database. |
| spark.sql.hive.verifyPartitionPath                       | false                                                        | When true, check all the partition paths under the table's root directory when reading data stored in HDFS. |
| spark.sql.hive.version                                   | 1.2.1                                                        | deprecated, please use spark.sql.hive.metastore.version to get the Hive version in Spark. |
| spark.sql.inMemoryColumnarStorage.batchSize              | 10000                                                        | Controls the size of batches for columnar caching.  Larger batch sizes can improve memory utilization and compression, but risk OOMs when caching data. |
| spark.sql.inMemoryColumnarStorage.compressed             | true                                                         | When set to true Spark SQL will automatically select a compression codec for each column based on statistics of the data. |
| spark.sql.inMemoryColumnarStorage.enableVectorizedReader | true                                                         | Enables vectorized reader for columnar caching.              |
| spark.sql.optimizer.metadataOnly                         | true                                                         | When true, enable the metadata-only query optimization that use the table's metadata to produce the partition columns instead of table scans. It applies when all the columns scanned are partition columns and the query has an aggregate operator that satisfies distinct semantics. |
| spark.sql.orc.compression.codec                          | snappy                                                       | Sets the compression codec used when writing ORC files. If either `compression` or `orc.compress` is specified in the table-specific options/properties, the precedence would be `compression`, `orc.compress`, `spark.sql.orc.compression.codec`.Acceptable values include: none, uncompressed, snappy, zlib, lzo. |
| spark.sql.orc.enableVectorizedReader                     | true                                                         | Enables vectorized orc decoding.                             |
| spark.sql.orc.filterPushdown                             | false                                                        | When true, enable filter pushdown for ORC files.             |
| spark.sql.orderByOrdinal                                 | true                                                         | When true, the ordinal numbers are treated as the position in the select list. When false, the ordinal numbers in order/sort by clause are ignored. |
| spark.sql.parquet.binaryAsString                         | false                                                        | Some other Parquet-producing systems, in particular Impala and older versions of Spark SQL, do not differentiate between binary data and strings when writing out the Parquet schema. This flag tells Spark SQL to interpret binary data as a string to provide compatibility with these systems. |
| spark.sql.parquet.compression.codec                      | snappy                                                       | Sets the compression codec used when writing Parquet files. If either `compression` or `parquet.compression` is specified in the table-specific options/properties, the precedence would be `compression`, `parquet.compression`, `spark.sql.parquet.compression.codec`. Acceptable values include: none, uncompressed, snappy, gzip, lzo. |
| spark.sql.parquet.enableVectorizedReader                 | true                                                         | Enables vectorized parquet decoding.                         |
| spark.sql.parquet.filterPushdown                         | true                                                         | Enables Parquet filter push-down optimization when set to true. |
| spark.sql.parquet.int64AsTimestampMillis                 | false                                                        | (Deprecated since Spark 2.3, please set spark.sql.parquet.outputTimestampType.) When true, timestamp values will be stored as INT64 with TIMESTAMP_MILLIS as the extended type. In this mode, the microsecond portion of the timestamp value will betruncated. |
| spark.sql.parquet.int96AsTimestamp                       | true                                                         | Some Parquet-producing systems, in particular Impala, store Timestamp into INT96. Spark would also store Timestamp as INT96 because we need to avoid precision lost of the nanoseconds field. This flag tells Spark SQL to interpret INT96 data as a timestamp to provide compatibility with these systems. |
| spark.sql.parquet.int96TimestampConversion               | false                                                        | This controls whether timestamp adjustments should be applied to INT96 data when converting to timestamps, for data written by Impala.  This is necessary because Impala stores INT96 data with a different timezone offset than Hive & Spark. |
| spark.sql.parquet.mergeSchema                            | false                                                        | When true, the Parquet data source merges schemas collected from all data files, otherwise the schema is picked from the summary file or a random data file if no summary file is available. |
| spark.sql.parquet.outputTimestampType                    | INT96                                                        | Sets which Parquet timestamp type to use when Spark writes data to Parquet files. INT96 is a non-standard but commonly used timestamp type in Parquet. TIMESTAMP_MICROS is a standard timestamp type in Parquet, which stores number of microseconds from the Unix epoch. TIMESTAMP_MILLIS is also standard, but with millisecond precision, which means Spark has to truncate the microsecond portion of its timestamp value. |
| spark.sql.parquet.recordLevelFilter.enabled              | false                                                        | If true, enables Parquet's native record-level filtering using the pushed down filters. This configuration only has an effect when 'spark.sql.parquet.filterPushdown' is enabled. |
| spark.sql.parquet.respectSummaryFiles                    | false                                                        | When true, we make assumption that all part-files of Parquet are consistent with summary files and we will ignore them when merging schema. Otherwise, if this is false, which is the default, we will merge all part-files. This should be considered as expert-only option, and shouldn't be enabled before knowing what it means exactly. |
| spark.sql.parquet.writeLegacyFormat                      | false                                                        | Whether to be compatible with the legacy Parquet format adopted by Spark 1.4 and prior versions, when converting Parquet schema to Spark SQL schema and vice versa. |
| spark.sql.parser.quotedRegexColumnNames                  | false                                                        | When true, quoted Identifiers (using backticks) in SELECT statement are interpreted as regular expressions. |
| spark.sql.pivotMaxValues                                 | 10000                                                        | When doing a pivot without specifying values for the pivot column this is the maximum number of (distinct) values that will be collected without error. |
| spark.sql.queryExecutionListeners                        | <undefined>                                                  | List of class names implementing QueryExecutionListener that will be automatically added to newly created sessions. The classes should have either a no-arg constructor, or a constructor that expects a SparkConf argument. |
| spark.sql.redaction.options.regex                        | (?i)url                                                      | Regex to decide which keys in a Spark SQL command's options map contain sensitive information. The values of options whose names that match this regex will be redacted in the explain output. This redaction is applied on top of the global redaction configuration defined by spark.redaction.regex. |
| spark.sql.redaction.string.regex                         | <value of spark.redaction.string.regex>                      | Regex to decide which parts of strings produced by Spark contain sensitive information. When this regex matches a string part, that string part is replaced by a dummy value. This is currently used to redact the output of SQL explain commands. When this conf is not set, the value from `spark.redaction.string.regex` is used. |
| spark.sql.session.timeZone                               | Asia/Shanghai                                                | The ID of session local timezone, e.g. "GMT", "America/Los_Angeles", etc. |
| spark.sql.shuffle.partitions                             | 200                                                          | The default number of partitions to use when shuffling data for joins or aggregations. |
| spark.sql.sources.bucketing.enabled                      | true                                                         | When false, we will treat bucketed table as normal table     |
| spark.sql.sources.default                                | parquet                                                      | The default data source to use in input/output.              |
| spark.sql.sources.parallelPartitionDiscovery.threshold   | 32                                                           | The maximum number of paths allowed for listing files at driver side. If the number of detected paths exceeds this value during partition discovery, it tries to list the files with another Spark distributed job. This applies to Parquet, ORC, CSV, JSON and LibSVM data sources. |
| spark.sql.sources.partitionColumnTypeInference.enabled   | true                                                         | When true, automatically infer the data types for partitioned columns. |
| spark.sql.sources.partitionOverwriteMode                 | STATIC                                                       | When INSERT OVERWRITE a partitioned data source table, we currently support 2 modes: static and dynamic. In static mode, Spark deletes all the partitions that match the partition specification(e.g. PARTITION(a=1,b)) in the INSERT statement, before overwriting. In dynamic mode, Spark doesn't delete partitions ahead, and only overwrite those partitions that have data written into it at runtime. By default we use static mode to keep the same behavior of Spark prior to 2.3. Note that this config doesn't affect Hive serde tables, as they are always overwritten with dynamic mode. |
| spark.sql.statistics.fallBackToHdfs                      | false                                                        | If the table statistics are not available from table metadata enable fall back to hdfs. This is useful in determining if a table is small enough to use auto broadcast joins. |
| spark.sql.statistics.histogram.enabled                   | false                                                        | Generates histograms when computing column statistics if enabled. Histograms can provide better estimation accuracy. Currently, Spark only supports equi-height histogram. Note that collecting histograms takes extra cost. For example, collecting column statistics usually takes only one table scan, but generating equi-height histogram will cause an extra table scan. |
| spark.sql.statistics.size.autoUpdate.enabled             | false                                                        | Enables automatic update for table size once table's data is changed. Note that if the total number of files of the table is very large, this can be expensive and slow down data change commands. |
| spark.sql.streaming.checkpointLocation                   | <undefined>                                                  | The default location for storing checkpoint data for streaming queries. |
| spark.sql.streaming.metricsEnabled                       | false                                                        | Whether Dropwizard/Codahale metrics will be reported for active streaming queries. |
| spark.sql.streaming.numRecentProgressUpdates             | 100                                                          | The number of progress updates to retain for a streaming query |
| spark.sql.thriftserver.scheduler.pool                    | <undefined>                                                  | Set a Fair Scheduler pool for a JDBC client session.         |
| spark.sql.thriftserver.ui.retainedSessions               | 200                                                          | The number of SQL client sessions kept in the JDBC/ODBC web UI history. |
| spark.sql.thriftserver.ui.retainedStatements             | 200                                                          | The number of SQL statements kept in the JDBC/ODBC web UI history. |
| spark.sql.ui.retainedExecutions                          | 1000                                                         | Number of executions to retain in the Spark UI.              |
| spark.sql.variable.substitute                            | true                                                         | This enables substitution using syntax like ${var} ${system:var} and ${env:var}. |
| spark.sql.warehouse.dir                                  | file:/home/wangfei3/todo/spark-2.3.1/spark-warehouse         | The default location for managed databases and tables.       |

