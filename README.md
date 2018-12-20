# 大数据调优
## 硬件调优
### 服务器是否处于省电模式下
TODO

## 操作系统调优
### 查看磁盘性能
#### 列出所有逻辑卷
fdisk -l
#### 查看所有逻辑卷的读写速度
在三台机器上执行一下命令：
```shell
$ dd if=/dev/vda bs=1M of=/dev/null count=1k
vda 250M/s
vdb 60M/s
```
#### hdfs数据目录所在分区
```shell 
$ df /hadoop -h
Filesystem               Size  Used Avail Use% Mounted on
/dev/mapper/centos-root   50G   21G   30G  42% /
```

```shell
df /data/hadoop/
Filesystem      1K-blocks     Used Available Use% Mounted on
/dev/vdb1      1056757172 96439960 906613788  10% /data
```
#### 问题
查询过程中观察到的负载在/dev/vda上，而/hadoop目录所在分区剩余磁盘空间只有30G，小于hadoop要求的保留磁盘空间，所以这个分区不会存储数据块。hive表中的数据应该都存储到/data/hadoop上，/dev/vdb1逻辑存储设备上。而实际观察到的负载在/dev/vda上

### 文件系统检查
#### 查看文件系统
```shell
$ parted /dev/sdb1
> print list
```
文件系统为ext4
#### 没有启用noatime
noatime，不更新文件系统目录的访问时间

### 网络速度检查

### 系统参数检查
永久修改内核参数：在/etc/sysctl.conf增加相应的配置，然后执行sysctl -p
+ somaxconn: socket监听的backlog上限，backlog就是socket的监听队列，当一个请求（request）尚未被处理或建立时，他会进入backlog。而socket server可以一次性处理backlog中的所有请求，处理后的请求不再位于监听队列中。当server处理请求较慢，以至于监听队列被填满后，新来的请求会被拒绝。
由于当前集群不是一个很忙的集群，以下参数可以接受
sysctl -a --pattern ".*somaxconn*"
net.core.somaxconn = 128

+ sysctl -w vm.swappiness=0

+ vm.dirty_background_ratio=20 
内存中可以填充脏数据的百分比
+ vm.dirty_ratio=50 
绝对脏数据百分比，如果内存中的脏数据超过这个百分比，那么新的IO请求会被阻塞

### 其他参数
+ MTU（需要所有服务器支持jumbo frames） 最大传输单元，TCP数据帧的大小，默认设置成1500,这里应该设置成9000
添加MTU=9000到文件 /etc/sysconfig/network-scripts/ifcfg-eth0，重启网络服务

+ 禁用透明大页压缩，这会导致cpu过载
echo never > /sys/kernel/mm/redhat_transparent_hugepages/defrag


### HDFS调优
+ 如果是一个很大的集群，du可能会导致一些过载
fs.du.interval=1200000

## 系统资源利用率，饱和度，错误
+ http://www.brendangregg.com/USEmethod/use-rosetta.html

## HDFS性能调优
### HADOOP Benchmark
TODO 整理资料
+ https://github.com/intel-hadoop/HiBench/blob/master/docs/build-hibench.md

### HTrace

### HDFS文件
#### HDFS文件的磁盘位置
``` shell
$ sudo -u hdfs hdfs fsck /warehouse/tablespace/managed/hive/orc_event/delta_0000002_0000002_0000 -files -locations -blocks

将得到如下输出：
Connecting to namenode via http://hdp1:50070/fsck?ugi=hdfs&files=1&locations=1&blocks=1&path=%2Fwarehouse%2Ftablespace%2Fmanaged%2Fhive%2Forc_event%2Fdelta_0000002_0000002_0000
FSCK started by hdfs (auth:SIMPLE) from /172.16.20.53 for path /warehouse/tablespace/managed/hive/orc_event/delta_0000002_0000002_0000 at Wed Dec 19 18:04:38 CST 2018
/warehouse/tablespace/managed/hive/orc_event/delta_0000002_0000002_0000 <dir>
/warehouse/tablespace/managed/hive/orc_event/delta_0000002_0000002_0000/_orc_acid_version 1 bytes, replicated: replication=3, 1 block(s**:  OK
0. BP-1544825953-172.16.20.53-1544669696881:**blk_1073742608_1784** len=1 Live_repl=3  [DatanodeInfoWithStorage[172.16.20.53:50010,DS-81ef060e-b753-45a0-951a-938dc933f018,DISK], DatanodeInfoWithStorage[172.16.20.54:50010,DS-1f319a93-4bc5-41e4-945b-625d7586a6ef,DISK], DatanodeInfoWithStorage[172.16.20.74:50010,DS-5f97d7a6-1217-48ef-8aaf-01c999d4c80c,DISK****
... 此处省略另外10个数据块的信息

**这说明数据块blk_1073742608_1784有3个在线副本，分别位于53, 54, 74机器上

Status: HEALTHY
 Number of data-nodes:	3
 Number of racks:		1
 Total dirs:			1
 Total symlinks:		0

** 这里描述的是hdfs里orc_event的小结，一共三个数据节点，一个机架，一个目录**

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
 
 **这里描述该目录下的数据快信息**

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
vmtouch current/BP-1544825953-172.16.20.53-1544669696881/current/finalized/subdir0/subdir3/blk_1073742608
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
dumpe2fs /dev/sda3 | grep -i "inode size"
```
[inode相关信息](https://www.ibm.com/developerworks/cn/aix/library/au-speakingunix14/)

## 参考资料
+ http://www.brendangregg.com/USEmethod/use-rosetta.html
+ http://crazyadmins.com/tune-hadoop-cluster-to-get-maximum-performance-part-1/
