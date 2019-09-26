# ugl
-XX:SurvivorRatio=8 -XX:+UseParNewGC  -XX:+UseConcMarkSweepGC -XX:+CMSClassUnloadingEnabled  -XX:+UseCMSCompactAtFullCollection  -XX:CMSFullGCsBeforeCompaction=0  -XX:+CMSParallelRemarkEnabled  -XX:+DisableExplicitGC -XX:+UseCMSInitiatingOccupancyOnly -XX:CMSInitiatingOccupancyFraction=60 -XX:SoftRefLRUPolicyMSPerMB=0 -XX:+PrintGCDetails  -XX:+PrintGCDateStamps -XX:+PrintTenuringDistribution -XX:+PrintAdaptiveSizePolicy -XX:+PrintReferenceGC -XX:+UseGCLogFileRotation  -XX:+PrintClassHistogramAfterFullGC -XX:+PrintClassHistogramBeforeFullGC -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=100M -Xloggc:/var/log/cloudera-scm-firehose/SERVICEMONITOR.gc.log


-Xloggc:/var/log/cloudera-scm-firehose/SERVICEMONITOR.gc.log -XX:+UseG1GC -XX:ParallelGCThreads=30 -XX:MaxGCPauseMillis=400 -XX:+ParallelRefProcEnabled -XX:-ResizePLAB -XX:+UnlockExperimentalVMOptions -verbose:gc -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps -XX:+PrintGCDetails -XX:+PrintAdaptiveSizePolicy -XX:+HeapDumpOnOutOfMemoryError -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=100m -XX:G1NewSizePercent=5 -XX:G1MaxNewSizePercent=5 -XX:G1MixedGCLiveThresholdPercent=85 -XX:G1HeapWastePercent=2 -XX:InitiatingHeapOccupancyPercent=65

https://github.com/patric-r/jvmtop/releases/download/0.8.0/jvmtop-0.8.0.tar.gz

Metadata Management
This topic describes various knobs you can use to control how Impala manages its metadata in order to improve performance and scalability.

Parent topic: Scalability Considerations for Impala
On-demand Metadata
In previous versions of Impala, every coordinator kept a replica of all the cache in catalogd, consuming large memory on each coordinator with no option to evict. Metadata always propagated through the statestored and suffers from head-of-line blocking, for example, one user loading a big table blocking another user loading a small table.

With this new feature, the coordinators pull metadata as needed from catalogd and cache it locally. The cached metadata gets evicted automatically under memory pressure.

The granularity of on-demand metadata fetches is now at the partition level between the coordinator and catalogd. Common use cases like add/drop partitions do not trigger unnecessary serialization/deserialization of large metadata.

This feature is disabled by default.

The feature can be used in either of the following modes.
Metadata on-demand mode
In this mode, all coordinators use the metadata on-demand.
Set the following on catalogd:
--catalog_topic_mode=minimal
Set the following on all impalad coordinators:
--use_local_catalog=true
Mixed mode
In this mode, only some coordinators are enabled to use the metadata on-demand.
We recommend that you use the mixed mode only for testing local catalog’s impact on heap usage.
Set the following on catalogd:
--catalog_topic_mode=mixed
Set the following on impalad coordinators with metdadata on-demand:
--use_local_catalog=true 
Limitation:

Global INVALIDATES are not supported when this feature is enabled. If your workload requires global INVALIDATES, do not use this feature.

Automatic Invalidation of Metadata Cache
To keep the size of metadata bounded, catalogd periodically scans all the tables and invalidates those not recently used. There are two types of configurations for catalogd and impalad.

Time-based cache invalidation
Catalogd invalidates tables that are not recently used in the specified time period (in seconds).
The ‑‑invalidate_tables_timeout_s flag needs to be applied to both impalad and catalogd.
Memory-based cache invalidation
When the memory pressure reaches 60% of JVM heap size after a Java garbage collection in catalogd, Impala invalidates 10% of the least recently used tables.
The ‑‑invalidate_tables_on_memory_pressure flag needs to be applied to both impalad and catalogd.
Automatic invalidation of metadata provides more stability with lower chances of running out of memory, but the feature could potentially cause performance issues and may require tuning.

Automatic Invalidation/Refresh of Metadata
When tools such as Hive and Spark are used to process the raw data ingested into Hive tables, new HMS metadata (database, tables, partitions) and filesystem metadata (new files in existing partitions/tables) is generated. In previous versions of Impala, in order to pick up this new information, Impala users needed to manually issue an INVALIDATE or REFRESH commands.

When automatic invalidate/refresh of metadata is enabled, catalogd polls Hive Metastore (HMS) notification events at a configurable interval and processes the following changes:

Note: This is a preview feature in Impala 3.3 and not generally available.
Invalidates the tables when it receives the ALTER TABLE event.
Refreshes the partition when it receives the ALTER, ADD, or DROP partitions.
Adds the tables or databases when it receives the CREATE TABLE or CREATE DATABASE events.
Removes the tables from catalogd when it receives the DROP TABLE or DROP DATABASE events.
Refreshes the table and partitions when it receives the INSERT events.
If the table is not loaded at the time of processing the INSERT event, the event processor does not need to refresh the table and skips it.

Changes the database and updates catalogd when it receives the ALTER DATABASE events. The following changes are supported. This event does not invalidate the tables in the database.
Change the database properties
Change the comment on the database
Change the owner of the database
Change the default location of the database
Changing the default location of the database does not move the tables of that database to the new location. Only the new tables which are created subsequently use the default location of the database in case it is not provided in the create table statement.

