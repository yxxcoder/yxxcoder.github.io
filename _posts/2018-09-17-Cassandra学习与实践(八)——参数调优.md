---
layout: post
title:  "Cassandra学习与实践(八)——参数调优"
date:   2018-09-17 12:00:00 +0800
categories: nosql
---

#### *参考资料*

- *Cassandra官方文档 [http://cassandra.apache.org/doc/latest/faq/index.html](http://cassandra.apache.org/doc/latest/faq/index.html)*


<br>



## Linux优化

### 1. 安装 jemalloc

jemalloc是一个在内存分配上比linux自带的malloc(ptmalloc)要优秀的分配工具

Github地址：<https://github.com/jemalloc/jemalloc>

下载后安装：

```shell
tar xjf jemalloc-3.6.0.tar.bz2
cd jemalloc-3.6.0
./configure
make &&make install
echo '/usr/local/lib'>/etc/ld.so.conf.d/local.conf
ldconfig
```

<br>

### 2. 调整系统参数

/etc/sysctl.conf

```shell
# 调大mmap限制
vm.max_map_count = 1048575
#最大限度使用物理内存
vm.swappiness = 0
# 调整keepalive
net.ipv4.tcp_keepalive_time=60 
net.ipv4.tcp_keepalive_intvl=60 
net.ipv4.tcp_keepalive_probes=5
# 提升网络性能
net.core.rmem_max=16777216
net.core.wmem_max=16777216
net.core.rmem_default=16777216
net.core.wmem_default=16777216
net.core.optmem_max=40960
net.ipv4.tcp_rmem=4096 87380 16777216
net.ipv4.tcp_wmem=4096 65536 16777216
```

使之生效

```shell
sudo sysctl -p
# 关闭swap
sudo swapoff --all
```

若永久关闭swap：

```shell
vim /etc/fstab
# 注释掉swap分区项
```

<br>

分别使用root用户和cassandra用户执行如下命令：

```shell
# max locked-in-memory address space (KB)
ulimit -l unlimited
# max number of open files
ulimit -n 100000
# max number of processes
ulimit -u 32768
# address space limit (KB)
ulimit -v 32768
```

若永久保存，请将如下参数添加到/etc/security/limits.conf 文件：

```shell
<cassandra_user> - memlock unlimited
<cassandra_user> - nofile 100000
<cassandra_user> - nproc 32768
<cassandra_user> - as unlimited
```

<br>

## Cassandra参数调优

- memtable_allocation_type

  指定Cassandra分配和管理memtable内存的方式，可选参数有heap_buffers（堆nio缓冲区）、offheap_buffers（非堆nio缓冲区）、offheap_objects（非堆对象）

- concurrent_reads


- concurrent_writes

  对于具有比内存可容纳数据更多的数据的工作负载，Cassandra的瓶颈将是需要从磁盘提取数据的读取。`concurrent_reads`应该设置为（16 * number_of_drives），以便降低堆栈中排队的操作，操作系统和驱动器可以对它们重新排序。这同样适用于`concurrent_counter_writes`，因为计数器写入在递增和写回它们之前读取当前值。 另一方面，由于写入几乎没有IO限制，`concurrent_writes`的理想数量取决于系统中的核心数;（8 * number_of_cores）是一个好的经验法则


- concurrent_compactors

  默认情况下，此选项被注释掉。 允许同时压缩的数量，不包括用于反熵修复的验证“压缩”。 同时压缩可以通过减少在单个长时间运行压缩期间小的sstables累积的趋势来帮助保持混合读/写工作负载中的读取性能。 所设置的默认值通常可以使压缩的性能很好，如果你遇到压缩运行太慢或太快的问题，你应该首先查看compaction_throughput_mb_per_sec。 concurrent_compactors默认为（磁盘数目，核心数目）中的较小值，最小值为2，最大值为8。 如果数据目录由SSD支持，则应将其增加到核心数。 默认值: 1


- memtable_flush_writers

  默认情况下，此选项被注释掉。 这将设置每个磁盘的memtable flush写线程数以及可以同时刷新的memtables的总数。这些通常是计算和IO绑定的组合。 memtable刷新比memtable提取在CPU上更加高效，单个线程可以跟上单个快速磁盘上整个服务器的吞吐率，直到它在通常使用压缩的争用中暂时变为IO绑定。这时你需要多个刷新线程。在将来的某个时候，它可能成为CPU限制所有的时间。  你可以使用MemtablePool.BlockedOnAllocation指标来判断刷新是否落后，该指标应该为0，但如果线程被阻塞等待刷新以释放内存，则该值将为非零。 对于单个数据目录，memtable_flush_writers默认为两个。这意味着两个memtables可以同时刷新到单个数据目录。如果你有多个数据目录，默认是一次刷新一个memtable，但每个数据目录刷新将使用一个线程，所以你会得到两个或更多的写入。 两个通常足以在作为单个数据目录安装的快速磁盘上刷新。添加更多刷新写入将导致更小的更频繁的刷新，从而引入更多的压缩开销。 在可以同时刷新的memtables数量、flush大小和频率之间有一个直接的权衡。更多并不一定更好，你只需要足够的flush写入保证不出现程序停止，等待刷新释放内存。 默认值: 2



