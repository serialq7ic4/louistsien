---
{"dg-publish":true,"permalink":"/fio/"}
---

#基准测试

## 操作前须知(重要)

> **警告**
>
> 使用 FIO 作为测试工具，**请不要在系统盘上进行 FIO 测试，避免损坏系统重要文件**。
>
> 由于文件系统存在一定干扰，默认使用裸盘测试硬盘性能，**为避免底层文件系统元数据损坏导致数据损坏，请不要在业务数据盘上进行测试。**
> 
> 如果真的需要测试含有数据的数据盘，**请务必针对挂载点内的测试文件做测试，避免数据丢失**

## 关注指标


- IOPS：每秒读/写次数，单位为次(计数)，一般 SSD 硬盘看该指标
- 吞吐量(throughoutput)：每秒的读写数据量，单位为 MB/s，一般 HDD 硬盘看该指标
- 时延(latency)：I/O 操作的发送时间到接收确认所经过的时间，单位一般为 ms

> 有时候吞吐也被人称为带宽(bandwidth)，但其实带宽一般是用来形容网络传输速度的。不过大家习惯性的称呼存储带宽的时候其实清楚说的是带宽，就像我们经常混淆手机的运存和闪存一样

## 工具安装

以 CentOS 7.4 为例

```shell
yum install fio -y
```

## 参数说明

| 参数名称            | 说明                                                                                                 | 取值样例                            |
| --------------- | -------------------------------------------------------------------------------------------------- | ------------------------------- |
| name            | 测试任务名称                                                                                             | 4k-randwrite                    |
| filename        | 测试对象                                                                                               | /dev/sdb(裸盘) 或 /data/test(文件系统) |
| bs              | 请求块大小                                                                                              | 4k、8k、16k、128k、1M...            |
| size            | I/O 测试的寻址空间                                                                                        | 10G                             |
| ioengine        | I/O 引擎，推荐使用 Linux 的异步 I/O 引擎                                                                       | libaio                          |
| iodepth         | 请求的 I/O 队列深度，处定义的队列深度是指每个线程的队列深度，如果有多个线程测试，意味着每个线程都是此处定义的队列深度。fio 总的**I/O 并发数**=iodepth * numjobs。 | 64                              |
| numjobs         | 测试的并发线程数                                                                                           | 1                               |
| direct⭐️        | 定义是否使用 direct I/O，可选值如下：值为0，表示使用 buffered I/O ；值为1，表示使用 direct I/O                                 | 1                               |
| rw              | 读写模式，取值包括顺序读（read）、顺序写（write）、随机读（randread）、随机写（randwrite）、混合随机读写（randrw）和混合顺序读写（rw，readwrite）     | read                            |
| time_based      | 指定采用时间模式。无需设置该参数值，只要 FIO 基于时间来运行                                                                   | N/A                             |
| runtime         | 指定测试时长，即 FIO 运行时长                                                                                  | 100                             |
| refill_buffers  | FIO 将在每次提交时重新填充 I/O 缓冲区。默认设置是仅在初始时填充并重用该数据                                                         | N/A                             |
| norandommap     | 在进行随机 I/O 时，FIO 将覆盖文件的每个块。若给出此参数，则将选择新的偏移量而不查看 I/O 历史记录                                            | N/A                             |
| randrepeat      | 随机序列是否可重复，True（1）表示随机序列可重复，False（0）表示随机序列不可重复。默认为 True（1）                                          | 0                               |
| group_reporting | 多个 job 并发时，打印整个 group 的统计值                                                                         | N/A                             |

## HDD 基准性能测试

> 注意将 `/dev/sdb` 换成实际要测试的硬盘盘符名称

1. 通过 fio.conf 测试硬盘性能，执行 `fio fio.conf` 即可

```fio.conf
[global]
ioengine=libaio      # 异步IO
direct=1             # 排除OS的IO缓存机制的影响
size=5g              # 每个fio进程/线程的最大读写
lockmem=1G           # 锁定所使用的内存大小
runtime=30           # 根据上面的结论以后采用30值
filename=/dev/sdb    # 裸盘
numjobs=1            # 同时进行的任务数
iodepth=1            # 队列深度
randrepeat=0         # 随机序列不可重复
group_reporting      # 多个job合并出报告
refill_buffers       # 每次提交时重新填充 I/O 缓冲区

[4k_randwrite]
stonewall            # 隔离各测试任务
rw=randwrite
bs=4k

[4k_randread]
stonewall
rw=randread
bs=4k
iodepth=64

[128k_write]
stonewall
rw=write
bs=128k

[128k_read]
stonewall
rw=read
bs=128k
```

