---
layout:     post
title:      Mapreduce原理
subtitle:   
date:       2019-7-25
author:     BY KiloMeter
header-img: img/2019-7-25-Mapreduce原理/66.jpg
catalog: true
tags:
    - 大数据
---

Map过程

- 第一阶段把输入文件按照一定的标准分片(InputSplit)，每个输入分片的大小是固定的，默认情况下，输入分片的大小和数据块(block)的大小是相同的，每一个分片由一个Map进程处理。
- 第二阶段是把输入分片的数据按照一定的规则解析成键值对，默认规则是以文件每一行的内容为值，每一行起始位置的字节作为键
- 第三阶段是调用Mapper的Map方法，第二阶段有多少键值对，Map方法就会调用多少次，把第二阶段的键值对按照Map实现的方法，产生0个或多个键值，在这个过程中，产生的数据首先会被存放到内存缓冲区当中，每个Map有一个环形缓冲区，默认大小为100M，当使用率达到80%后，将会把数据溢写到本地磁盘，溢写到磁盘前，会先经历以下阶段
- 第四阶段是分区，对于每个键，会有一个partition值，该值是通过计算键的hash值后对reduce的数量进行取模获得，分区的数量决定了reduce**任务运行的数量**(注：这里指的是reduce任务运行的数量而不是reduce的数量，也就是说，reduce数量如果大于partition的数量，将会有部分reduce无法接收到数据)
- 第五阶段是对键值对进行排序，首先根据键排序，键相同的情况下按照值进行排序，然后将排序后的结果写到磁盘上，由于可能会出现多个溢写文件，在最后输出时，会把所有的溢写文件都合并成一个文件。
- 第六阶段是combiner，是可选的，默认是没有的，如果要开启第六阶段得实现代码，第六阶段在本地进行归约，就是一次小型的reduce，把本地键相同的数据合并到一块，减少网络IO

Reduce过程

- 第一阶段，reduce先从map端拉取数据
- 第二阶段，把拉取到本地的数据进行一次合并，把键相同的数据的值合并到一块，然后进行排序
- 第三阶段，对排完序的键值对调用Reduce方法，键相同的键值对调用一次reduce方法，最后把reduce的结果写入到hdfs。

reduce的详细过程如下:

reduce会在部分map已经完成的情况下，开始拉取这部分map的数据。由于job的每一个map都会根据reduce(n)数将数据分成map 输出结果分成n个partition，所以map的中间结果中是有可能包含每一个reduce需要处理的部分数据的。所以，为了优化reduce的执行时间，hadoop中是等job的第一个map结束后，所有的reduce就开始尝试从完成的map中下载该reduce对应的partition部分数据，因此map和reduce是交叉进行的。Reduce任务通过HTTP向各个Map任务拖取（下载）它所需要的数据（网络传输），Reducer是如何知道要去哪些机器取数据呢？一旦map任务完成之后，就会通过常规心跳通知应用程序的Application Master。reduce的一个线程会周期性地向master询问，直到提取完所有数据。数据被reduce提走之后，map机器不会立刻删除数据，这是为了预防reduce任务失败需要重做。因此map输出数据是在整个作业完成之后才被删除掉的。

reduce的每一个下载线程在下载某个map数据的时候，有可能因为那个map中间结果所在机器发生错误，或者中间结果的文件丢失，或者网络瞬断等等情况，这样reduce的下载就有可能失败，所以reduce的下载线程并不会无休止的等待下去，当一定时间后下载仍然失败，那么下载线程就会放弃这次下载，并在随后尝试从另外的地方下载（因为这段时间map可能重跑）。reduce下载线程的这个最大的下载时间段是可以通过mapreduce.reduce.shuffle.read.timeout（default180000秒）调整的。如果集群环境的网络本身是瓶颈，那么用户可以通过调大这个参数来避免reduce下载线程被误判为失败的情况。一般情况下都会调大这个参数。

reduce端在轮流拉取map端数据时，也是先把拉取的数据保存到reduce任务的JVM内存缓冲区当中，这个缓冲区的大小由mapreduce.reduce.shuffle.input.buffer.percent 这个参数指定，默认是0.7，意思是缓冲区的大小为reduce堆内存大小的70%，和map端一样，在map端当使用了0.8的内存后，将会进行溢写，reduce端同样也有溢写操作，通过mapreduce.reduce.shuffle.merge.percent这个参数指定开始溢写时的缓冲区使用比例(默认是0.66)。在reduce端也会进行conbiner操作(如果有设置的话)，直到map端数据全部拉取完毕后，生成了多个溢写文件，会启动磁盘到磁盘的merge方式生成最终的文件。

Mapreduce调优：

从上面Map过程和reduce过程可以得出一些调优的方法：

Map端调优：

1、通过调整mapreduce.task.io.sort.mb参数，来设置map输出时所使用的内存缓冲区大小

2、设置maprduce.map.sort.spill.percent参数来改变溢写时的内存使用率，默认内存缓冲区使用0.8后开始溢写

3、mapreduce.task.io.sort.factor，排序文件时，一次最多合并的流数。由于map输出的结果中，可能会出现多个溢写文件，这个参数就是设置了可以同时合并溢写文件数量。

4、mapreduce.map.output.compress，设置压缩map输出

5、mapreduce.map.output.compress.codec，设置map输出的压缩编码器

6、mapreduce.map.combine.minspills 这个参数指定了运行combiner所需要的最少溢写文件数量(如果有指定combiner的话)

Reduce端调优：

1、mapreduce.reduce.shuffle.parallelcopies  该参数设置了reduce从map端拉取数据时，可以使用多少个线程同时拉取数据，默认是5，当map的数量比较多时，可以把该参数的值设置大一些

2、mapreduce.task.io.sort.factor，这个参数在Map端调优中也有，在Map端是是合并一个Map的多个溢写文件，但在reduce端合并的是多个Map中属于自己分区的溢写文件。

3、mapreduce.reduce.shuffle.input.buffer.percent  在shuffle的复制阶段，分配给存放map输出的缓冲区大小占reduce的JVM堆内存比例，默认是0.7

4、mapreduce.reduce.input.buffer.percent 该参数是reduce过程中，内存中保存map输出空间占内存缓冲区的比例，当达到这个比例时，会启动溢写。

5、mapreduce.reduce.merge.inmem 该参数和第四个参数一样，也是用于控制溢写条件，不过这个值指的是内存中map的输出键值对的个数，默认是1000，如果设置为0，那么溢写条件仅由mapreduce.reduce.input.buffer.percent 参数单独控制