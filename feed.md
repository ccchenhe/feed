# 业务简介
准确而言，Feed是内嵌在购物APP中的一个平台，可通过基于兴趣的社交联系，改善购物过程中的信息探索，提升内容 -> 电商的转化，增加用户粘性
整体对标淘逛、小红书，不过与之不同的是，Feed仍以卖家秀为主，即新品推广、爆品推荐、活动预告，内容类目较为单一，且几乎没有UGC

![某业务overview](https://upload-images.jianshu.io/upload_images/13491351-de8a8e7ae0ea272e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 背景和范围
## 背景
在Feed数仓建设过程中，缺少统一的建设方法论和标准，仅在Infra Team的公共规范（也不是强制性约束）不足以保证数仓模型数据一致性和数据孤岛的存在。由于建模随意，模型设计理论知识参差不齐，为了应对数据需求，研发人员之间缺少建模交流和沟通，同时，也缺少模型建设流程的评审和约束，导致整体数仓模型的成本和效率得不到有效的控制
本方法论旨在为数据研发、数据分析人员提供指导性建模思想和数据全责范围以及数据使用管理方案，降低建设成本，提高数据使用效率

## 范围
所有参与Feed业务的数据研发工程师、数据分析师


# 建模指导思想
## 维度建模
维度模型是数据仓库领域的Ralph Kimball大师所倡导的，是数据仓库工程领域最经典的数据仓库建模方法
维度建模从分析决策的需求出发构建模型，为分析需求服务。因此它重点关注用户如何更快速的完成需求分析，同时具有较好的大规模复杂查询的响应性能。其典型的代表就是星形模型，以及一些特殊场景下使用的雪花模型。其设计主要分为以下几个步骤：
- 选择业务过程：业务过程可以是单个业务事件，例如一篇帖子的曝光。也可以是某个事件的状态，例如当前帖子的状态（活跃，封禁，删除），还可以是一系列相关业务事件组成的业务流程
- 选择粒度：在事件分析中，要预判所有分析需要细分的程度，从而决定选择的粒度。粒度是维度的一个组合
- 选择维度：选择好粒度之后，需要基于此粒度设计维表，包括维度属性，用于分析时进行分组和筛选
- 选择事实：确定分析需要衡量的指标

## 建模基本原则
|序号|基本原则|详细描述|
|--|--|--|
|1|高内聚、低耦合|一个逻辑和物理模型由哪些记录和字段组成，应该遵循最基本的软件设计方法论的高内聚和低耦合原则。主要从数据业务特性和访问特性两个角度来考虑：将业务相近或者相关的数据、粒度相同数据设计为一个逻辑或者物理模型；将高概率同时访问的数据放一起，将低概率同时访问的数据分开存储|
|2|核心模型与扩展模型分离|建立核心模型与扩展模型体系，核心模型包括的字段支持常用核心的业务，扩展模型包括的字段支持个性化或少量应用的需要，不能让扩展字段过度侵入核心模型，破坏核心模型的架构简洁性与可维护性|
|3|公共处理逻辑下沉及单一|越是底层公用的处理逻辑更应该再数据调度依赖的底层进行封装和实现，不要让公共的处理逻辑暴露给应用层实现，不要让公共逻辑在多处同时存在|
|4|成本与性能平衡|适当的数据冗余换取查询和刷新性能，不宜过度冗余和数据复制|
|5|数据可回滚|处理逻辑不变，在不同时间多次运行数据结果确定不变|
|6|数据一致性|相同的字段含义在不同表中字段命名必须相同，必须使用规范定义中的名称|
|7|命名清晰可理解|表命名需清晰、一致，表名需易消费者理解和使用|

# 数仓架构设计
## 名词术语解释
|名词术语|名词解释|备注描述|
|--|--|--|
|数据模型|数据组织和存储方法，它强调从业务、数据存取和使用角度合理存储数据，是对现实世界数据特征的描述||
|业务过程|是指企业的业务活动事件，业务过程是一个不可拆分的行为事件，例如交易的支付，退款等；也可以是某个事件的状态，例如当前的用户基本|如：下单、点击、曝光、访问|
|数据域|面向业务分析，将业务过程或者维度进行抽象的集合，数据域需要抽象提炼，并且长期维护和更新，不轻易变动。在划分数据域时，既能涵盖当前所有的业务需求，又能让新业务的进入时可以被包含进已有的数据域或扩展新的数据域|如：Feed业务，流量域，订单域|
|主题域|通常是联系较为紧密的数据主题的集合，主题是面向数据分析应用，是在较高层次上将数仓系统中的数据综合，归类并进行分析利用的抽象，每一个主题对应一个宏观的分析领域|如：算法主题域|
|指标|用于衡量事物发展程度的单位或方法，也称为度量，指标需要经过加和、平均等汇总计算方式得到，并且是需要在一定的前提下进行汇总计算，如时间、地点、范围，也就是我们常说的统计口径和范围，需要结合不同维度来统计指标|如：用户数、订单数、订单金额|
|指标矩阵|通过对指标维度横向树立，业务过程的度量与维度、维度属性的关联关系，构成纵横相交的指标矩阵||
|总线矩阵|数据域/主题域、业务过程、维度形成交叉点，并做标记，表示业务处理过程与维度相关的矩阵||
|维度|维度是度量的环境，用来反映业务的一类属性，这类属性叫做维度属性，而维度属性的集合构成一个维度。维度可以形成一个维度体系，具备访问和过滤事实的能力，作为维度建模的核心|如：post id，帖子标题|


## 数仓架构图
|层级|层级定义|备注描述|
|--|--|--|
|ODS|属于操作数据层，存放贴源数据，DB系统和日志系统，是直接从业务系统采集过来的最原始数据，包含了所有业务的变更过程，数据粒度也是最细的||
|DWD|数据仓库明细层，面向业务过程，根据维度建模思想，基于业务过程建模的事实明细层，存在业务过程事实明细数据||
|DMA|数据融合轻量级汇总建模，基于业务分析应用对业务过程数据进行融合，实现从业务过程到业务分析主题的过渡，存放轻度汇总数据，该层起到承上启下的重要作用||
|DMT|主题域建模，面向业务，确定分析主题，将DMA层数据进行深度聚合，存放深加工汇总数据||
|DA|面向业务应用与数据产品，结合业务场景，进行个性化指标加工，基于应用的数据组装，以数据集市的形式存放数据||

![分层](https://upload-images.jianshu.io/upload_images/13491351-d36a58ebb21bd642.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 数仓层级设计

### ODS层设计
ODS层是存放从业务系统获取的最原始的数据和埋点日志数据，是数仓上层数据的源数据

**(1)层级功能**

|序号|概述|功能描述|
|--|--|--|
|1|源数据收集|从业务系统获取最原始的数据，或者app中埋点，获取最原始日志数据|
|2|业务隔离|在业务系统和数据仓库之间形成一个隔离层，降低了数据转化的复杂性|
|3|结构化|结构化存储业务历史明细数据，数据灾备|
|4|转移查询|转移一部分业务系统细节查询的功能，降低业务系统查询的压力|

**(2)数据组织形式**

|序号|概述|数据组织形式|
|--|--|--|
|1|贴源数据|表或字段命名基本与源业务系统保持一致，通过额外的标识来区分增量和全量表|

### DWD层设计
以业务过程作为建模驱动，存在业务过程的明细事实数据，将业务划分为单一的业务过程节点，作为建模依据，沉淀数据

**(1)层级功能**

|序号|概述|功能描述|
|--|--|--|
|1|支撑上层|为DM及上层级提供来源明细数据|
|2|历史查询|为未来分析类需求的扩展提供历史数据支持|
|3|异常处理|对事实的null值做统一处理|
|4|维度退化|关联维表，退化维度到事实表，提高事实表的易用性|
|5|公共逻辑下沉|公共逻辑下沉，包括数据标签化，公共复杂逻辑封装等|
|6|数据标准化|数据标准化清晰，将多源数据标准统一，包括代码转换、枚举值定义、复杂字段拆解等|

**(2)数据组织形式**

|序号|概述|数据组织形式|
|--|--|--|
|1|明细数据|基于业务过程沉淀的明细事实数据，包括引用的维度和业务过程有关的度量|
|2|业务过程|业务过程中产生的数值型数据常称为度量，是明细事实表的依据|

### DM层级
DM层为面向业务分析应用数据加工的中间层，是核心数据指标计算层，要满足现有业务核心指标的覆盖，尽可能在这一层进行逻辑加工计算，将DM作为业务主要使用数据的源头，屏蔽上游变更对下游应用和依赖的影响，基于数据量和业务使用数据收口，将DM层进行功能拆分DMA（轻量级汇总层）和DMT（主题深加工汇总层）
#### DMA层设计

**(1)层级功能**

|序号|概述|功能描述|
|--|--|--|
|1|数据融合|结合数据域和主题域，将不同或者相同的数据域下DW层模型融合|
|2|宽表化|面向业务分析，将业务相近，粒度相似的数据进行融合，宽表化处理，支撑OLAP分析|
|3|轻度汇总|实现融合数据的轻度汇总，实现原子指标的生产，提升指标丰富度|
|4|承上启下|实现业务数据化到数据业务化的过渡，降低数据理解和使用成本|
|5|复杂计算|实现复杂的，特殊的业务逻辑处理和加工，缓解上层计算压力（归因等）|


**(2)数据组织形式**

|序号|概述|数据组织形式|
|--|--|--|
|1|数据多维|拥有不同业务过程的维度、维度属性以及原子指标|
|2|最细粒度|存放最细粒度的维度、维度属性数据，宽表化组织|

#### DMT层设计
**(1)层级功能**

|序号|概述|功能描述|
|--|--|--|
|1|主题数据聚合|面向业务分析，进行主题域划分，沉淀相同主题域下不同维度组合的模型|
|2|深度汇总|分时线主题数据的深度汇总，对衍生指标加工和沉淀|
|3|数据复用|结合数据分析和应用场景，提供可复用模型|


**(2)数据组织形式**

|序号|概述|数据组织形式|
|--|--|--|
|1|主题数据|以分析主题为核心，不同维度组合下的衍生指标数据|
|2|深加工数据|存放比较通用的维度组合深加工数据，避免个性化维度组合数据存储|

### DA层设计
面向特定业务应用、数据产品、专题数据分析、数据分析项目、临时分析查数等，结合业务特点构建特定格式组织数据，模型结构以应用便捷和快速为原则，模型设计更多的贴近业务，结合业务场景，个性化指标加工，基于应用的数据组装：如大宽表集市、横表转纵表，趋势指标串等

**(1)层级功能**

|序号|概述|功能描述|
|--|--|--|
|1|数据需求满足|面向应用分析以及响应个性化的数据需求|

**(2)数据组织形式**
|序号|概述|数据组织形式|
|--|--|--|
|1|需求数据|以具体分析需求为核心，不同维度组合下的指标数据（复合与衍生指标为主）|
|2|个性化数据|存放比较个性化维度组合数据，只适合单个业务需求场景（报表，应用）|

### DIM层设计
主要由维度表（维表）构成。维度是逻辑概念，是衡量和观察业务的角度。维表是根据维度及其属性在数据平台上构建的物理化的表，采用宽表设计的原则。
在划分数据域、构建总线矩阵时，需要结合对业务过程的分析定义维度。作为维度建模的核心，在企业级数据仓库中必须保证维度的唯一性。

**(1)层级功能**

|序号|概述|功能描述|
|--|--|--|
|1|维度统一|建立一致数据分析维表，可以降低数据计算口径和算法不统一风险|
|2|缓慢变化|结合维度的变化的特征，构建固定或者缓慢变化的维度表|

**(2)数据组织形式**

|序号|概述|数据组织形式|
|--|--|--|
|1|维度信息|相关业务维度和维度属性的集合，宽表化存储|





# 数仓建设实施过程

## 数据调研
根据业务线情况可分为业务调研和需求调研
- 业务调研：构建大数据数据仓库，需要了解各个业务线的业务有什么异同，以及各个业务现可以细分为哪几个业务模块，每个业务模块具体的业务流程又是怎样的。
- 需求调研：途径有两种：一是与数据产品经理、数据分析师、业务运营人员了解他们的数据诉求；二是对报表系统中现有的报表进行研究分析。
- 调研报告：基于调研结果，需要输出调研报告

梳理出业务线的整体业务架构，各个业务模块之间的联系与信息流动的流程。
梳理出业务线的整体数据框架，各个业务模块中的主要业务功能和数据类型。

![业务流程1](https://upload-images.jianshu.io/upload_images/13491351-53fda315e7ecd6e3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![业务流程2](https://upload-images.jianshu.io/upload_images/13491351-7f556a5f3f7a0e12.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![ER](https://upload-images.jianshu.io/upload_images/13491351-759a6c2b93d3da09.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 域设计
### 业务过程
结合业务线调研报告，确定业务模块/项目以及每个模块中的事件或者动作，抽象出业务过程
![业务过程](https://upload-images.jianshu.io/upload_images/13491351-0530ed9466731f30.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 数据域
划分原则：
1. 面向业务数据，将业务过程或者维度进行抽象的集合
2. 需要长期维护，不轻易变化和频繁修改
3. 数据域必须具有扩展性，新增业务能不影响的扩展或者新增
4. 把业务相近、粒度兼容的维度和度量值进行抽象整合

![数据域](https://upload-images.jianshu.io/upload_images/13491351-e2a33ea17eca3cc7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 主题域
划分原则：
1. 面向数据应用分析，针对具体的业务分析主体，如订单分析，帖子分析
2. 数据具备一定的相关性或者业务相近，突出分析的重点（主题）

## 总线矩阵
明确每个数据域下有哪些业务过程后，即可构建总线矩阵，明确业务过程与哪些维度相关，并定义每个数据域下的业务过程和维度
![总线矩阵](https://upload-images.jianshu.io/upload_images/13491351-5f2da9cbae60ce6e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 模型设计
主要包括维度及属性的规范定义，维表、明细事实表和汇总事实表的模型设计
### 维表设计
基于维度建模理念，建立数据维表，以降低数据计算口径和算法不统一风险

**(1)设计步骤**
- 结合业务，确定维表使用范围，完成维度的初步定义，并保证维度的一致性
- 确定主维表，通常是ODS表，直接与业务系统同步
- 确定相关维表，确定哪些表与主维表存在关联关系，并选择其中的某些表用于生成维度属性
- 确定维度属性，从主维表以及相关维表中选择维度属性或生产新的维度属性

**(2)设计原则**
- 优先使用公共维表，维表设计考虑复用性和一致性
- 维度属性尽量覆盖业务的统计、分析、探查等需求
- 维度属性除编码字段外，还应尽可能包含文字性描述字段，如id和title
- 避免过于频繁的更新维表的数据


### 明细事实表设计
**(1)设计步骤**
- 选择业务过程。结合业务数据情况，可以为每个业务过程建立一个事实表，也可以将多个相近或相似的业务过程建立一个事实表
- 确定粒度。 针对业务过程一个粒度，就确定事实表中每一行所表达的细节层次。保证所有的事实按照同样的细节层次记录。如果有字段可以表达这个粒度，可以定义为事实表的主键。应该尽量选择最细级别的粒度，以确保事实表的应用具有最大的灵活性
- 确定维度。选定好业务过程并确定粒度后，就可以确定维度信息，选择能够描述清楚业务过程的维度信息。
- 确定事实。事实表应该包含与业务过程描述有关的所有事实，且事实的粒度要与所确定的事实表粒度一致
- 冗余维度。确定需要哪些相关维度，进行维度冗余。在事实表中存储各种类型的常用维度信息，减少下游用户使用时关联多个表的操作，减少计算开销，提高使用效率

**(2)设计原则**
- 尽可能包含所有与业务过程相关的事实
- 只选择与业务过程相关的事实
- 在同一个事实表中，不能包含多种不同粒度的事实。事实表中所有事实的粒度需要与表声明的粒度保持一致
- 事实的单位要保持一致
- 对事实的null值要做统一处理

### 汇总事实表设计
**(1)设计步骤**
- 确定汇总的数据域/主题域
- 确定汇总的维度
- 确定汇总的事实

**(2)设计原则**
- 数据公用性，维度和事实尽可能覆盖相关业务使用数据的场景
- 尽量不要在同一个表中存储不同粒度的汇总数据，如有必要，可用分区存储
- 模型复用性，尽可能多的覆盖下游使用数据的场景
- 指标加工范围尽量不包含复合型指标


## 实施过程输出件
### 输出件
1. 调研和梳理环节：调研报告、总线矩阵、指标矩阵
2. 设计环节：模型设计文档
3. 评审环节：评审标准（评审项）
4. 上线环节：上线通告

# 通用设计方法
## 层级调用
**总体原则：**
1. 数据从ODS层开始到DA层结束，ODS-DW-DM(DMA/DMT/DIM)-DA
2. 每一层至多同级调用2次
3. 不允许层级回流调用

**约定细则：**
1. 允许DMA层模型表同层级调用加工生产DMA模型表
2. 允许DIM层 模型表同层级调用加工生产DIM模型表
3. 不建议DA层模型表直接依赖DW层模型表进行加工
4. 不建议DIM层模型表直接依赖DMA层模型表进行加工
5. 不建议DA层同层级调用加工生产DA模型表
6. 不允许DA层模型表数据回流到ODS/DW/DM(DMA/DMT/DIM)层
7. 不允许DMT层模型表数据回流到ODS/DW/DMA/DIM层
8. 不允许DMA层模型表数据回流到ODS/DW层
9. 不允许DW层模型表数据回流到ODS层

![层级调用](https://upload-images.jianshu.io/upload_images/13491351-69db181c5d2a1552.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![层级调用](https://upload-images.jianshu.io/upload_images/13491351-e9402d023554da7c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 退维方法
退维是指在模型物理实现中将各维度的常用属性退化到事实表中，以大大提高对事实表的过滤查询、统计聚合等操作的效率，下游层级模型使用的维度属性数据下沉本层模型中进行，在这里指DW/DMA/DMT/DA层模型中的维度属性下沉，将维度属性从上一层级下沉到1-n层级模型表

**DW层退维：** 将下游DMA/DMT/DA层常规且稳定的属性下沉在该层进行存放，方便使用。减少重复关联维表，需考虑数据回溯计算成本因素，易变动的维度不建议退到该层
**DMT层退维：** 将下游DA层的维度属性退到该层，将能够关联使用的维度尽可能下沉到该层，解决易变动维度问题，灵活应用
**DIM退维：**  将维表做扁平化处理，维度打横。扁平化处理就是将能够整合的维度全部以字段的形式放到一个模型表里，包含易变动维度


## 通用逻辑下沉
1. 复杂的计算逻辑且口径易变动的计算规则下沉到DMA层，稳定的逻辑或者通用标签下沉到DW表
2. 常用where条件规则可以封装成标签字段，下沉逻辑，比如是否点击，是否访问等

## 缓慢变化维
1. 重写维度值。不保留历史数据，始终取最新数据
2. 插入新的维度行。保留历史数据，维度值变化前的事实和过去的维度值关联，维度值变化后的事实和当前的维度值关联
3. 添加维度列

## 数据拆分
1. 在物理上划分核心模型和扩展模型，将其字段进行垂直拆分
2.将访问相关度较高的列在一个表存储，访问相关程度低的字段分开存储
3. 将经常用到的where条件按记录行进行水平切分或冗余。水平切分考虑二级分区字段，以避免多余的数据复制和冗余
4. 将出现大量空值和零值的统计汇总表，依据其空值零值分布状况可以做适当的水平和垂直切分，以减少存储和下游的扫描数据量


## 一致性维度
公共层的维度表中相同维度属性在不同物理表中的字段名称、数据类型、数据内容必须保持一致。除以下情况：
1. 在不同的实际物理表中，如果由于维度角色的差异，需要使用其他的名称，其他名称也必须是规范的维度属性的别名。例如定义一个user id时，如果在一个表中，分别要表示卖家id，买家id，那么设计规范阶段就预先对user id定义卖家和买家id
2. 如果由于历史原因，在暂时不一致的情况下，必须在规范的维度定义一个标准维度属性，不同的物理名也必须是来自标准维度属性的别名

## 维度的组合和拆分
1.组合原则
- 将维度所描述业务相关性强的字段在一个物理维表实现。相关性强是指经常需要一起查询或进行报表展现，两个维度属性间是否存在天然的关系等。例如商品基本属性和品牌属性
- 无相关性的维度可以适当考虑杂项维度（例如交易），可以构建一个交易杂项维度手机交易的特殊标记属性、业务分类等信息。也可以将杂项属性退化在事实表中处理，不过容易造成事实表相对庞大，加工处理较为复杂
- 所谓的行为维度是经过汇总计算的指标，在下游的应用使用时将其当作维度处理。如果有需要，度量指标可以作为行为维度冗余到维度表中

2.拆分与冗余
对于维度属性过多，涉及源较多的维度表（例如会员表），可以做适当拆分：
- 拆分成核心表和扩展表，核心表相对字段较少，刷新产出时间较早，优先使用。扩展表字段较多，且可以冗余核心表部分字段，刷新产出时间较晚，适合数据分析人员使用
- 根据维度属性的业务不相关性，将相关度不大的维度属性拆分成多个物理表存储

数据记录数较大的维度表（例如商品表），可以适当冗余一些子集合，以减少应用的数据扫描量
- 可以根据当天是否有行为，产出一个活跃行为的相关维表，以减少应用的数据扫描量
- 可以根据所属业务扫描数据范围大小的不同，进行适当的子集合冗余

## 事实表设计原则
### 事务型事实表
主要用于分析行为和追踪事件。事务事实表获取业务过程中的事件或者行为细节，然后通过事实与维度之间关联，可以非常方便计算各种事件相关的度量。例如浏览uv
- 基于数据应用需求的分析设计事务型事实表，如果下游存在较大的针对某个业务过程事件的分析指标需求，可以考虑基于某一个事件过程构建事务型事实表
- 事务型事实表一般选用事件发生日期或事件作为分区字段，可以方便下游作业数据扫描执行分区裁剪
- 明细层事实表的冗余子集的原则能有利于降低上层数据访问的IO开销
- 明细层事实表维度退化到事实表原则能有利于减少上层数据访问的join成本

### 周期快照型事实表
主要用于分析状态型或存量型事实。快照是指以预定的时间间隔来采样状态度量

### 累计快照事实表
基于多个业务过程联合分析从而构建的事实表，如采购单的流转环节等。主要用于分析事件之间的时间间隔和周期。例如用交易的支付与发货之间的间隔，来分析发货速度，或在支付和退款环节分析支付退款率等等



# 规范
## 命名规范
### 表命名规范
|层级|规范|示例|
|--|--|--|
|ODS|`ods_{数据库名/kafka topic名}__{表名}_{刷新周期}{存储策略}`|ods_shopee_feed_db__feed_content_tab_df |
|DWD|`{mart_name}_dwd_{业务域}_{业务过程}_{刷新周期}{存储策略}`|feed_mart_dwd_feed_comment_df|
|DM|`{mart_name}_dm_{业务域}_{主体域｜自定义名称}_{聚合粒度}`| feed_mart_dm_feed_interact_events_1d|
|DA|`{mart_name}_da_{业务域}_{主体域｜自定义名称}_{aggregation_window}`|feed_mart_da_app_overview_1d |
|DIM|`{mart_name}_dim_{业务板块}_{扩展维度}`| feed_mart_dim_author_ext |
|临时表|`tmp_{开发者}_{模型表命名规范}`|tmp_ab_feed_mart_dws_app_overview_1d |
|视图|`view_{模型表命名规范}`|view_feed_mart_dws_app_overview_1d |
### 字段命名规范
1. 英文尽量用全程，如果字段太长可以从后向前进行缩写，尽量精简
2. 对于编号作为标识符的属性/列，一般统一命名为“xx编号”的属性/列，后缀应是id，如post_id
3. 取值只有“是/否”的属性/列，中文名前缀必须为“是否”，英文名前缀是is，数据类型为int
4. 日期类型的属性/列，数据为日期类型，后缀应是date，如grass_date。若数据类型为datetime，后缀应是time，如create_time

### 任务命名规范
原则上一个节点对应产出一张表，特殊情况遇到多路产出时，任务名称以后缀数字来区分
1. 任务名、表名建议一致
2. 任务配置中输出项必须维护，如库名.表名


|任务类型|任务描述|命名规范|
|--|--|--|
|数据同步|hive到mysql|h2m_源表名|
|数据同步|mysql到hive|m2h_目标表名|
|数据同步|hive到redis|h2rds_源表名|
|数据同步|hive到mq|h2mq_源表名|
|数据同步|本地文件到redis|f2rds_源文件名|
|数据同步|hbase到mysql|hbs2m_源表名|
|数据同步|mysql到es|m2es_源表名|
|数据同步|hbase到redis|hbs2rds_源表名|
|数据同步|hdfs到tair|dfs2tr_源文件名|
|脚本|shell|语言_功能_xxxxx（实现某个功能的任务）|
|hive|sql|计算结果表|
|spark|sql|计算结果表|
|spark|streaming|fow_sprk2h_计算结果表|
|kylin|cube构建|kyln_cube名（同表名）|



## 生命周期管理规范
**1.临时表生命周期**
参与计算的非分区临时表，默认1天
参与计算的分区临时表，默认3天
临时数据备份表，dev自行清理

**2.ODS表生命周期**
日志流水表，字段未做全部解析，生命周期90天。如果已做全部解析（或小时表），生命周期7天
日志解析表，生命周期3650天
同步任务增量表，生命周期3650天
同步任务全量表，生命周期7天

**3.DW表生命周期**
业务过程明细数据，生命周期548天
非常核心的增量中间表，生命周期3650天
全量数据，生命周期7天

**4.DM表生命周期**
dma，生命周期548天
dmt，生命周期548天
非常核心的增量中间表，生命周期3650天
全量数据，生命周期7天

**5.DA表生命周期**
报表应用，生命周期548天
明细数据，生命周期60天
全量数据，生命周期7天
数据产品应用，明细不超过365天，汇总不超过548天
核心da表，生命周期3650天

**6.超大存储表**
永久保存，考虑冷存储


## 刷新周期规范
|刷新周期|年|半年|季度|月|周|日|小时|分钟|快照|
|--|--|--|--|--|--|--|--|--|--|
|刷新周期标识|y|hy|s|m|w|d|h|mi|c

## 数据类型规范
|MySQL数据类型|Hive数据类型|
|--|--|
|tinyint/smallint/mediumint/integer/bigint|bigint|
|float/double/decimal(非金额类)|double|
|decimal(金额类)|decimal|
|longtext/text/varchar/char|string|
|date|date|
|datetime/timestamp|datetime|


## 存储格式和压缩方式
|数据层级|文件格式|压缩格式|
|--|--|--|
|ods|text file/parquet|lzo/snappy|
|dw|orc|snappy|
|dm|orc|snappy|
|da|orc|snappy|
|dim|orc|snappy|



# 关键逻辑描述
## 停留时长
### 帖子粒度
- 依托用户在浏览滑动帖子时，帖子出现（operation = 'action_post_appear'）和消失（operation = 'action_post_disappear'）,成对相减
- full-srceen video在滑动或退出时，会上报video的播放时长（operation = 'action_video_stop')

**具体步骤：**
1.基于click表筛选页面打点数据page_type=me/shop/my_like/post/feed/feed_explore/hashtag_detail/comment/product_post operation=action_post_appear/action_post_disappear
![step 1](https://upload-images.jianshu.io/upload_images/13491351-4ca17a391a4e3ddb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2.按照session_id、post_id、user_id、device_id、page_type、tab_id、tab_name、tab_location分组 → 根据event_timestamp进行降序排序 → 对临近的action_post_appear和action_post_disappear计算差值 → 得到当前用户当前post的停留时间
![step 2](https://upload-images.jianshu.io/upload_images/13491351-b0ed452d04d7b826.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


3.按照page_type、user_id、tab_id、tab_name、tab_location分组 → 聚合计算总停留时长
![step 3](https://upload-images.jianshu.io/upload_images/13491351-af2493fcaaf51881.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 页面粒度
表示用户在feed页面访问，然后进入pdp，然后返回feed页面，一行代表一个traffic数据。目前逻辑没有将pdp相关的页面（也就是feed场景可以去到的相关页面）纳入进来，只计算了feed场景所有相关的页面，计算出来的停留时长是8s（1+1+4+1+1），如果纳入pdp相关的页面（也就是feed场景可以去到的相关页面），停留时长是5s（1+1+1+1+1），但是feed场景能去到的页面目前难以梳理，目前是按照8s的逻辑来算的，有一定的失真

![页面粒度停留时长示例](https://upload-images.jianshu.io/upload_images/13491351-e96ed1363673ec39.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 总结
|计算方式|资源损耗|执行时长|数据准确性|优势|弊端|适用场景|
|--|--|--|--|--|--|--|
|页面粒度|极大|2-3h|偏大或接近真实数据|1. 数据准确性高|1.无法计算每个帖子的停留时长 2.资源开销大 3.强依赖截止事件的定义（退出post场景）|1.某页面粒度的停留时长 2.大盘类的停留时长|
|帖子粒度|一般|15min|偏小|1.资源开销小，时长短 2.可以计算每篇post时长|1.无法计算帖子封面的时长 2.无法计算跨页面，跨场景的时长|1.计算某帖子的停留时长 2. 某推荐实验的停留时长|

## 归单逻辑
### 总览
|订单类型|定义|备注|优先级|
|--|--|--|--|
|direct|用户[有效浏览post] → click item_tag/item_card事件 → ATC/BuyNow事件/7天内下单|ATC的item_id = click post item_card/item_tag的item_id = order 的item_id|1|
|indirect PDP|用户[有效浏览post] → click item_tag/item_card事件 → Any Actions → 7天内下单|orde中的item_id = click item_tag/item_card的item_id|2|
|indirect Shop|用户[有效浏览post] → click item_tag/item_card/profile/seller_name事件 → Any Actions → 7天内下单|如果是click seller_name/profile，那就是post author的shop_id = order中item_id的shop_id；如果是click item_tag/item_card，那就是order中item_id的shop_id = click item_tag/item_card的item_id的shop_id，并且order中item_id !=不等于 click item_tag/item_card的item_id|3|


![summary](https://upload-images.jianshu.io/upload_images/13491351-afe09f6ba97a55f1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### 直接订单
![两步归单，即加购/checkout前2个事件可以追溯到点击帖子卡片](https://upload-images.jianshu.io/upload_images/13491351-ad8764b152fbb78b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 间接订单
![7天内有点击过帖子卡片的行为](https://upload-images.jianshu.io/upload_images/13491351-7620d65ff9b7233f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![7天内有通过帖子进入到卖家主页，并购买过商品](https://upload-images.jianshu.io/upload_images/13491351-14b7fcb67bd13990.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![7天内点击过帖子的商品卡片，且购买过其中的商品](https://upload-images.jianshu.io/upload_images/13491351-68bd3fb3c743d350.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



## 判断通过帖子关注卖家
1. 取当天新产生的关注记录
2. 取当天有点击过click 关注按钮的记录
3. 对2中的记录按author、viewer、post进行聚合，取event time最大的记录
4. 对1和3中的记录关联，关联键为author，viewer。并且关注的时间要比点击关注按钮的时间要大


![logic](https://upload-images.jianshu.io/upload_images/13491351-52d0b7633c580aa3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![取关后又关注](https://upload-images.jianshu.io/upload_images/13491351-abfca727dbbaa3a4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![关注后又取关的](https://upload-images.jianshu.io/upload_images/13491351-070aaa007eed1fe8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 如何定义浏览一篇帖子
1. 封面的曝光，因为没有内容的露出，所以不算有效访问
2. 帖子内容的曝光，即有内容的露出，或直接访问帖子详情页，算有效访问


# 表物理结构
## ODS
|表名|功能描述|更新频率|数据来源|生命周期|
|--|--|--|--|--|
|feed_mart_ods_traffic_feed_impression_hi|feed业务埋点曝光事件小时增量表|小时增量|埋点SDK，其他团队预处理|7天|
|feed_mart_ods_traffic_feed_click_hi|feed业务埋点点击事件小时增量表|小时增量|埋点SDK，其他团队预处理|7天|
|feed_mart_ods_traffic_feed_view_hi|feed业务埋点访问事件小时增量表|小时增量|埋点SDK，其他团队预处理|7天|
|feed_mart_ods_traffic_atc_buynow_hi|feed业务埋点加购事件小时增量表|小时增量|埋点SDK，其他团队预处理|7天|
|ods_feed_db__comment_reply_like_tab_hf|feed评论点赞表|小时更新|业务团队|无快照，永久|
|ods_feed_db__comment_reply_tab_hf|feed评论回复表|小时更新|业务团队|无快照，永久|
|ods_feed_db__comment_tab_hf|feed评论表|小时更新|业务团队|无快照，永久|
|ods_feed_db__like_tab_hf|feed点赞表|小时更新|业务团队|无快照，永久|
|ods_feed_db__hashtag_tab_hf|hashtag话题明细表|小时更新|业务团队|无快照，永久|
|ods_feed_db__hashtag_feed_tab_hf|feed <-> hashtag关联关系表|小时更新|业务团队|无快照，永久|
|ods_feed_db__feed_meta_tab_hf| feed 基础信息表|小时更新|业务团队|无快照，永久|
|ods_feed_db__feed_content_tab_hf| feed正文/图片 存储表|小时更新|业务团队|无快照，永久|
|ods_feed_db__push_msg_tab_hf| feed 推送表|小时更新|业务团队|无快照，永久|
|ods_feed_db__following_hashtag_tab_hf| user <-> hashtag关联关系表|小时更新|业务团队|无快照，永久|
|ods_feed_db__follow_flow_tab_hf| user关注发贴或作者关联关系表|小时更新|业务团队|无快照，永久|
|ods_feed_db__multitabs_contents_tab_hf|运营人工干预配置表，用于高亮置顶post或hashtag|小时更新|业务团队|无快照，永久|
| ods_mkt__feed_official_accounts_df|官方账户配置表(主要用于带货，导流)|人工配置，天级更新|业务团队|快照，7天|
                                                                                                                                                                                                                                                                                                                                                                         

## DW
### ETL
- 过滤测试user_id
- 对异常user_id的处理（例如游客，user_id is null, 赋default值）
- 细化二级分区。例如基于埋点事件的分析，基础条件大多基于页面（例如某主页，某详情页），为了减少IO，去除ODS的小时分区，细化为以page_type为分区
- 对event_time的处理，从UTC统一转化为当地时区，便于下游使用
- 上报事件中加密的字段解密
- 复杂数据结构打横
- 基于埋点对上下文（用户路径的拼接）
![用户路径](https://upload-images.jianshu.io/upload_images/13491351-2af7ed845c4d71b2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- 处理订单表，分割当日增量和历史全量

### 退维
对于高频聚合、查询、关联的不变化的维度属性，退化到DW，以适度冗余为代价，提升查询效率
- 例如埋点事件中，帖子的曝光。帖子的作者id，帖子的类型（图文，视频...）等高频且稳定的属性下沉到DW

- 对易变动的维度属性，或维度属性较多的情况，下沉外键。例如帖子中提到的商品，埋点中并未上报，由维表关联出商品item_id并冗余，便于后续对商品（例如类目、品牌）的分析查询


### 公共逻辑下沉
前文中所提到，对于统计非常频繁、或统计逻辑较为复杂的指标（例如某帖子的曝光、某帖子的浏览，统计口径涉及到8个页面）
```sql
-- old query

select grass_date,
       count(case when page_type = 'feed' and target_type = 'post' and json_extract_scalar(data, '$.tab_id') <> '2' then user_id
       	          when page_type = 'hashtag_detail' and target_type = 'post' then user_id
       	          when page_type = 'product_post' and target_type = 'post' then user_id
       	          when page_type = 'feed_video' and target_type = 'post' then user_id
       	          when page_type = 'feed_explore' and target_type = 'post' then user_id
       	          when page_type in ('me', 'shop', 'my_like') and page_section[1] = 'post' then user_id
       	          else null end ) as post_impression_pv_1d
from feed.feed_mart_dwd_traffic_feed_impression_di
where tz_type = 'local'
  and grass_region = 'ID'
  and grass_date = date'2021-11-01'
  and operation = 'impression'
group by grass_date;
```
因此，我们通过人工配置维表，拼接SQL的方式，将类似这样的逻辑下沉到DW，减少下游的工作量和避免口径变化时带来的影响
```sql
-- new query

select grass_date,
       count(1) as post_impression_pv_1d
from feed.feed_mart_dwd_traffic_feed_impression_di
where tz_type = 'local'
  and grass_region = 'ID'
  and grass_date = date'2021-11-01'
  and post_impression_type = 1
group by grass_date;

```


![人工配置维表](https://upload-images.jianshu.io/upload_images/13491351-077ada8588ec4d21.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


|下沉逻辑|数据类型|具体含义|
|--|--|--|
|is_view_post|INT|0=no, 1=yes|
|feed_source|INT|0=not belong view post, and source is nothing;1= feed page;2=feed post;3=feed page & feed post|
|post_impression_type|INT|0=not belong to feed post impression;1=post impression;2=post cover impression|
|is_item_impression|INT|0=no,1=yes|
|is_voucher_impression|INT|0=no, 1=yes|
|is_click_item|INT|0=no, 1=yes|
|is_click_shop|INT|0=no, 1=yes|
|is_click_follow|INT|0=no, 1=yes|
|is_click_share|INT|0=no, 1=yes|
|is_click_cover|INT|0=no, 1=yes|
|is_claim_voucher |INT|0=no, 1=yes|


![下沉逻辑](https://upload-images.jianshu.io/upload_images/13491351-6ed5605c67b0d876.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## DM
### viewer粒度轻量聚合表
- 结合数据域和主题域，将不同或相同的数据域下的DWD模型融合
- 面向业务分析，粒度相似的数据进行融合，做宽表处理
- 实现主题数据的深度汇总，对衍生指标的加工和沉淀
- 结合数据分析和应用场景，提供可复用模型

![层级关系](https://upload-images.jianshu.io/upload_images/13491351-b111b4fabd81acd0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![表层级关系](https://upload-images.jianshu.io/upload_images/13491351-8185d9cdda9aa59b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



  
                                                                                                                                                                                                                                                                                                                                             



