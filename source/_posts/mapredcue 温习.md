---
title: mapreduce 温故
tags: mapreduce shuffer
toc: true
comments: true
---

## 需求背景：
>做大数据有一段时间了，梳理下用到mapreduce的一些问题和解决方案。

## mapreduce
>mapreduce:顾名思义，map做映射，reduce做规约。
>主要分以下步骤：
1.输入分块
2.map
3.shuffer
4.reduce

![mapredcue流程图](/ITWO/assets/mapreduce01.jpg)
重点是shuffer阶段


## reduce个数的计算方法:

``` java
double bytes = Math.max(totalInputFileSize, bytesPerReducer);
int reducers = (int) Math.ceil(bytes / bytesPerReducer);
reducers = Math.max(1, reducers);
reducers = Math.min(maxReducers, reducers);
```
>　从计算逻辑可以看出该量由输入文件的大小以及设置的每个reduce可以处理的字节数大小决定．

>　map接受分块数据，然后使用默认的分区算法（对输入文件的kv中对key hash后再对reduce task数量取模(reduce个数的算法见前文)），再通过内存缓冲区进行排序．
>　这个溢写是由单独线程来完成，不影响往缓冲区写map结果的线程。溢写线程启动时不应该阻止map的结果输出，所以整个缓冲区有个溢写的比例spill.percent。这个比例默认是0.8，也就是当缓冲区的数据已经达到阈值（buffer size * spill percent = 100MB * 0.8 = 80MB），溢写线程启动，锁定这80MB的内存，执行溢写过程。Map task的输出结果还可以往剩下的20MB内存中写，互不影响。
> 　达到阈值以后溢写到磁盘上，排序用的是字典顺序的排序,排序的对象是序列化好的字节。最后溢写得到的多个文件根据hash值merge，也可以自定义combiner（可以理解为map端的reduce）），然后reduce开始fetch（拉取）map端合并好的数据，拉取来自不同map的相同分区的数据（相同key的数据）过来，然后在reduce端合并，此时在reduce端也会进行一次sort。

![shuffer流程图](/ITWO/assets/mapreduce02.png)

