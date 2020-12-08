### 逆向处理会遇到的问题

一般业务场景如下，数据源头产生数据，进入Kafka，然后由消费者（如Flink，Spark，Kafka API）处理数据后进入到HBase。这是一个很典型的实时处理流程。流程图如下：

![典型的实时处理流程](https://raw.githubusercontent.com/MylittleTown/notes/master/Linked_pictures/%E7%AC%94%E8%AE%B06-%E5%85%B8%E5%9E%8B%E7%9A%84%E5%AE%9E%E6%97%B6%E5%A4%84%E7%90%86%E6%B5%81%E7%A8%8B.png)

上述这类实时处理流程，处理数据都比较容易，毕竟数据流向是顺序处理的。但是，下面将介绍当这个流程逆向会遇到的一些问题。

1. 海量数据

   HBase 的分布式特性，集群的横向拓展，HBase 中的数据往往都是百亿，千亿级别的，或者数量级更大。这类级别的数据，对于这类逆向数据流的场景，会有个很麻烦的问题，那就是取数问题。如何将这海量数据从HBase 中取出来？

2. 没有数据分区

   我们知道HBase 做数据Get 或者List\<Get> 很快，也比较容易。而它又没有类似Hive 这类数据仓库分区的概念，不能提供某段时间内的数据*（Q: 没有索引工具吗？没有时间属性上的索引，这里应该不是“不能提供某段时间内的数据”，而是“不能快速的提供”，原因就是全表扫描的延迟）*。如果要提取最近一周的数据，可能全表扫描，通过过滤时间戳来获取一周的数据。数据量小的时候，可能问题不大，而数据量很大的时候，全表区扫描HBase 很困难。

### 解决思路

对于这类逆向数据流程，如何处理。其实，我们可以利用HBase Get 和List\<Get> 的特性来实现，其中get() 方法也有对应的Get类，需要构造Get 实例，由于get() 操作每次只能读取一行数据（不限制一行当中取多少列或者多少单元格），HBase 还提供了Get 列表，使用列表参数的get() 方法，用户可以用一次请求来获取多行数据，允许用户快速高效的从远程服务器获取相关的或完全随机的多行数据。因为HBase 通过RowKey 来构建一级索引，对于RowKey 级别的取数，速度是很快的（原因估计是列簇下的多列数据都可能值对应一个RowKey，全表扫描速度更快）。实现流程细节如下：

![逆向数据流程设计](https://raw.githubusercontent.com/MylittleTown/notes/master/Linked_pictures/%E7%AC%94%E8%AE%B06-%E9%80%86%E5%90%91%E6%95%B0%E6%8D%AE%E6%B5%81%E7%A8%8B%E8%AE%BE%E8%AE%A1.png)

MapReduce 是一种分布式计算框架，以一种可靠的，具有容错能力的方式并行处理上TB 级别的海量数据集。由两个阶段组成：Map 和Reduce，用户只需实现map() 和reduce() 两个函数，即可实现分布式计算，其中map() 负责把一个大的block 块切片并计算，reduce() 负责把map() 切片的数据进行汇总，计算。

![MapReduce框架](https://raw.githubusercontent.com/MylittleTown/notes/master/Linked_pictures/%E7%AC%94%E8%AE%B06-MapReduce%E6%A1%86%E6%9E%B6.png)

具体原理流程如下：

1. 第一步对输入的数据进行切片，每个切片分配一个map() 任务，map() 对其中的数据进行计算，对每个数据用键值对的形式记录，然后输出到环形缓冲区（图中sort 的位置）
2. map() 中输出的数据在环形缓冲区内进行快排，每个环形缓冲区默认大小为100M，当数据达到80M时（默认），把数据输出到磁盘上。形成很多个内部有序整体无序的小文件（图中merge 的输出结果）。
3. 框架把磁盘中的小文件传到reduce() 中来，然后进行归并排序

HDFS（Hadoop Distributed File System）是分布式计算中数据存储骨干里的基础，是基于流数据模式访问和处理超大文件的需求而开发的，可以运行于廉价的商用服务器上。它具有高容错，高可靠性，高可扩展性，高获得性，高吞吐率等特征。

其特点包括：高容错性，可构建在廉价机器上；适合批处理；适合大数据处理；流式文件访问。

局限性包括：不支持低延迟访问；不适合小文件存储；不支持并发写入；不支持修改。*（Q: 不支持并发写入，相当于海量数据要通过流式数据的形式输入，窗口化处理，存在延迟吗？）*

扩展：大数据存储生态圈简介

Hive 和HBase 的数据一般都存储在HDFS 上。HDFS为他们提供了高可靠性的底层存储支持

Hive 不支持更改数据的操作，Hive 基于数据仓库，提供静态数据的动态查询。其使用类SQL 语言，底层经过编译转为MapReduce 程序，在Hadoop 上运行，数据存储在HDFS上。

HBase 是Hadoop database，即Hadoop 数据库。是一个适合于非结构化数据存储的数据库，HBase 基于列的而不是基于行的模式。HBase 利用HDFS作为其文件存储系统，利用Hadoop MapReduce 来处理HBase 中的海量数据。

下面将介绍从HBase 中提取数据到消费者端的原理：

1. RowKey 的提取

   HBase 针对RowKey 取数做了一级索引，我们可以将海量数据中的RowKey 从HBase 表中抽取，然后按照我们制定的抽取规则和存储规则将抽取的RowKey 存储到HDFS*（Q: 作为文件存储系统，适合海量数据存储，但是不适合低延迟访问，这个如何解决）*上。

   这里需要注意一个问题，关于HBase RowKey 的抽取，海量数据级别的RowKey 抽取，建议采用MapReduce 来实现。这个得益于HBase 提供了TableMapReduceUtil 类来实现，通过MapReduce 任务，将HBase 中的RowKey 在Map 阶段按照指定的时间范围进行过滤，在Reduce 阶段将RowKey 拆分为多个文件，最后存储到HDFS上。

   这里可能会有疑问，都用MapReduce 抽取RowKey 了，为什么不直接在扫描处理列簇下的列数据呢？这里，在启动MapReduce 任务的时候，扫描（Scan） HBase 的数据时只过滤RowKey （利用FirstKeyOnlyFilter 来实现），不对列簇数据做处理，这样会快很多。对HBase RegionServer 的压力也会小很多。

   HBase 中的行键过滤器常用的有PrefixFilter, KeyOnlyFilter, FirstKeyOnlyFilter 和InclusiveStopFilter 几种，其中FirstKeyOnlyFilter 过滤器只扫描显示相同键的第一个单元格的键值对，可以用来实现对逻辑行进行计数的功能，并且比其他计数方式效率高。

   ![HBase中特征数据结构](https://raw.githubusercontent.com/MylittleTown/notes/master/Linked_pictures/%E7%AC%94%E8%AE%B06-HBase%E4%B8%AD%E7%89%B9%E5%BE%81%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84.png)

   这里举个例子，比如上表中的数据，其实我们只需要取出RowKey(row001)。但是，实际业务数据中，HBase 表描述一条数据可能有很多特征属性（比如姓名，性别，年龄，身份证等等），可能有些业务数据一个列簇下有十几个特征，但是只有一个RowKey（所以抽取RowKey 将减少搜索列簇下的十几个数据的数据量），那么我们使用FirstKeyOnlyFilter 来实现就很合适了。

2. RowKey 生成

   抽取的RowKey 如何生成，这里可能根据实际的数量级来确认Reduce 个数。建议生成RowKey 文件时，切合实际的数据量来算Reduce 的个数。尽量不用为了使用方便就一个HDFS文件，这样后面不好维护。举个例子，比如HBase 表有100GB，我们可以拆分为100个文件。

3. 数据处理

   在步骤1中，按照抽取规则和存储规则，将数据从HBase 中通过MapReduce 抽取RowKey 并存储到HDFS上。然后，我们再通过MapReduce 任务读取HDFS上的RowKey 文件，通过List\<Get> 的方式去HBase 中获取数据。拆解细节如下：

   ![利用MapReduce完成数据迁移](https://raw.githubusercontent.com/MylittleTown/notes/master/Linked_pictures/%E7%AC%94%E8%AE%B06-%E5%88%A9%E7%94%A8MapReduce%E5%AE%8C%E6%88%90%E6%95%B0%E6%8D%AE%E8%BF%81%E7%A7%BB.png)

   Map 阶段，我们从HDFS读取RowKey 的数据文件，然后批量Get 的方式从HBase 取数，然后组装数据发送到Reduce 阶段。

   在Reduce 阶段，获取来自Map 阶段的数据，写数据到Kafka，通过Kafka生产者回调函数，获取写入Kafka 状态信息，根据状态信息判断数据是否写入成功。

   如果成功，记录成功的RowKey 到HDFS，便于统计成功的进度；如果失败，记录失败的RowKey 到HDFS，便于统计失败的进度。

4. 失败重跑

   通过MapReduce 任务写数据到Kafka 中，可能会有失败的情况，对于事变的情况，我们只需要记录RowKey 到HDFS 上，当任务执行完成后，再去程序检查HDFS上是否存在失败的RowKey 文件，如果存在，那么再次启动步骤10，即读取HDFS上失败的RowKey 文件，然后再List\<Get> HBase 中的数据，进行数据处理后，最后再写Kafka，以此类推，直到HDFS上失败的RowKey 处理完成为止。

### 总结

整个逆向数据处理流程的实现是很基本的MapReduce 逻辑，处理过程中，需要注意几个细节问题：

RowKey 生成到HDFS 上时，可能存在行位空格的情况，在读取HDFS上RowKey 文件去List\<Get> 时，最好对每条数据做个过滤空格处理。另外，对于成功处理RowKey 和失败处理RowKey 的记录，这样便于任务失败重跑和数据对账。可以知晓数据迁移进度和完成情况。同时，可以使用Kafka Eagle监控工具来查看Kafka 的写入进度。