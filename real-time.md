# 实时数仓
## 背景
在早期数仓建设中，大多是以批处理的方式为基线进行开发，随着业务发展，需求对于时效性和准确性的要求越来越高，为能满足日益强烈的诉求，并且兼容批处理本身已有的基础建设，本文结合当前infra team所能支撑的前提下，对实时数仓构建的一些想法

## 现有方式的一些不足
### lambda

**概述**
![lambda](https://upload-images.jianshu.io/upload_images/13491351-c216b21938cfefef.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

作为最为经典和广泛应用的设计，抽象如下：
- 所有的数据需要分别写入批处理层和流处理层
- 批处理持久化数据，并做预计算
- 流处理作为速度层，对实时数据计算近似的 real-time view，作为高延迟 batch view 的快速补偿视图
- 所有查询需要合并batch view 和 real-time view

**优势**
- 在实时性和准确性上达到均衡，从而满足业务诉求
- 批处理持久化存储，数据可重放，可扩展性和容错性较好
- 流处理产出近似结果，时效性上得到满足

**不足**
- 数据从源头上区分传输和存储，存在数据一致性问题
- 两套计算逻辑，开发维护成本较大
- 因为框架的天然劣势，实时只能是近似计算，需要T+1的批处理进行修正

### kappa

**概述**
核心点在于去掉批处理的数据流向，全部用流处理代替，近些年也有依赖同DB的方式，即兼容OLAP和OLTP，统一数据流向

![kappa](https://upload-images.jianshu.io/upload_images/13491351-b247726c0d89e859.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


**优势**
- 数据源头不一致问题得到改善，或者全部持久化到消息中间件（kafka为代表），或者consume消息中间件，持久化到兼容DB（Tidb为代表）
- 服务层不一致问题得到改善，通过重放持久化的消息中间件或兼容DB，实现容错或可扩展目的

**不足**
- 无法解决存储成本，以kafka为例，kafka的压缩能力有限，存储成本较大
- 计算资源开销随着窗口步长的增加而飙升。例如计算月活或年报统计时，涉及到大状态的缓存，无论是持久化到kafka或兼容DB，可重放性极差，存在消费雪崩甚至无法消费的情况

### kappa+

**概述**

核心点在于统一计算引擎，同套计算逻辑消费不同流向的数据，以flink为代表，无论是datastream/dataset，还是力推的flink sql，皆以此为目的

![kappa+](https://upload-images.jianshu.io/upload_images/13491351-6456002a4ddbe067.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


**优势**

*   计算过程不一致问题得到改善，以flink sql为例，流或批处理时，无需维护两套代码，由框架本身承接场景切换时的兼容逻辑

**不足**

- 不能完全兼容hive sql/spark sql，已经成熟的数仓体系下，切换有较大成本
- 流批sql同个function实际表达的功能可能不同
- 在流join或大数据量去重上，开销或实现成本仍较大或较为困难

### 总结

可以看到，痛点主要有以下三点：
- 数据源的流向和存储不同，从源头上不能一致
- 计算逻辑或框架不同，计算过程不能一致
- 计算结果需要合并，服务层不能一致

因此，迭代方案主要围绕以上三点进行优化


## 设计思路
> 我们假设数据源只包含业务库（以MySQL为代表）和用户埋点（Nginx Log）

如图所示，实时数仓的建设思路中，大量参考和绑定离线数仓的构建，包括数据链路，加工流程，交互的Services。重点强调三个”Same“

![lambda+](https://upload-images.jianshu.io/upload_images/13491351-088d355fd9c53b13.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### Same Source

即流或批处理的源数据流向统一由consume kafka获得，通过at least once的机制，确保数据在消费阶段不会丢失

![same source](https://upload-images.jianshu.io/upload_images/13491351-c2d045520d5ec5e3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


- At least once导致的消费重复，流、批作业需根据场景选择性的处理，比如ETL process的DB支持幂等，Application不需要任何处理。下游DB是append-only的，则在层级流转时，做额外的去重、排序处理
- Producer的ack也是需要根据场景灵活调整。例如cdc类绝不允许丢失的情况（kafka极端丢失的case除外），可以给-1。针对埋点类数据量庞大的，为了保证吞吐，容忍一部分潜在的丢失风险，可以给1
- Same source只确保流、批作业的original datasource一致，避免传统lambda的方式中，两套作业是完全独立的路径，为之后的数据一致性、补偿机制上带来大量的工作量
- 针对cdc类数据，批处理则需要额外的row number来维护T+1的状态

### Same SQL

即无论是流、批处理，所处理的（submit application）的引擎不同，语义可能也不同，但需要确保业务（加工）逻辑一致

![same sql](https://upload-images.jianshu.io/upload_images/13491351-41f3290268920408.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


- 在主流引擎中，流、批处理统一用一套框架不是很现实。flink对批处理的兼容程度（例如大量的Hive/Spark built-in func不支持，对复杂数据类型支持较弱，读Hive表数据略复杂）不够。Spark对流处理，在state和event time上没有Flink优秀，因此还是lambda的方式，两套引擎对应两种场景
- 尽量避免两套代码的开发，最大程度上直接复用/移植批处理的代码，即Spark sql = Flink sql[+udf]
- 对于窗口聚合等语义不一致的点，需要确保业务逻辑的代码保持一致

### Same DB[option]

即OLAP，OLTP两种模式的兼容

- 终极目标是一套DB可以兼容。既能支持事务，有类cdc的通知机制，亦能支持多范围、多层次、大时间粒度的统计。例如OceanBase、TiDB
- 在实际场景中，大多是数仓ODS或DWD层的DB统一。特点是开发或迁徙成本低，ETL process不复杂，强贴合Hive或Hadoop套件。例如Hudi，或流式写Hive（+presto/impala）
- 实时数仓的DWS因为业务场景、业务需求的不同，几乎都是case by case的方案，很难做到一套框架+DB的统一

### 框架选择

根据上文提到的三个“Same”，框架选择则如下图所示

- 业务库和埋点数据ingest到kafka
- 计算统一使用Flink，且以SQL为主

![框架选择](https://upload-images.jianshu.io/upload_images/13491351-f60a2d3a7216939c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 数仓分层
![数仓分层](https://upload-images.jianshu.io/upload_images/13491351-56beb7350a7a33f0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


# 关键计算逻辑描述

## 计算UV
### 流内构建

#### bitmap
即在流内通过状态维护bitmap，相对于HyperLogLog可以做到精准去重，且资源开销在可接受范围内。如以下的示例：

```java
StateTtlConfig ttlConfig = StateTtlConfig
        .newBuilder(Time.days(2))
        .setUpdateType(StateTtlConfig.UpdateType.OnCreateAndWrite)
        .setStateVisibility(StateTtlConfig.StateVisibility.NeverReturnExpired)
        .build();
 
 
ValueStateDescriptor<Roaring64Bitmap> bitmapDescriptor = new ValueStateDescriptor(
        "Roaring64Bitmap",
        TypeInformation.of(new TypeHint<Roaring64Bitmap>() {
        }));
 
 
 
bitmapDescriptor.enableTimeToLive(ttlConfig);
 
 
 
// ...
if (!bitmap.contains(uid)) {
    bitmap.addLong(uid);
    bitmapState.update(bitmap);
}
```
缺陷在于
1. 纯flink sql的count distinct，state的实现与之不同，需要额外的udf
2. 对于多个key而言，累计开销巨大。例如通常我们计算uv的维度组合不止一个
3. 基于第2点，通常业务需求的时间范围或跨度都较大，例如当日实时uv，累计uv，新用户数等。state在不设置ttl的情况下，会导致cp时间过长，背压，甚至流量雪崩的情况
3. 强依赖于state

#### 将count distinct 转化为 count(1)



### 借助外部存储
通俗而言，就是不借助flink本身的state，而是靠外存来维护bitmap或其他结构，在query层完成最终的统计（或同时在外存中维护统计值）
![bitmap sink hbase](https://upload-images.jianshu.io/upload_images/13491351-b8732b6db608482d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![api merge](https://upload-images.jianshu.io/upload_images/13491351-2419e6e7c7aadb2a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

好处在于flink除IO外存时，没有额外的开销，state也不大，并且因为外存的原因，保证at least once即可

缺陷在于
1. 外存序列化/反序列化的IO开销
2. 同样受多个key的限制
3. 写压力 -> 读压力的转移，query的性能和并发受较大影响

### query下沉
流任务只承担ETL的功能，保证数据不丢失。明细数据灌入例如doris/hudi等组件，在query层完成最终的聚合统计。优势在于解放流任务，将压力最终转移到query。缺陷依然是在大数据量、高并发的情况下，开销和查询性能的影响





## 宽表构建
### 流JOIN
[flink joining](https://nightlies.apache.org/flink/flink-docs-master/docs/dev/datastream/operators/joining/)，即在流中完成join操作。Flink具有多样的join语义，且易用性很高。主要的问题点在于，流内join时（无论是事实join事实，还是事实join维表，都在变化之中），无法判断数据是否全部到达。因为生产环境数据量的原因，不可能做全关联。窗口join则需要考虑数据延迟、是否回补，retrigger的机制等，开发和维护成本急剧上升。因此大多用于对时效性要求较高，一致性要求不强，业务逻辑不复杂的场景

![流join](https://upload-images.jianshu.io/upload_images/13491351-2fd9d3c3daa9b835.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


- 全关联需要考虑大状态的资源开销、故障恢复的成本
- 窗口join需要考虑数据延迟带来的额外成本，比如窗口等待的时长和时效性的取舍，迟到数据的回补等
- 侧流数据没有解决迟到数据的回补，从另一种意义上而言，还会导致额外的数据重复（sink不幂等的话）

### Source Union
多流union，按主键更新不同列，变相实现join的功能。优势在于，流处理仅仅是ETL，逻辑复杂度低。靠sink DB支持upsert和按列更新（或更新非空列）完成功能。但缺陷在于，依然没有解决迟到数据的retrigger（即什么时候算数据全部到达，补充完成，下游可以重新计算），并且一定程度上牺牲了时效性保障一致性
![source union](https://upload-images.jianshu.io/upload_images/13491351-a8fe1692e62664de.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### Sink Union

分两种情况

1.事实流union，在DB不支持按列更新的前提下，在聚合前union case when 的方式聚合计算

![sink union 1](https://upload-images.jianshu.io/upload_images/13491351-ebc5cc81b3c1f520.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


2.事实流关联维表，在聚合完成后，query查询前，点查维表去format维度属性

![sink union 2](https://upload-images.jianshu.io/upload_images/13491351-deb68b6a5f919fd8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 与离线数仓联动和差异
### 数据补偿
在业务需求中，并不全是只依赖或计算增量数据，例如最近7天的订单数，支付金额等，又或者在流任务故障，需要从离线数仓回补时，都需要与存量数据交互。存量数据初始化或数据补偿的流程如图所示

利用Hudi提供的bulk_insert的write.operation（类似hbase的bulkload）将Hive表中的数据load到Hudi
任务结束后，增量任务调起。两者之间的records有一定堆叠，避免数据丢失
![数据补偿](https://upload-images.jianshu.io/upload_images/13491351-10d6cbdbacfece0a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 共享ODS、DIM[option]
将原本小时级/天级的ODS ETL任务改由流式sink，将速度提升至分钟/小时级
- 加速原本T+1才能产出的业务需求
- 确保流和批的数据源一致，除却加工逻辑和语义的不同外，不会有源数据所带来的数据不一致

### 数据延迟、乱序
### 数据丢失
### 维表JOIN
### 对于迟到数据的trigger


# 具体实施
**主旨思想**

1. 对本身有窗口要求的业务需求，对实时敏感程度一般，要求一致性的业务需求，从流 -> 批转化
例如每15分钟/每小时的当天uv，pv，gmv
2. 对原本的T+1输出的业务需求，从天 -> 小时的转化，即流式写ODS，给下游层级做加速
3. 对于无法规避的业务需求，例如对实时敏感程度高，即时trigger类需求，抽象流kafka（dwd逻辑表）
4. 数据流向不宜过长，通常为ODS -> [DWD] -> DWS
4. dws因业务需求各异，没有统一的方案输出


![](https://upload-images.jianshu.io/upload_images/13491351-e951e924a9612c91.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 流DW
### 退维
### ETL
### 公共逻辑下沉


## Hudi的一些建议
### index.type的选择

![index type](https://upload-images.jianshu.io/upload_images/13491351-f06c3879d7115bbc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### table.type的选择
![image.png](https://upload-images.jianshu.io/upload_images/13491351-3ae69a8ecf9a38d2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### index.bootstrap.enabled的潜在问题
- 参数为是否将已存在表中的记录的index load到flink state，默认为false
- 索引导入是一个阻塞过程，在这期间无法完成cp
- index bootstrap由输入数据触发，需要确保每个分区中至少有一条记录
- index bootstrap是并发执行的，可以在日志文件中通过finish loading the index under partition以及Load record form file观察index bootstrap的进度
- 第一个成功的checkpoint表明index bootstrap已完成。 从checkpoint恢复时，不需要再次加载索引
- sink 多个hudi表时，需要将write.index_bootstrap.tasks set 为1，否则会出现部分index丢失的情况。sink 单hudi表时没有这个限制
- exist hudi 表数据量极大时，index导入会失败

### 其他
- cow表强依赖于flink cp，因为写放大的问题，通常建议调大cp的间隔和timeout，避免因cp失败导致的数据无法更新
- flink on hudi不支持[schema evolution](https://hudi.apache.org/docs/schema_evolution/)，字段变更会频繁的变动任务