2. 通过命令执行测试

- 4k 随机写测试

```shell
fio --name=4k_randwrite --ioengine=libaio --direct=1 --size=5g --lockmem=1G --runtime=30 --filename=/dev/sdb --numjobs=1 --iodepth=1 --randrepeat=0 --group_reporting --refill_buffers --rw=randwrite --bs=4k
```

- 4k 随机读测试

```shell
fio --name=4k_randread --ioengine=libaio --direct=1 --size=5g --lockmem=1G --runtime=30 --filename=/dev/sdb --numjobs=1 --iodepth=1 --randrepeat=0 --group_reporting --refill_buffers --rw=randread --bs=4k --iodepth=64
```

- 128k 顺序写测试

```shell
fio --name=128k_write --ioengine=libaio --direct=1 --size=5g --lockmem=1G --runtime=30 --filename=/dev/sdb --numjobs=1 --iodepth=1 --randrepeat=0 --group_reporting --refill_buffers --rw=write --bs=128k
```

- 128k 顺序读测试

```shell
fio --name=128k_read --ioengine=libaio --direct=1 --size=5g --lockmem=1G --runtime=30 --filename=/dev/sdb --numjobs=1 --iodepth=1 --randrepeat=0 --group_reporting --refill_buffers --rw=read --bs=128k
```

## SSD 基准性能测试

> 注意将 `/dev/sdb` 换成实际要测试的硬盘盘符名称

1. 通过 fio.conf 测试硬盘性能，执行 `fio fio.conf` 即可

```fio.conf
[global]
ioengine=libaio      # 异步IO
direct=1             # 排除OS的IO缓存机制的影响
size=5g              # 每个fio进程/线程的最大读写
lockmem=1G           # 锁定所使用的内存大小
runtime=30           # 根据上面的结论以后采用30值
filename=/dev/sdb    # 裸盘
numjobs=1            # 同时进行的任务数
iodepth=1            # 队列深度
group_reporting      # 多个job合并出报告
randrepeat=0         # 随机序列不可重复
group_reporting      # 多个job合并出报告
refill_buffers       # 每次提交时重新填充 I/O 缓冲区

[4k_randwrite]
stonewall            # 隔离各测试任务
rw=randwrite
bs=4k
numjobs=2
iodepth=4

[4k_randread]
stonewall
rw=randread
bs=4k
numjobs=2
iodepth=32

[128k_write]
stonewall
rw=write
bs=128k
iodepth=2

[128k_read]
stonewall
rw=read
bs=128k
numjobs=2
iodepth=16
```

2. 通过命令执行测试

- 4k 随机写

```shell
fio --name=4k_randwrite --ioengine=libaio --direct=1 --size=5g --lockmem=1G --runtime=30 --filename=/dev/sdb --numjobs=2 --iodepth=4 --randrepeat=0 --group_reporting --refill_buffers --rw=randwrite --bs=4k
```

- 4k 随机读

```shell
fio --name=4k_randread --ioengine=libaio --direct=1 --size=5g --lockmem=1G --runtime=30 --filename=/dev/sdl5 --numjobs=2 --iodepth=32 --randrepeat=0 --group_reporting --refill_buffers --rw=randread --bs=4k
```

- 128k 顺序写

```shell
fio --name=128k_write --ioengine=libaio --direct=1 --size=5g --lockmem=1G --runtime=30 --filename=/dev/sdb --numjobs=1 --iodepth=2 --randrepeat=0 --group_reporting --refill_buffers --rw=write --bs=128k
```

- 128k 顺序读

```shell
fio --name=128k_read --ioengine=libaio --direct=1 --size=5g --lockmem=1G --runtime=30 --filename=/dev/sdb --numjobs=2 --iodepth=16 --randrepeat=0 --group_reporting --refill_buffers --rw=read --bs=128k
```

## 结果解读

以 1.2T 10000RPM 的 SAS HDD 测试结果为例，4K 测试主要看的是 IOPS 指标和延迟，128K 测试主要看的是 BW 指标和延迟

![image.png](https://ob-img-nest.oss-cn-hongkong.aliyuncs.com/2023/07/6b0b3f7310d863efd4f0a92379e7cc01.png?x-oss-process=style/ob-img-shuiyin)