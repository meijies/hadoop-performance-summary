## HDFS性能调优
### HDFS调优
+ 如果是一个很大的集群，du可能会导致一些过载
fs.du.interval=1200000
+ HDFS块大小： [134217728,1073741824)
+ 启用short-circuit读
想要启用short-circuit,需要先启用libhadoop.so (dfs.domain.socket.path)</br>
dfs.client.read.shortcircuit=true</br>
dfs.domain.socket.path=/var/lib/hadoop-hdfs/dn_socket
+ 避免过多小文件
  1) 运行hive和hbase压缩
  2) 合并小文件
  3) 使用HAR来压缩小文件  
+ 优化数据节点JVM配置，配置样例如下
  1) -Djava.net.preferIPv4Stack=true
  2) -XX:ParallelGCThreads=8
  3) -XX:+UseConcMarkSweepGC
  4) -Xloggc:*
  5) -verbose:gc
  6) -XX:+PrintGCDetails
  7) -XX:+PrintGCTimeStamps,
  8) -XX:+PrintGCDateStamps
  9) -Xms should be same as -Xmx
  10) 新生代应该是总JVM内存的1/8  
+ 避免读写不健康的节点 </br>
dfs.namenode.avoid.read.stale.datanode=true </br>
dfs.namenode.avoid.write.stale.datanode=true </br>
+ 使用JNI-based group lookup实现
hadoop.security.group.mapping=org.apache.hadoop.security.JniBasedUnixGroupsMappingWithFallback

### HADOOP Benchmark
TODO 整理资料
+ https://github.com/intel-hadoop/HiBench/blob/master/docs/build-hibench.md

### HTrace

### HDFS文件
#### HDFS文件的磁盘位置
``` shell
$ sudo -u hdfs hdfs fsck /warehouse/tablespace/managed/hive/orc_event/delta_0000002_0000002_0000 -files -locations -blocks
Connecting to namenode via http://hdp1:50070/fsck?
ugi=hdfs&files=1&locations=1&blocks=1&path=%2Fwarehouse%2Ftablespace%2Fmanaged%2Fhive%2Forc_event%
2Fdelta_0000002_0000002_0000FSCK started by hdfs (auth:SIMPLE) from /172.16.20.53 for path 
/warehouse/tablespace/managed/hive/orc_event/delta_0000002_0000002_0000 at Thu Dec 20 15:26:05 CST 2018
/warehouse/tablespace/managed/hive/orc_event/delta_0000002_0000002_0000 <dir>
/warehouse/tablespace/managed/hive/orc_event/delta_0000002_0000002_0000/_orc_acid_version 1 bytes, replicated: 
replication=3, 1 block(s):  OK
0. BP-1544825953-172.16.20.53-1544669696881:blk_1073742608_1784 len=1 Live_repl=3  
[DatanodeInfoWithStorage[172.16.20.53:50010,DS-81ef060e-b753-45a0-951a-938dc933f018,DISK], 
DatanodeInfoWithStorage[172.16.20.54:50010,DS-1f319a93-4bc5-41e4-945b-625d7586a6ef,DISK], 
DatanodeInfoWithStorage[172.16.20.74:50010,DS-5f97d7a6-1217-48ef-8aaf-01c999d4c80c,DISK]]

... 此处省略另外10个数据块的信息

这说明数据块blk_1073742608_1784有3个在线副本，分别位于53, 54, 74机器上

Status: HEALTHY
 Number of data-nodes:	3
 Number of racks:		1
 Total dirs:			1
 Total symlinks:		0

这里描述的是hdfs里orc_event的小结，一共三个数据节点，一个机架，一个目录
Replicated Blocks:
 Total size:	1621528981 B
 Total files:	11
 Total blocks (validated):	11 (avg. block size 147411725 B)
 Minimally replicated blocks:	11 (100.0 %)
 Over-replicated blocks:	0 (0.0 %)
 Under-replicated blocks:	0 (0.0 %)
 Mis-replicated blocks:		0 (0.0 %)
 Default replication factor:	3
 Average block replication:	3.0
 Missing blocks:		0
 Corrupt blocks:		0
 Missing replicas:		0 (0.0 %)
 
这里描述该目录下的数据快信息
Erasure Coded Block Groups:
 Total size:	0 B
 Total files:	0
 Total block groups (validated):	0
 Minimally erasure-coded block groups:	0
 Over-erasure-coded block groups:	0
 Under-erasure-coded block groups:	0
 Unsatisfactory placement block groups:	0
 Average block group size:	0.0
 Missing block groups:		0
 Corrupt block groups:		0
 Missing internal blocks:	0
FSCK ended at Wed Dec 19 18:04:38 CST 2018 in 1 milliseconds


The filesystem under path '/warehouse/tablespace/managed/hive/orc_event/delta_0000002_0000002_0000' is HEALTHY
```
#### HDFS文件的磁盘缓存
##### 安装vmtouch
```shell
$ git clone https://github.com/hoytech/vmtouch.git
$ cd vmtouch
$ make
$ sudo make install
```
##### 在本地磁盘上找到上面列出的数据块文件
以blk_1073742608_1784数据块为例
+ 查看hdfs的配置DateNode Directories的路径为: /data/hadoop/hdfs/data
+ 查看数据快文件位置
```shell
$ ls current/BP-*/current/finalized/subdir0/*/blk_1073742608*
current/BP-1544825953-172.16.20.53-1544669696881/current/finalized/subdir0/subdir3/blk_1073742608
current/BP-1544825953-172.16.20.53-1544669696881/current/finalized/subdir0/subdir3/blk_1073742608_1784.meta
```
##### 查看数据块文件的缓存情况
```shell
$ vmtouch current/BP-1544825953-172.16.20.53-1544669696881/current/finalized/subdir0/subdir3/blk_1073742608
           Files: 1
     Directories: 0
  Resident Pages: 0/1  0/4K  0%
         Elapsed: 0.000165 seconds
**Resident Pages表示内存驻留的页，当前为0**
```
###### vmtouch
+ 发现您的操作系统正在缓存哪些文件
+ 告诉操作系统缓存或驱逐某些文件或文件区域
+ 将文件锁定到内存中，以便操作系统不会驱逐它们
+ 在故障转移服务器时保留虚拟内存配置文件
+ 保持“热备用”文件服务器
+ 随时间绘制文件系统缓存使用情况
+ 维护缓存使用的“软配额”
+ 加快批处理/ cron作业
+ 以及更多...
[vmtouch手册](https://github.com/hoytech/vmtouch/blob/master/vmtouch.pod)

#### 查看磁盘INODE的大小
``` shell
$ dumpe2fs /dev/sda3 | grep -i "inode size"
```
[inode相关信息](https://www.ibm.com/developerworks/cn/aix/library/au-speakingunix14/)

