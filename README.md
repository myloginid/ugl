# ugl
-XX:SurvivorRatio=8 -XX:+UseParNewGC  -XX:+UseConcMarkSweepGC -XX:+CMSClassUnloadingEnabled  -XX:+UseCMSCompactAtFullCollection  -XX:CMSFullGCsBeforeCompaction=0  -XX:+CMSParallelRemarkEnabled  -XX:+DisableExplicitGC -XX:+UseCMSInitiatingOccupancyOnly -XX:CMSInitiatingOccupancyFraction=60 -XX:SoftRefLRUPolicyMSPerMB=0 -XX:+PrintGCDetails  -XX:+PrintGCDateStamps -XX:+PrintTenuringDistribution -XX:+PrintAdaptiveSizePolicy -XX:+PrintReferenceGC -XX:+UseGCLogFileRotation  -XX:+PrintClassHistogramAfterFullGC -XX:+PrintClassHistogramBeforeFullGC -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=100M -Xloggc:/var/log/cloudera-scm-firehose/SERVICEMONITOR.gc.log


-Xloggc:/var/log/cloudera-scm-firehose/SERVICEMONITOR.gc.log -XX:+UseG1GC -XX:ParallelGCThreads=30 -XX:MaxGCPauseMillis=400 -XX:+ParallelRefProcEnabled -XX:-ResizePLAB -XX:+UnlockExperimentalVMOptions -verbose:gc -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps -XX:+PrintGCDetails -XX:+PrintAdaptiveSizePolicy -XX:+HeapDumpOnOutOfMemoryError -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=100m -XX:G1NewSizePercent=5 -XX:G1MaxNewSizePercent=5 -XX:G1MixedGCLiveThresholdPercent=85 -XX:G1HeapWastePercent=2 -XX:InitiatingHeapOccupancyPercent=65

https://github.com/patric-r/jvmtop/releases/download/0.8.0/jvmtop-0.8.0.tar.gz
