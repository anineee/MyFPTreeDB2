# 论文阅读与前期工作总结
姓名：陆俞因，廖婕， 陈文丰<br/>
学号：17343081，17343072，17343016

---
## 前期工作

### 使用示意图展示普通文件IO方式(fwrite等)的流程，即进程与系统内核，磁盘之间的数据交换如何进行？为什么写入完成后要调用fsync？
![普通文件IO方式示意图](http://blog.chinaunix.net/attachment/201207/11/27105712_13419741441gpp.gif)

普通文件IO需要穿越多层，从进程写到系统内核，再从系统内核最终写到磁盘。

`fwrite()`是系统提供的最上层接口，作用是在用户进程空间开辟 **CLib buffer** ，将多次小数据量相邻写操作先从 **application buffer** 拷贝到 **CLib buffer** 缓存起来，如果此时进程终止，数据会丢失。此时可以调用`fclose()`主动将 **CLib buffer** 的数据刷新到磁盘介质上。

一种方式是调用`write()`，将数据从应用层中的 **CLib buffer** 的数据一次性拷贝到内核层中的 **page cache** ，此时会触发内核态/用户态的切换。数据到达 **page cache** 后由内核IO调度决定何时写入磁盘。另一种方式是调用`fflush()`将数据从 **CLib buffer** 拷贝到 **page cache** 中。

内核中pdflush不停监测脏页，判断是否要写回到磁盘中，并将需要写回的页提交到IO队列中，根据IO调度队列的调度策略决定何时写回。从IO队列出来后，进入驱动层，驱动层通过DMA将数据写入 **disk cache** 。由磁盘控制器决定何时将 **disk cache** 的数据写到磁盘介质上，或是通过调用`fsync()`主动将 **page cache** 中的数据刷新到磁盘上。

### 简述文件映射的方式如何操作文件。与普通IO区别？为什么写入完成后要调用msync？文件内容什么时候被载入内存？
常规文件操作为了提高读写效率和保护磁盘，使用了页缓存机制,即读文件时需要先将文件页从磁盘拷贝到 **page cache** 中，由于 **page cache** 处在内核空间，不能被用户进程直接寻址，所以还需要将 **page cache** 中数据页再次拷贝到内存对应的用户空间中。这样，通过了两次数据拷贝过程，才能完成进程对文件内容的获取任务。写操作也是一样，待写入的 **application buffer** 在内核空间不能直接访问，必须要先拷贝至内核空间对应的主存，再写回磁盘中（延迟写回），也是需要两次数据拷贝。

区别于普通IO，文件映射通过将一个文件或者其它对象映射到进程的地址空间，实现文件磁盘地址和进程虚拟地址空间中一段虚拟地址的一一对映关系。实现这样的映射关系后，进程就可以采用指针的方式读写操作这一段内存，而系统会自动回写脏页面到对应的文件磁盘上，即完成了对文件的操作。在之后访问数据时发现内存中并无数据而发起的缺页异常过程，可以通过已经建立好的映射关系，只使用一次数据拷贝，就从磁盘中将数据传入内存的用户空间中，供进程使用。
![文件映射示意图](https://images0.cnblogs.com/blog2015/571793/201507/200501092691998.png)

文件映射的实现过程可以分为三个阶段。

首先，进程在用户空间调用`mmap()`启动映射过程并在用户的虚拟地址空间创建一个新的虚拟映射区。

然后，在内核空间调用`mmap()`（不同于用户空间函数），实现文件物理地址和进程虚拟地址的一一映射关系。至此，没有将任何文件数据拷贝到内存。

最后，进程发起对这片映射空间的访问，引发缺页异常，确认无非法操作后，内核发起请求调页。先在交换缓存空间 **swap cache** 中寻找需要访问的内存页，如果没有则将文件内容从磁盘拷贝到物理内存。之后进程即可对这片主存进行读写操作，如果写操作改变了其内容，一定时间后系统会自动回写脏页面到对应磁盘地址，完成写入到文件的过程。修改过的脏页面并不会立即更新回文件中，而是有一段时间的延迟，可以调用`msync()`来强制同步, 将所写的内容立即保存到文件里了。

### 参考Intel的NVM模拟教程模拟NVM环境，用fio等工具测试模拟NVM的性能并与磁盘对比（关键步骤结果截图）。
在 Ubuntu 下配置，跳过内核配置、编译和安装步骤。

首先，通过`dmesg | grep BIOS-e820`命令查看测试机的内存情况,可见从地址0x100000000到地址0x23fffffff有4.25G空闲内存。

![内存情况](https://raw.githubusercontent.com/anineee/Images/master/set-again2.JPG)

然后，`sudo su`进入根模式后通过`gedit /etc/default/grub`修改内存配置，添加语句`memmap=4G!4G`表示从内存4G位置（!标识符后）开始，划分4G大小（!标识符前）内存空间为非易失性内存。

![gedit grub](https://raw.githubusercontent.com/anineee/Images/master/gedit-grub2.JPG)

若配置成功，在 \dev 目录下生成 pmem0m 设备。通过`dmesg | grep user`查看分配后的内存情况，通过`sudo fdisk -l \dev\pmem0m`确认系统能够识别设备并查看设备详情，在 \dev 目录下通过`lsblk`查看内存情况。

![dmesg](https://raw.githubusercontent.com/anineee/Images/master/grup-succeed2_LI.jpg)

![fdisk](https://raw.githubusercontent.com/anineee/Images/master/grup-succeed_LI.jpg)

![lsblk](https://raw.githubusercontent.com/anineee/Images/master/lsblk_LI.jpg)

然后为进行 fio 测试，先将模拟 NVM 挂载在文件系统下，然后就能像普通磁盘一样操作。

![fs](https://raw.githubusercontent.com/anineee/Images/master/fs1.jpg)

随后安装 fio 工具测试模拟NVM的性能并与磁盘对比。首先在官网下载安装包后，通过`.\configure` `make` `make install` 进行安装。具体参数可通过`fio -help`查看。

共进行三次测试，第一次测试为随机读写(`-rw=randrw`)2G大小(`-size=2G`)内存，其中读操作占70%(`-rwmixread=70`)。

```
$ fio -filename=/dev/sda -direct=1 -iodepth 1 -thread -rw=randrw -rwmixread=70 -ioengine=psync -bs=4k -size=2G -numjobs=50 -runtime=180 -group_reporting -name=randrw_70read_4k_disk

$ fio -filename=/dev/pmem0m -direct=1 -iodepth 1 -thread -rw=randrw -rwmixread=70 -ioengine=psync -bs=4k -size=2G -numjobs=50 -runtime=180 -group_reporting -name=randrw_70read_4k_nvm
```

第二次测试为随机读(`-rw=randread`)2G大小内存。

```
$ fio -filename=/dev/sda -direct=1 -iodepth 1 -thread -rw=randread -ioengine=psync -bs=4k -size=2G -numjobs=50 -runtime=180 -group_reporting -name=randr_4k_disk

$ fio -filename=/dev/pmem0m -direct=1 -iodepth 1 -thread -rw=randread -ioengine=psync -bs=4k -size=2G -numjobs=50 -runtime=180 -group_reporting -name=randr_4k_nvm
```

第三次测试为随机写(`-rw=randwrite`)2G大小内存。

```
$ fio -filename=/dev/sda -direct=1 -iodepth 1 -thread -rw=randwrite -ioengine=psync -bs=4k -size=2G -numjobs=50 -runtime=180 -group_reporting -name=ranrw_4k_disk

$ fio -filename=/dev/pmem0m -direct=1 -iodepth 1 -thread -rw=randwrite -ioengine=psync -bs=4k -size=2G -numjobs=50 -runtime=180 -group_reporting -name=randw_4k_nvm
```

姑且只关注结果报告中画线部分，

## 论文阅读

### 总结一下本文的主要贡献和观点(500字以内)(不能翻译摘要)。
#### 动机背景
本文工作的动机背景简单来说就是，出现了一种新型内存——SCM（Storage Class Memory）。
SCM的特性与传统的存储介质有很大的不同。如何基于SCM介质的特性，在软件层面做出相关的改变，充分发挥SCM的能力，是下一代超高性能存储系统需要解决的至关重要的问题。目前，一些研究已经研究了如何在SCM中设计持久树作为这些新系统的基本构建块。然而，这些树明显慢于基于DRAM的树，因为树对延迟敏感，并且SCM表现出比DRAM更高的延迟。
#### 主要工作和所解决的问题
1）提出了FPTree，即Fingerprinting Persistent Tree
2）对FPTree进行了全面的性能评估：证明了FPTree的性能优于具有不同SCM延迟的最先进的持久树，最高可达8.2倍；证明了FPTree在具有88个逻辑核心的机器上可以很好地扩展；将FPTree、NV-Tree和wBTree集成到memcached和数据库原型中，证明了与使用完全瞬态数据结构相比，FPTree的性能开销几乎可以忽略不计，同时显著优于其他持久树。
#### 技术原理
1）指纹技术——将平均的中间节点搜索键数量限制为1个；
2）选择性持久——叶子节点保存在SCM中，而中间节点保存在DRAM中并在恢复时重建。
3）选择性并发——使用硬件事务内存（HTM）来处理中间节点的并发性，使用细粒度锁来处理叶节点的并发性
#### 意义
提出了一种性能优良的SCM中的持久树——FPTree

### SCM硬件有什么特性？与普通磁盘有什么区别？普通数据库以页的粒度读写磁盘的方式适合操作SCM吗？
XXXXXX

### 操作SCM为什么要调用CLFLUSH等指令？
(写入后不调用，发生系统崩溃有什么后果)
XXXXXX

### FPTree的指纹技术有什么重要作用？
由于线性查找未经过排序的叶子结点会在SCM中消耗大量的时间。为了提高效率，使用指纹技术，通过记录叶键的一个字节长度的哈希，防止探测没有匹配的键，极大的提高性能。

### 为了保证指纹技术的数学证明成立，哈希函数应如何选取？
（哈希函数生成的哈希值具有什么特征，能简单对键值取模生成吗？）
哈希函数需要满足均匀分布m个元素在n个可能的哈希值中，在一个字节的fingerprints的情况下n=256。
可以对键值进行简单取模，因为这样满足均匀分布的定理且满足期望推理中的概率公式

### 持久化指针的作用是什么？与课上学到的什么类似？
由于程序重新启动的时候，通常会分配新的地址，使得之前的指针非法，我们通常需要采用持续化指针。其中包含8字节的文件ID以及8字节的偏移。通过持久性分配器切换易失性和持久性。在失败的时候通过刷新操作清除。

>>>>>>> 5c166240d8d600638423f6677bd2f585d83f0fd3
