.. _jvm:

==========
JVM
==========

jvm.options (enable G1 / disable CMS)

Configure your JVM to use G1 garbage collector.  By default, jvm.options uses the older CMS garbage collector.  

.. Note::   Where is JVM configuration ?  < 3.0.0: cassandra-env.sh  &   >= 3.0.0: jvm.options.

.. code-block:: none

   #################
   #  GC SETTINGS  #
   #################
 
 
   ### CMS Settings
 
   -XX:+UseParNewGC
   -XX:+UseConcMarkSweepGC
   -XX:+CMSParallelRemarkEnabled
   -XX:SurvivorRatio=8
   -XX:MaxTenuringThreshold=1
   -XX:CMSInitiatingOccupancyFraction=75
   -XX:+UseCMSInitiatingOccupancyOnly
   -XX:CMSWaitDuration=10000
   -XX:+CMSParallelInitialMarkEnabled
   -XX:+CMSEdenChunksRecordAlways
   # some JVMs will fill up their heap when accessed via JMX, see CASSANDRA-6541
   -XX:+CMSClassUnloadingEnabled
 
   ### G1 Settings (experimental, comment previous section and uncomment section below to enable)

   ## Use the Hotspot garbage-first collector.
   #-XX:+UseG1GC
   #
   ## Have the JVM do less remembered set work during STW, instead
   ## preferring concurrent GC. Reduces p99.9 latency.
   #-XX:G1RSetUpdatingPauseTimePercent=5
   #
   ## Main G1GC tunable: lowering the pause target will lower throughput and vise versa.
   ## 200ms is the JVM default and lowest viable setting
   ## 1000ms increases throughput. Keep it smaller than the timeouts in cassandra.yaml.
   #-XX:MaxGCPauseMillis=500
   
   ## Optional G1 Settings
   # Save CPU time on large (>= 16GB) heaps by delaying region scanning
   # until the heap is 70% full. The default in Hotspot 8u40 is 40%.
   #-XX:InitiatingHeapOccupancyPercent=70

   # For systems with > 8 cores, the default ParallelGCThreads is 5/8 the number of logical cores.
   # Otherwise equal to the number of cores when 8 or less.
   # Machines with > 10 cores should try setting these to <= full cores.
   #-XX:ParallelGCThreads=16
   # By default, ConcGCThreads is 1/4 of ParallelGCThreads.
   # Setting both to the same value can reduce STW durations.
   #-XX:ConcGCThreads=16

   ### GC logging options -- uncomment to enable

   -XX:+PrintGCDetails
   -XX:+PrintGCDateStamps
   -XX:+PrintHeapAtGC
   -XX:+PrintTenuringDistribution
   -XX:+PrintGCApplicationStoppedTime
   -XX:+PrintPromotionFailure
   #-XX:PrintFLSStatistics=1
   #Xloggc:/var/log/cassandra/gc.log ← recommend you enable gc logging and rotate the logs
   -XX:+UseGCLogFileRotation
   -XX:NumberOfGCLogFiles=10
   -XX:GCLogFileSize=10M