This feature is controlled by the ‑‑hms_event_polling_interval_s flag. Start the catalogd with the ‑‑hms_event_polling_interval_s flag set to a positive integer to enable the feature and set the polling frequency in seconds. We recommend the value to be less than 5 seconds.

The following use cases are not supported:

When you bypass HMS and add or remove data into table by adding files directly on the filesystem, HMS does not generate the INSERT event, and the event processor will not invalidate the corresponding table or refresh the corresponding partition.
It is recommended that you use the LOAD DATA command to do the data load in such cases, so that event processor can act on the events generated by the LOAD command.

The Spark API that saves data to a specified location does not generate events in HMS, thus is not supported. For example:
Seq((1, 2)).toDF("i", "j").write.save("/user/hive/warehouse/spark_etl.db/customers/date=01012019")
This feature is turned off by default with the ‑‑hms_event_polling_interval_s flag set to 0.

Configure HMS for Event Based Automatic Metadata Sync
To use the HMS event based metadata sync:

Add the following entries to the hive-site.xml of the Hive Metastore service.
 <property>
    <name>hive.metastore.transactional.event.listeners</name>
    <value>org.apache.hive.hcatalog.listener.DbNotificationListener</value>

    <name>hive.metastore.dml.events</name>
    <value>true</true>
  </property>
Save hive-site.xml.
Set the hive.metastore.dml.events configuration key to true in HiveServer2 service's hive-site.xml. This configuration key needs to be set to true in both Hive services, HiveServer2 and Hive Metastore.
If applicable, set the hive.metastore.dml.events configuration key to true in hive-site.xml used by the Spark applications (typically, /etc/hive/conf/hive-site.xml) so that the INSERT events are generated when the Spark application inserts data into existing tables and partitions.
Restart the HiveServer2, Hive Metastore, and Spark (if applicable) services.
Disable Event Based Automatic Metadata Sync
When the ‑‑hms_event_polling_interval_s flag is set to a non-zero value for your catalogd, the event-based automatic invalidation is enabled for all databases and tables. If you wish to have the fine-grained control on which tables or databases need to be synced using events, you can use the impala.disableHmsSync property to disable the event processing at the table or database level.

When you add the DBPROPERTIES or TBLPROPERTIES with the impala.disableHmsSync key, the HMS event based sync is turned on or off. The value of the impala.disableHmsSync property determines if the event processing needs to be disabled for a particular table or database.

If 'impala.disableHmsSync'='true', the events for that table or database are ignored and not synced with HMS.
If 'impala.disableHmsSync'='false' or if impala.disableHmsSync is not set, the automatic sync with HMS is enabled if the ‑‑hms_event_polling_interval_s global flag is set to non-zero.
To disable the event based HMS sync for a new database, set the impala.disableHmsSync database properties in Hive as currently, Impala does not support setting database properties:
CREATE DATABASE <name> WITH DBPROPERTIES ('impala.disableHmsSync'='true');
To enable or disable the event based HMS sync for a table:
CREATE TABLE <name> WITH TBLPROPERTIES ('impala.disableHmsSync'='true' | 'false');
To change the event based HMS sync at the table level:
ALTER TABLE <name> WITH TBLPROPERTIES ('impala.disableHmsSync'='true' | 'false');
When both table and database level properties are set, the table level property takes precedence. If the table level property is not set, then the database level property is used to evaluate if the event needs to be processed or not.

If the property is changed from true (meaning events are skipped) to false (meaning events are not skipped), you need to issue a manual INVALIDATE METADATA command to reset event processor because it doesn't know how many events have been skipped in the past and cannot know if the object in the event is the latest. In such a case, the status of the event processor changes to NEEDS_INVALIDATE.

Metrics for Event Based Automatic Metadata Sync
You can use the web UI of the catalogd to check the state of the automatic invalidate event processor.

Under the web UI, there are two pages that presents the metrics for HMS event processor that is responsible for the event based automatic metadata sync.
/metrics#events
/events
This provides a detailed view of the metrics of the event processor, including min, max, mean, median, of the durations and rate metrics for all the counters listed on the /metrics#events page.

/metrics#events Page
The /metrics#events page provides the following metrics about the HMS event processor.

Name	Description
events-processor.avg-events-fetch-duration	Average duration to fetch a batch of events and process it.
events-processor.avg-events-process-duration	Average time taken to process a batch of events received from the Metastore.
events-processor.events-received	Total number of the Metastore events received.
events-processor.events-received-15min-rate	Exponentially weighted moving average (EWMA) of number of events received in last 15 min.
This rate of events can be used to determine if there are spikes in event processor activity during certain hours of the day.

events-processor.events-received-1min-rate	Exponentially weighted moving average (EWMA) of number of events received in last 1 min.
This rate of events can be used to determine if there are spikes in event processor activity during certain hours of the day.

events-processor.events-received-5min-rate	Exponentially weighted moving average (EWMA) of number of events received in last 5 min.
This rate of events can be used to determine if there are spikes in event processor activity during certain hours of the day.

events-processor.events-skipped	Total number of the Metastore events skipped.
Events can be skipped based on certain flags are table and database level. You can use this metric to make decisions, such as:
If most of the events are being skipped, see if you might just turn off the event processing.
If most of the events are not skipped, see if you need to add flags on certain databases.
events-processor.status	Metastore event processor status to see if there are events being received or not. Possible states are:
PAUSED
The event processor is paused because catalog is being reset concurrently.

ACTIVE
The event processor is scheduled at a given frequency.

ERROR
The event processor is in error state and event processing has stopped.
NEEDS_INVALIDATE
The event processor could not resolve certain events and needs a manual INVALIDATE command to reset the state.

STOPPED
The event processing has been shutdown. No events will be processed.

DISABLED
The event processor is not configured to run.
