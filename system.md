## 操作系统调优
### 文件系统分区
假定机器配置为：12块2T磁盘外加2块500G的磁盘，双8核处理器，125G内存
1)主节点

| 挂载点         | 磁盘                       | 描述                                                                                             |
| --             | --                         | --                                                                                               |
| /              | 以镜像方式使用两个500G磁盘 | 确保即使一个磁盘失败，操作系统仍然能够运行                                                       |
| /var/log       | 以镜像方式使用两个2T磁盘   | 完全希望管理员对这个策略有所支持，但只是为了防止日志的大小成为运行时关注的问题                   |
| /hadoop        | 以镜像方式使用两个2T磁盘   |                                                                                                  |
| /master/data/1 | 以镜像方式使用两个2T磁盘   | 主要被用于NameNode和JournalNode处理                                                              |
| /master/data/2 | 以镜像方式使用两个2T磁盘   | 同样被用于NameNode和JournalNode                                                                  |
| /master/zk     | 以镜像方式使用两个2T磁盘   | 随着越来越多的组件开始利用ZooKeeper，以及HBase与之通信的强度，允许ZK拥有自己的磁盘是非常有意义的 |

还剩下两个磁盘驱动器，用于在系统恢复方面提供灵活性。包括组件文件系统重新分配，适应系统故障，容忍额外的扩展。

2)工作节点
工作节与主节点有不同的恢复策略，应该倾向于最大存储而不是最小失败。

| 挂载点               | 磁盘           | 描述                 |
| --                   | --                     | --                   |
| /var/log             | 其中一块500G磁盘       |                      |
| /work/00 到 /work/12 | 单独挂在剩下的12快磁盘 | 策略核心就是一堆磁盘 |
| /                    | 其中一块500G磁盘       |                      |
| /hadoop              | 使用一块2T磁盘         |                      |

3)边缘节点
边缘和摄取节点主要确保有足够的空间存储瞬时数据。简而言之，他们是可替换的节点，他们所包含的数据应该能够从数据源重建

| 挂载点   | 磁盘                                           | 描述                     |
| --       | --                                             | --                       |
| /var/log | 其中一块500G磁盘                               |                          |
| /        | 其中一块500G磁盘                               |                          |
| /hadoop  | 一块2T磁盘                                     | 保持和其他节点一样的策略 |
| 用于指定 | 将一块或多块磁盘做成逻辑卷, 并提供磁盘失败保护 |                          |

### 文件系统挂载
+ ext4: "inode_readahead_blks=128","data=writeback","noatime","nodev"
+ xfs: "noatime"


### 查看磁盘性能
#### 列出所有逻辑卷
```shell
$ fdisk -l
Disk /dev/vda: 1099.5 GB, 1099511627776 bytes, 2147483648 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x000d1e52

   Device Boot      Start         End      Blocks   Id  System
/dev/vda1   *        2048     2099199     1048576   83  Linux
/dev/vda2         2099200  2147483647  1072692224   8e  Linux LVM

Disk /dev/vdb: 1099.5 GB, 1099511627776 bytes, 2147483648 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x85eb7157

   Device Boot      Start         End      Blocks   Id  System
/dev/vdb1            2048  2147483647  1073740800   83  Linux

Disk /dev/mapper/centos-root: 53.7 GB, 53687091200 bytes, 104857600 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/mapper/centos-swap: 4294 MB, 4294967296 bytes, 8388608 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/mapper/centos-home: 1040.5 GB, 1040451633152 bytes, 2032132096 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```
#### 查看所有逻辑卷的读写速度
在三台机器上执行一下命令：
```shell
$ dd if=/dev/vda bs=1M of=/dev/null count=1k
1024+0 records in
1024+0 records out
1073741824 bytes (1.1 GB) copied, 5.18163 s, 207 MB/s
```
```shell
$ dd if=/dev/vdb bs=1M of=/dev/null count=1k
1024+0 records in
1024+0 records out
1073741824 bytes (1.1 GB) copied, 11.4596 s, 93.7 MB/s
```
#### hdfs数据目录所在分区
```shell 
$ df /hadoop -h
Filesystem               Size  Used Avail Use% Mounted on
/dev/mapper/centos-root   50G   21G   30G  42% /
```

```shell
$ df /data/hadoop/
Filesystem      1K-blocks     Used Available Use% Mounted on
/dev/vdb1      1056757172 96439960 906613788  10% /data
```
#### 问题
查询过程中观察到的负载在/dev/vda上，而/hadoop目录所在分区剩余磁盘空间只有30G，小于hadoop要求的保留磁盘空间，所以这个分区不会存储数据块。hive表中的数据应该都存储到/data/hadoop上，/dev/vdb1逻辑存储设备上。而实际观察到的负载在/dev/vda上

### 文件系统检查
#### 查看文件系统
```shell
$ df -Th | grep "^/dev"
/dev/mapper/centos-root xfs        50G   22G   29G  44% /
/dev/vdb1               ext4     1008G   93G  864G  10% /data
/dev/vda1               xfs      1014M  142M  873M  14% /boot
/dev/mapper/centos-home xfs       969G   33M  969G   1% /home
```
#### 文件系统的选择
| 文件系统类型 | 说明                     |
| --           | --                       |
| ext4         |                          |
| ResierFS     | 针对小型文件具有最佳性能 |
| XFS          | 针对大型文件具有最佳性能 |

+ https://blog.csdn.net/liuzhoulong/article/details/41213253
+ http://blog.chinaunix.net/uid-9460004-id-3294714.html
+ https://blog.csdn.net/zhouchang3/article/details/52996100/
#### 没有启用noatime
noatime，不更新文件系统目录的访问时间

### 网络速度检查

### 系统参数检查
永久修改内核参数：在/etc/sysctl.conf增加相应的配置，然后执行sysctl -p
+ somaxconn: socket监听的backlog上限，backlog就是socket的监听队列，当一个请求（request）尚未被处理或建立时，他会进入backlog。而socket server可以一次性处理backlog中的所有请求，处理后的请求不再位于监听队列中。当server处理请求较慢，以至于监听队列被填满后，新来的请求会被拒绝。
由于当前集群不是一个很忙的集群，以下参数可以接受
sysctl -a --pattern ".*somaxconn*"
net.core.somaxconn = 128
+ sysctl -w vm.swappiness=1
+ vm.dirty_background_ratio=20 
内存中可以填充脏数据的百分比
+ vm.dirty_ratio=50 
绝对脏数据百分比，如果内存中的脏数据超过这个百分比，那么新的IO请求会被阻塞

### 其他参数
+ 为性能扩展配置CPU</br>
执行cpufreq-set -r -g performance </br>
编辑/sys/devices/system/cpu/cpu*/cpufreq/scaling_governor，并设置为performance </br>
+ MTU（需要所有服务器支持jumbo frames） 最大传输单元，TCP数据帧的大小，默认设置成1500,这里应该设置成9000 </br>
添加MTU=9000到文件 /etc/sysconfig/network-scripts/ifcfg-eth0，重启网络服务</br>
+ 禁用透明大页压缩，这会导致cpu过载</br>
echo never > /sys/kernel/mm/redhat_transparent_hugepages/defrag
+ 优化固态盘配置
```shell
echo 'deadline' > {{device}}/queue/scheduler ; 
echo '256' > {{device}}/queue/read_ahead_kb ; 
echo '256' > {{device}}/queue/nr_requests ; 
```
### 参考文章和书籍
+ https://martin.atlassian.net/wiki/spaces/lestermartin/blog/2015/09/02/45580306/hadoop+mount+points+more+art+than+science
