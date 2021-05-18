About
=======

This project covers basics of batch data processing on the dockerized Hadoop cluster.

Simple MapReduce app is the classic one (collects statistics about words in input file)


Cluster installation and running
=================================

Build cluster using docker compose file from /hadoop-cluster/docker-compose.yml

```
docker-compose -f docker-compose.yml up -d
```

Make sure all containers are up and running:

```
docker container list

CONTAINER ID   IMAGE                                                    COMMAND                  CREATED         STATUS                   PORTS                                            NAMES
818989699cf1   bde2020/hadoop-namenode:2.0.0-hadoop3.2.1-java8          "/entrypoint.sh /run…"   2 minutes ago   Up 2 minutes (healthy)   0.0.0.0:9000->9000/tcp, 0.0.0.0:9870->9870/tcp   namenode
06865eccae10   bde2020/hadoop-datanode:2.0.0-hadoop3.2.1-java8          "/entrypoint.sh /run…"   2 minutes ago   Up 2 minutes (healthy)   9864/tcp                                         datanode
85998aa7c898   bde2020/hadoop-nodemanager:2.0.0-hadoop3.2.1-java8       "/entrypoint.sh /run…"   2 minutes ago   Up 2 minutes (healthy)   8042/tcp                                         nodemanager
a4a1c1cba4e7   bde2020/hadoop-historyserver:2.0.0-hadoop3.2.1-java8     "/entrypoint.sh /run…"   2 minutes ago   Up 2 minutes (healthy)   8188/tcp                                         historyserver
1f6373828e94   bde2020/hadoop-resourcemanager:2.0.0-hadoop3.2.1-java8   "/entrypoint.sh /run…"   2 minutes ago   Up 2 minutes (healthy)   8088/tcp                                         resourcemanager
```

Go to the name node UI to see the status of hadoop cluster:

```
http://localhost:9870/dfshealth.html#tab-overview
```
Containers can be stopped using the following command

```
docker-compose -f docker-compose.yml down
```

Running MapReduce app on the cluster
=====================================

Build the project:

```
mvn clean package
```

First of all, copy input and compiled jar file to namenode container (which is playing a role of orchestrator this time)

```
docker cp ./input/sonet104.txt namenode:/tmp
docker cp ./target/hadoop-wordcounter-1.0.jar namenode:/tmp
```

Connect a bash session to the namenode container:

```
docker exec -it namenode /bin/bash

root@a6af3563e823:/# ls /tmp -sh
total 76K
4.0K hadoop-root-namenode.pid    4.0K hsperfdata_root                                        4.0K sonet104.txt
 60K hadoop-wordcounter-1.0.jar  4.0K jetty-0.0.0.0-9870-hdfs-_-any-7028729040041975610.dir
```

Copy all necessary files to datanode(s):

```
# HDFS list commands to show all the directories in root "/"
hdfs dfs -ls /

# Create a new directory inside HDFS using mkdir tag.
hdfs dfs -mkdir -p /user/root

# Copy the files to the input path in HDFS.
hdfs dfs -put /tmp/sonet104.txt /user/root/sonet104.txt

# Take a look at the content of your input file.
hdfs dfs -cat /user/root/sonet104.txt
```

The 3rd command will result in the following output showing that Hadoop used network operations to transfer data to the datanode:

```
2021-05-17 10:56:30,757 INFO sasl.SaslDataTransferClient: SASL encryption trust check: localHostTrusted = false, remoteHostTrusted = false
```

Finally run the MapReduce app as a hadoop job:

```
# Run map reduce job from the path where you have the jar file.
hadoop jar /tmp/hadoop-wordcounter-1.0.jar hadoop.mr.wordcount.WordCount /user/root/sonet104.txt /user/root/sonet104-output
```
 

If run was successful, the output should be similiar to this one:

