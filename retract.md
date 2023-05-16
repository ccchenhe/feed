
为了控制营销成本，实现精准营销，我们需要提供一定的运营规则，或者说用户标签。例如没有支付过的用户，没有发过红包的用户，并对其做优惠券派发、满减折扣等。乍一听好像比较简单，就是统计每个用户，每种业务状态的所有支付订单数

但它存在以下难点：第一是数据量巨大，历史数据有72亿，平增日增200多万；第二是业务状态生命周期长，例如在某些国家，退款流程最长有3个月，一些线下商家业务（比如充话卡）时长从1天到数天不等；第三是因为涉及到营销支出，比如我们不能给不符合要求的用户派券，所以要求强数据一致性和时效性，潜在的也要灵活支持数据补偿。最后一个就是必须要考虑节约开发成本。
![problem](https://github.com/ccchenhe/use-jpgs/blob/master/flink/summary.png?raw=true)


为什么说基于业务状态变化的实时计算统计比较难，根本原因在于append only和change log的不同，以及change log计算的复杂度。比如我们浏览网页，一次动作就是一个事实，一旦发生不可修改，统计相对较容易，这种就是append only。而change log则是消息记录可以变化和修改，只不过是以增量的形式来记录，我们的统计是针对那一时刻的状态的快照，并且还需要额外考虑消息先后顺序的问题，统计相对复杂。



![change log](https://github.com/ccchenhe/use-jpgs/blob/master/flink/changelog.png?raw=true)


针对change log统计值的计算，解决方案通常有两种，最简单的就是在某种DB中存储所有明细数据，按需计算。好处是很轻易的能捕捉和变更业务状态变化，但因为数据量巨大的关系，读写IO就非常慢。并且这种方式相当于double了一份存储，所以成本问题并没有得到解决。
![use mysql](https://github.com/ccchenhe/use-jpgs/blob/master/flink/mysql.png?raw=true)

那第二种方式呢，就是通过流内聚合，输出统计值。flink的retract回撤流能够很好的契合我们的诉求。它将change log拆分成两种消息类型，也就是插入和删除，并且允许在输出流上撤回一些先前输出的结果。如图所示，在更新已存在的记录state3 update state4时，flink将其拆分先删除state3，再插入state4，并更新已经计算过的聚合结果。

![what's retract](https://github.com/ccchenhe/use-jpgs/blob/master/flink/flinkretract.png?raw=true)

那是不是用flink retract这种方案就高枕无忧了呢，其实并没有。flink retract虽然也解决了捕捉业务状态变化，但这种实现是依靠flink自身的state来完成，因此我们依然要在flink state中去维护全量数据。并且因为强依赖flink自身的state，以及只能输出统计值，那么数据补偿和数据初始化就变得非常困难，也就是每次都需要从头开始全量消费。而且最致命的是，这种机制在特殊业务场景下有一定的局限性。我们的优化方案也随之展开。

![rn](https://github.com/ccchenhe/use-jpgs/blob/master/flink/rownumber.png?raw=true)

正如前文提到，虽然数据量比较大，2t多，但业务状态的生命变化周期最多一个多月。那为什么不给状态设置一个过期时间TTL，只处理在业务状态周期里的热数据，减小资源开销呢？主要还是考虑到infra的稳定性不够，比如上游binlog漏发，在回补binlog时，假设补偿的数据在flink状态的TTL外，也就是一条过期记录，那么flink就会把它当作一条新纪录参与运算，数据就失真了。再比如集群下线、集群切换等等，状态在此时并不是完全可用的，那么对整个任务的稳定性和数据一致性就有很大挑战。

![TTL](https://github.com/ccchenhe/use-jpgs/blob/master/flink/ttl.png?raw=true)

那么既然设置TTL不行，可不可以按照经典lambda架构的体系来优化，即实时只计算当天的数据，对应的状态也只保留一天，离线计算历史到昨天的，两个统计值在查询时完成合并。也就是“日切”。这种方案很好的解决了flink大状态的问题，并且数据补偿相对容易，但局限性在于，第一对实时和离线计算的时间边界要求非常高，因为两边都是统计值，实时消费中无法确认消息有没有被离线重复计算，一旦计算的数据有重叠就会失真，比如数据漂移。第二呢就是图中的特殊场景，为了更直观的描述，我们假设存在这种循环更新的案例，即已被统计的历史记录在今天，也就是流计算当中完成了循环更新，比如历史记录state1变更为state2，state2再变更为state1，计算结果也同样会出现偏差。
![use lambda](https://github.com/ccchenhe/use-jpgs/blob/master/flink/lambda.png?raw=true)

无论状态设置TTL，或者lambda流批日切计算导致的数据失真，其实都是因为flink原生retract的局限性。也就是依赖flink自身的状态来判断记录是否需要回撤。以刚才日切的场景为例，在我们的理想状态下，两条update应该被拆分两对delete和insert，成对出现成对消失，最终合并的结果不变。

![ideal](https://github.com/ccchenhe/use-jpgs/blob/master/flink/%E5%B1%80%E9%99%90%E6%80%A71.png?raw=true)


但实际情况呢，流、批计算是分开的。并且流只保存了一天的状态，那么对于第一条update，尽管它的binlog属性是update，因为并不在flink的状态中，所以只会被flink当作新纪录，也就只生成一条insert，从而计算合并的结果产生了偏差。
![actual](https://github.com/ccchenhe/use-jpgs/blob/master/flink/%E5%B1%80%E9%99%90%E6%80%A72.png?raw=true)


我们现在对flink retract的运行机制进行分析，发现它是否参与回撤的根本在于这条记录的RowKind。以官方文档为例，枚举类型update before表示旧值，也就是回撤删除，update after表示新值，代表插入，这是实现change log增删改的关键。而在我们消费的binlog中，以maxwell为例，update类型是携带了更新前后的新旧值，也就是data和old的，结合binlog的特性我们能否自行去构建binlog记录的flink rowkind，从而实现retract呢？因此我们修改了maxwell connector的代码，以update类型的binlog为例，我们将其拆分成新旧值，并赋给其对应的rowkind枚举。从而完成了升级。
![upgrade](https://github.com/ccchenhe/use-jpgs/blob/master/flink/%E9%97%AE%E9%A2%98%E5%88%86%E6%9E%90.png?raw=true)

最终方案如图所示，分为3步。1.在流计算中依然只计算当天的数据，消费binlog时，根据binlog的dml类型，比如insert，update，delete赋给对应的flink rowkind枚举（在这其中update将会额外拆分成before和after两条记录），并同时新增一个新值的update_time，2.流计算根据对应的业务键和新值的update time进行聚合计算，比如用户id，业务状态state，新值的update_time，其实就是以rowkind为依据，依次执行删除和新增。而批处理则计算历史数据，周期调度到hbase中，在最终查询时完成流批计算的合并。
![final](https://github.com/ccchenhe/use-jpgs/blob/master/flink/%E6%9C%80%E7%BB%88%E6%96%B9%E6%A1%88.png?raw=true)

这个方案有两个优化点，第一在消费binlog时我们做了对应的rowkind赋值和拆分。不再依赖flink自身的state来维护记录是否回撤，只是单纯根据rowkind来执行删除或插入。这种方式解决了之前提到的日切中时间边界不好切割，和历史记录在实时计算内循环更新，导致数据失真的问题。

因为只消费今天的binlog记录，故障恢复和数据补偿变得极为容易。并且因为不强依赖flink自身的状态，整个计算开发的成本大幅降低。之前的状态需要维护71亿+230万增量。现在只需要维护最多230*2=460万的增量

![build flag](https://github.com/ccchenhe/use-jpgs/blob/master/flink/%E7%89%B9%E7%82%B91.png?raw=true)
第二点则同样是为了解决时间边界问题，保障对历史记录的修改，能够在今天也就是流计算中得到抵消。我们假设记录存在创建时间ctime，修改时间mtime，mysql在5月12日更新了5月11创建的历史记录，并且被流计算捕捉到。假设以ctime作为聚合，那么这条update变更将因为不是5月12的记录而被舍弃，也就是不执行对应的retract，因此必须要以mtime为聚合条件，在可以和离线的统计结合做合并，从而做到抵消。这套方案适用于大数据量下跨天状态变更的，计数求和的实时计算。

![add new mtime](https://github.com/ccchenhe/use-jpgs/blob/master/flink/%E7%89%B9%E7%82%B92.png?raw=true)