```
39,322 INFO client.RMProxy: Connecting to ResourceManager at resourcemanager/172.20.0.3:8032
39,471 INFO client.AHSProxy: Connecting to Application History server at historyserver/172.20.0.2:10200
39,498 INFO client.RMProxy: Connecting to ResourceManager at resourcemanager/172.20.0.3:8032
39,499 INFO client.AHSProxy: Connecting to Application History server at historyserver/172.20.0.2:10200
39,644 WARN mapreduce.JobResourceUploader: Hadoop command-line option parsing not performed. Implement the Tool interface and execute your application with ToolRunner to remedy this.
39,660 INFO mapreduce.JobResourceUploader: Disabling Erasure Coding for path: /tmp/hadoop-yarn/staging/root/.staging/job_1621246640599_0001
39,755 INFO sasl.SaslDataTransferClient: SASL encryption trust check: localHostTrusted = false, remoteHostTrusted = false
39,847 INFO mapred.FileInputFormat: Total input files to process : 1
39,877 INFO sasl.SaslDataTransferClient: SASL encryption trust check: localHostTrusted = false, remoteHostTrusted = false
40,303 INFO sasl.SaslDataTransferClient: SASL encryption trust check: localHostTrusted = false, remoteHostTrusted = false
40,311 INFO mapreduce.JobSubmitter: number of splits:2
40,416 INFO sasl.SaslDataTransferClient: SASL encryption trust check: localHostTrusted = false, remoteHostTrusted = false
40,434 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1621246640599_0001
40,434 INFO mapreduce.JobSubmitter: Executing with tokens: []
40,609 INFO conf.Configuration: resource-types.xml not found
40,609 INFO resource.ResourceUtils: Unable to find 'resource-types.xml'.
41,043 INFO impl.YarnClientImpl: Submitted application application_1621246640599_0001
41,098 INFO mapreduce.Job: The url to track the job: http://resourcemanager:8088/proxy/application_1621246640599_0001/
41,101 INFO mapreduce.Job: Running job: job_1621246640599_0001
47,198 INFO mapreduce.Job: Job job_1621246640599_0001 running in uber mode : false
47,199 INFO mapreduce.Job:  map 0% reduce 0%
52,251 INFO mapreduce.Job:  map 50% reduce 0%
53,258 INFO mapreduce.Job:  map 100% reduce 0%
57,290 INFO mapreduce.Job:  map 100% reduce 100%
57,298 INFO mapreduce.Job: Job job_1621246640599_0001 completed successfully
57,380 INFO mapreduce.Job: Counters: 54
        File System Counters
                FILE: Number of bytes read=523
                FILE: Number of bytes written=689698
                Peak Reduce Virtual memory (bytes)=8452407296
                
                ...
                
        Shuffle Errors
                BAD_ID=0
                CONNECTION=0
                IO_ERROR=0
                WRONG_LENGTH=0
                WRONG_MAP=0
                WRONG_REDUCE=0
        File Input Format Counters
                Bytes Read=983
        File Output Format Counters
                Bytes Written=738
```

Check the output file created by app:

```
 hdfs dfs -ls -h /user/root/sonet104-output
 
-rw-r--r--   3 root supergroup        738 2021-05-17 11:02 /user/root/sonet104-output/part-00000

hdfs dfs -cat /user/root/sonet104-output/part-00000

2021-05-17 11:09:57,462 INFO sasl.SaslDataTransferClient: SASL encryption trust check: localHostTrusted = false, remoteHostTrusted = false

Ah,     1
April   1
Ere     1
For     2
Hath    1
...
```

Check the status of completed jobs:

```
mapred job -list all

51,194 INFO client.RMProxy: Connecting to ResourceManager at resourcemanager/172.21.0.4:8032
51,352 INFO client.AHSProxy: Connecting to Application History server at historyserver/172.21.0.6:10200
51,827 INFO conf.Configuration: resource-types.xml not found
51,827 INFO resource.ResourceUtils: Unable to find 'resource-types.xml'.

Total jobs:2
JobId       JobName     State         StartTime      UserName           Queue      Priority       UsedContainers   RsvdContainers  UsedMem   RsvdMem   NeededMem         AM info
job_1621246640599_0001            wordcount     SUCCEEDED       1621249360737          root         default       DEFAULT     N/A              N/A      N/A             N/A               N/A      http://resourcemanager:8088/proxy/application_1621246640599_0001/
 job_1621254046819_0001            wordcount     SUCCEEDED       1621254310556          root         default       DEFAULT    N/A              N/A      N/A             N/A               N/A      http://resourcemanager:8088/proxy/application_1621254046819_0001/
```

