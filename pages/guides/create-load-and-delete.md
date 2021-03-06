title=建表、数据导入与删除
date=2018-11-06
type=guide
status=published
~~~~~~

# Doris 建表、数据导入与删除

本文档介绍了 Doris 的建表、数据导入和删除操作、部分工作原理，以及这些操作中可能遇到的问题和解决方法。

本文档中使用的命令，可以通过在 mysql-client 中执行以下 help 命令获得更多帮助：

* `help create table;`
* `help load;`
* `help mini load;`
* `help delete;`
* `help alter table;`

## 基本概念

在 Doris 中，数据都以表（Table）的形式进行逻辑上的描述。

### Row & Column

一张表包括行（Row）和列（Column）。Row 即用户的一行数据。Column 用于描述一行数据中不同的字段。

Column 可以分为两大类：Key 和 Value。从业务角度看，Key 和 Value 可以分别对应维度列和指标列。从聚合模型的角度来说，Key 列相同的行，会聚合成一行。其中 Value 列的聚合方式由用户在建表时指定。关于更多聚合模型的介绍，可以参阅 [Doris 数据模型](https://github.com/apache/incubator-doris/wiki/Data-Model%2C-Rollup-%26-Prefix-Index)。

### Tablet & Partition

在 Doris 的存储引擎中，用户数据被水平划分为若干个数据分片（Tablet，也称作数据分桶）。每个 Tablet 包含若干数据行。各个 Tablet 之间的数据没有交集，并且在物理上是独立存储的。

多个 Tablet 在逻辑上归属于不同的分区（Partition）。一个 Tablet 只属于一个 Partition。而一个 Partition 包含若干个 Tablet。因为 Tablet 在物理上是独立存储的，所以可以视为 Partition 在物理上也是独立。Tablet 是数据移动、复制等操作的最小物理存储单元。

若干个 Partition 组成一个 Table。Partition 可以视为是逻辑上最小的管理单元。数据的导入与删除，都可以或仅能针对一个 Partition 进行。

## 建表 Create Table

Doris 的建表是一个同步命令，命令返回成功，即表示建表成功。

可以通过 `help create table;` 查看更多帮助。

本小节通过一个例子，来介绍 Doris 的建表方式。

```
CREATE TABLE IF NOT EXISTS example_db.expamle_tbl
(
    `user_id` LARGEINT NOT NULL COMMENT "用户id",
    `date` DATE NOT NULL COMMENT "数据灌入日期时间",
    `timestamp` DATETIME NOT NULL COMMENT "数据灌入的时间戳",
    `city` VARCHAR(20) COMMENT "用户所在城市",
    `age` SMALLINT COMMENT "用户年龄",
    `sex` TINYINT COMMENT "用户性别",
    `last_visit_date` DATETIME REPLACE DEFAULT "1970-01-01 00:00:00" COMMENT "用户最后一次访问时间",
    `cost` BIGINT SUM DEFAULT "0" COMMENT "用户总消费",
    `max_dwell_time` INT MAX DEFAULT "0" COMMENT "用户最大停留时间",
    `min_dwell_time` INT MIN DEFAULT "99999" COMMENT "用户最小停留时间",
)
ENGINE=olap
AGGREGATE KEY(`user_id`, `date`, `timestamp`, `city`, `age`, `sex`)
PARTITION BY RANGE(`date`)
(
    PARTITION `p201701` VALUES LESS THAN ("2017-02-01"),
    PARTITION `p201702` VALUES LESS THAN ("2017-03-01"),
    PARTITION `p201703` VALUES LESS THAN ("2017-04-01")
)
DISTRIBUTED BY HASH(`user_id`) BUCKETS 16
PROPERTIES
(
    "replication_num" = "3",
    "storage_medium" = "SSD",
    "storage_cooldown_time" = "2018-01-01 12:00:00"
);

``` 

### 列定义

这里我们只以 AGGREGATE KEY 数据模型为例进行说明。更多数据模型参阅 [Doris 数据模型](https://github.com/apache/incubator-doris/wiki/Data-Model%2C-Rollup-%26-Prefix-Index)。

列的基本类型，可以通过在 mysql-client 中执行 `help create table;` 查看。

AGGREGATE KEY 数据模型中，所有没有指定聚合方式（SUM、REPLACE、MAX、MIN）的列视为 Key 列。而其余则为 Value 列。

定义列时，可参照如下建议：

1. Key 列必须在所有 Value 列之前。
2. 尽量选择整型类型。因为整型类型的计算和查找比较效率远高于字符串。
3. 对于不同长度的整型类型的选择原则，遵循“够用即可”。
4. 对于 VARCHAR 和 STRING 类型的长度，遵循“够用即可”。
5. 所有列的总字节长度（包括 Key 和 Value）不能超过 100KB。

### 数据划分

Doris 支持两层的数据划分。第一层是 Partition，仅支持 Range 的划分方式。第二层是 Bucket（Tablet），仅支持 Hash 的划分方式。

也可以仅使用一层分区。使用一层分区时，只支持 Bucket 划分。

1. Partition
    * Partition 列只能指定一列。且必须为 KEY 列。
    * Partition 的区间界限是左闭右开。比如如上示例，如果想在 p201702 存储所有2月份的数据，则分区值需要输入 "2017-03-01"，即范围为：[2017-02-01, 2017-03-01)。
    * 不论分区列是什么类型，在写分区值时，都需要加双引号。
    * 分区列通常为时间列，以方便的管理新旧数据。
    * 分区数量理论上没有上限。
    * 当不使用 Partition 建表时，系统会自动生成一个和表名同名的，全值范围的 Partition。该Partition对用户不可见，并且不可删改。
    * 这里举例说明，当分区在进行增删操作时，分区范围的变化情况。
        * 如上示例，当建表完成后，会自动生成如下3个分区：
            * p201701: [MIN_VALUE,  2017-02-01)
            * p201702: [2017-02-01, 2017-03-01)
            * p201703: [2017-03-01, 2017-04-01)
        * 当我们增加一个分区 p201705 VALUES LESS THAN ("2017-06-01")，分区结果如下：
            * p201701: [MIN_VALUE,  2017-02-01)
            * p201702: [2017-02-01, 2017-03-01)
            * p201703: [2017-03-01, 2017-04-01)
            * p201705: [2017-04-01, 2017-06-01)
        * 此时我们删除分区 p201703，则分区结果如下：
            * p201701: [MIN_VALUE,  2017-02-01)
            * p201702: [2017-02-01, 2017-03-01)
            * p201705: [2017-04-01, 2017-06-01)
            * 注意到 p201702 和 p201705 的分区范围并没有发生变化，而这两个分区之间，出现了一个空洞：[2017-03-01, 2017-04-01)。即如果导入的数据范围在这个空洞范围内，是如法导入的。
        * 继续删除分区 p201702，分区结果如下：
            * p201701: [MIN_VALUE,  2017-02-01)
            * p201705: [2017-04-01, 2017-06-01)
            * 空洞范围变为：[2017-02-01, 2017-04-01)
        * 现在增加一个分区 p201702new VALUES LESS THAN ("2017-03-01")，分区结果如下：
            * p201701:    [MIN_VALUE,  2017-02-01)
            * p201702new: [2017-02-01, 2017-03-01)
            * p201705:    [2017-04-01, 2017-06-01)
            * 可以看到空洞范围缩小为：[2017-03-01, 2017-04-01)
        * 现在删除分区 p201701，并添加分区 p201612 VALUES LESS THAN ("2017-01-01")，分区结果如下：
            * p201612:    [MIN_VALUE,  2017-01-01)
            * p201702new: [2017-02-01, 2017-03-01)
            * p201705:    [2017-04-01, 2017-06-01) 
            * 即出现了一个新的空洞：[2017-01-01, 2017-02-01)
        * 综上，分区的删除不会改变已存在分区的范围。删除分区可能出现空洞。增加分区时，分区的下界紧接上一个分区的上界。
        * 不可添加范围重叠的分区。

2. Bucket
    * 如果使用了 Partition，则 `DISTRIBUTED ...` 语句描述的是数据在**各个分区内**的划分规则。如果不使用 Partition，则描述的是对整个表的数据的划分规则。
    * 分桶列可以是多列，但必须为 Key 列。分桶列可以和 Partition 列相同或不同。
    * 分桶列的选择，是在**均匀划分数据**和**查询效率**之间的一种权衡。如果选择多个分桶列，则数据分布可能更均衡，但是如果只按照其中某一列作为查询条件时，则需要访问所有的分桶。
    * 分桶的数量理论上没有上限。

3. 关于 Partition 和 Bucket 的数量和数据量的建议。
    * 一个表的 Tablet 总数量等于 (Partition num * Bucket num)。
    * 一个表的 Tablet 数量，在不考虑扩容的情况下，推荐略多于整个集群的磁盘数量。
    * 单个 Tablet 的数据量理论上没有上下界，但建议在 1G - 10G 的范围内。如果单个 Tablet 数据量过小，则数据的聚合效果不佳，且元数据管理压力大。如果数据量过大，则不利于副本的迁移、补齐，且会增加 Schema Change 或者 Rollup 操作失败重试的代价（这些操作失败重试的粒度是 Tablet）。
    * 当 Tablet 的数据量原则和数量原则冲突时，建议优先考虑数据量原则。
    * 在建表时，每个分区的 Bucket 数量统一指定。但是在动态增加分区时（`ADD PARTITION`），可以单独指定新分区的 Bucket 数量。可以利用这个功能方便的应对数据缩小或膨胀。
    * 一个 Partition 的 Bucket 数量一旦指定，不可更改。所以在确定 Bucket 数量时，需要预先考虑集群扩容的情况。比如当前只有 3 台 host，每台 host 有 1 块盘。如果 Bucket 的数量只设置为 3 或更小，那么后期即使再增加机器，也不能提高并发度。

### PROPERTIES

在建表语句的最后 PROPERTIES 中，可以指定以下两个参数：

1. replication_num

    * 每个 Tablet 的副本数量。默认为3，建议保持默认即可。在建表语句中，所有 Partition 中的 Tablet 副本数量统一指定。而在增加新分区时，可以单独指定新分区中 Tablet 的副本数量。
    * 副本数量可以在运行时修改。强烈建议保持奇数。
    * 最大副本数量取决于集群中独立 IP 的数量（注意不是 BE 数量）。Doris 中副本分布的原则是，不允许同一个 Tablet 的副本分布在同一台物理机上，而识别物理机即通过 IP。所以，即使在同一台物理机上部署了 3 个或更多 BE 实例，如果这些 BE 的 IP 相同，则依然只能设置副本数为 1。

2. storage_medium & storage\_cooldown\_time

    * BE 的数据存储目录可以显式的指定为 SSD 或者 HDD（通过 .SSD 或者 .HDD 后缀区分）。建表时，可以统一指定所有 Partition 初始存储的介质。注意，后缀作用是显式指定磁盘介质，而不会检查是否与实际介质类型相符。
    * 默认初始存储介质为 HDD。如果指定为 SSD，则数据初始存放在 SSD 上。
    * 如果没有指定 storage\_cooldown\_time，则默认 7 天后，数据会从 SSD 自动迁移到 HDD 上。如果指定了 storage\_cooldown\_time，则在到达 storage_cooldown_time 时间后，数据才会迁移。
    * 注意，当指定 storage_medium 时，该参数只是一个“尽力而为”的设置。即使集群内没有设置 SSD 存储介质，也不会报错，而是自动存储在可用的数据目录中。同样，如果 SSD 介质不可访问、空间不足，都可能导致数据初始直接存储在其他可用介质上。而数据到期迁移到 HDD 时，如果 HDD 介质不可访问、空间不足，也可能迁移失败（但是会不断尝试）。

### ENGINE

本示例中，ENGINE 的类型是 olap，即默认的 engine 类型。在 Doris 中，只有这个 engine 类型是由 Doris 负责数据管理和存储的。其他 engine 类型，如 mysql、broker，本质上只是对外部其他数据库或系统中的表的映射，以保证 Doris 可以读取这些数据。而 Doris 本身并不创建、管理和存储任何非 olap engine 类型的表和数据。

### 其他

    `IF NOT EXISTS` 表示如果没有创建过该表，则创建。注意这里只判断表名是否存在，而不会判断新建表结构是否与已存在的表结构相同。所以如果存在一个同名但不同构的表，该命令也会返回成功，但并不代表已经创建了新的表和新的结构。
    

## 导入（Load）

通过 `help load;` 或 `help mini load;` 查看更多帮助。

### 基本概念

1. 所有的外部数据源的数据，都是通过 load 的方式导入到 Doris 中的。每一个导入任务称为一个批次。每一批次，会生成一个新的数据版本（version）。关于数据版本的介绍，请参阅 [OLAPEngine文档](https://github.com/apache/incubator-doris/wiki/OLAPEngine)

2. 导入是一个异步命令，导入命令返回成功不代表数据导入成功，只代表导入任务提交成功。之后，需通过 `show load;` 命令来查看导入任务的具体执行情况。

3. 每个导入任务，都有一个在单 database 内部唯一的 label。该 label 可能是用户在导入命令中自定义的名称，也可能是某些命令执行后，系统生成并返回给用户的。通过这个 label，用户可以查看对应导入任务的执行情况。label 的另一个作用，是防止用户重复导入相同的数据。当 label 对应的导入作业状态为 CANCELLED 时，该 label 可以再次被使用。

4. 默认情况下，历史导入作业的完成信息会保留 7 天。可以通过 FE 的参数 `label_keep_max_second` 来修改。过期的历史任务的完成信息会被自动删除。删除后，这些任务中使用过的 label 可以再次被使用。

5. Doris 只会删除处于 FINISHED 或者 CANCELLED 状态的历史导入任务完成信息。不会删除尚未结束的导入任务的信息。

6. Doris 目前不支持如 `update` 或 `insert` 等操作单条数据的 DML 语句。

    > 注：Doris 支持 `insert into xxx select xxx;`，即从可访问的数据源读取数据后插入到指定表中。但实现上，仍是一个 load 任务，执行后，会由系统产生一个 label，通过 `show load where label='xxx';` 可以查看进度。
    
### Pull Load & Mini Load

1. Pull Load

    Pull Load 是通过部署的 Broker 进程来读取外部数据导入 Doris 的。用户通过 mysql-client 提交 load 命令。FE 收到请求后，直接生成 load 作业并开始调度。Broker 进程仅仅负责从外部数据源（如 hdfs、baidu object storage 等）读取数据并传递给 BE 进程。之后的 ETL 以及 LOADING 等工作都由 BE 完成。

    在 Pull Load 的命令中使用通配符，匹配多个文件。但是通配符必须匹配到文件而不能是目录。

    Pull Load 每批次的导入数据量受限于多台 BE 的 ETL 处理能力。对于非压缩的导入文件，导入作业会分散到多个 BE 执行，每个 BE 负责处理一部分数据。而对于压缩文件，可能因为无法拆分，只能通过一个 BE 进行处理。

2. Mini Load

    Mini Load 是通过 http 请求，推送本地文件到 Doris 进行导入的。实现上，用户通过 http 请求连接到 FE，FE 通过 http redirect 将请求转发到某一个 BE，然后用户的本地数据文件直接推送到这个 BE 上。接收文件是一个同步的过程。当 BE 接收完文件后，会向用户返回作业提交成功，并向 FE 汇报情况，由 FE 生成 load 作业开始调度。

        目前是用 Mini Load 的方式导入，需要保证提交导入请求的机器和所有 BE 所在节点的机器，网络互通。否则提交将会失败。

    一个 Mini Load 导入受限于单 BE 的 ETL 处理能力。我们限制一批次的导入文件在 1.5GB 以下。

3. 几点说明

    * 两种导入任务，都支持指定导入源文件中列和对应的表中的列的对应关系。所以无需调整源数据文件中列的顺序和 Doris 中表的列顺序的对应关系。

### 导入任务调度

Doris 是被设计为面向批量导入场景的数据库。在这种场景下，导入作业提交的频率通常比较低。最初，Doris 设计为只保证 5 分钟一次批次的导入频率和延迟。后来加入 Mini Load 方式后，在每批次数据量较小的情况下，可以满足支持 5秒钟 一批次的 Mini Load 导入频率。

> 注：高频率的导入，会占用更多的 BE 内存和计算资源，以及更多 FE 的内存资源。

在代码实现角度，FE 有专门的线程，默认每 5秒钟 轮询一次导入作业，进行相应的处理和状态修改。因为一个导入作业要经理 PENDING->ETL->LOADING->FINISHED 四个阶段，所以理论上，一个导入作业，最少要经过 15秒（即3个轮询间隔）才能完成。可以通过修改 FE 配置参数 `load_checker_interval_second` 降低轮询间隔以期更快的导入。但是通常不建议调整这个数值，因为过高的导入频率会有导致 FE 内存暴涨的风险（导入作业堆积、历史作业过多等原因）。

在 ETL 阶段，所有已提交的导入任务都是并发执行的。即所有已提交的导入任务，都会立即开始 ETL 阶段的工作，而不会等待之前提交的导入作业的进度如何。但是对于同一张表，Loading 阶段是串行的。即对于**同一张表**，一个导入作业，必须等待前一个作业的 LOADING 阶段结束，才可以由 ETL 转入 LOADING 状态。而不同表之间的 LOADING 可以并行执行。

> 注：LOADING 阶段的并行执行特性正在开发中，预计2018年1月份 release。

### 导入超时

默认情况下，Pull Load 的导入超时是 14400 秒，而 Mini Load 的导入超时是 3600 秒。当导入作业超时后，会被自动 Cancel 掉。

用户可以在导入命令中自行设置每个导入作业的超时时间。

**强烈建议**用户设置导入超时时间在一个合理的范围内。如果设置的超时时间过长，可能会导致大量导入任务堆积而耗尽 FE 的内存。

### QUORUM\_FINIHSED

导入作业的 QUORUM\_FINIHSED 是对普通用户隐藏的一个状态。

一个完整的导入作业状态转换过程为：
`PENDING->ETL->LOADING->QUORUM_FINISHED->FINISHED`

QUORUM\_FINIHSED 表示对于导入作业对应的表（或分区）中，所有数据分片的**多数副本**都已经完成导入。引入该状态的目的，在于保证当副本缺失（如 BE 宕机）的情况下，不影响导入的进行。

对于普通用户来说，QUORUM\_FINIHSED 对外显示就是 FINISHED。表示该导入已经完成并无法取消。

用户可以通过 `show load where state='quorum_finished';` 语句来查看所有处于 QUORUM\_FINIHSED 状态的导入任务（但显示结果依然为 FINISHED）；也可以通过 FE web 页面 `jobs` 里，看到所有 QUORUM\_FINIHSED 状态的作业。

从系统角度看，QUORUM\_FINIHSED 状态的导入作业对于用户来说是 “**完成的**”，但是对于系统来说是 “**未完成、但一定会完成**” 的状态。处于 QUORUM\_FINIHSED 状态的作业会一直尝试完成所有副本的导入后，才会进入 FINISHED 状态（有时会自动借助副本恢复流程来完成）。如果无法进入 FINISHED 状态，则会一直驻留在内存中，即使超过 Timeout 时间，也不会取消或结束。

QUORUM\_FINIHSED 的一个潜在风险，即大量该状态的作业驻留在内存中导致 FE 内存耗尽。在本文最后的常见问题中，有相关风险的描述和解决方案。

## DELETE 删除数据

Doris 目前可以通过两种方式删除数据：`DELETE FROM` 语句和 `ALTER TABLE DROP PARTITION` 语句。

### DELETE FROM Statement（条件删除）

`delete from` 语句类似标准 delete 语法，具体使用可以查看 `help delete;` 帮助。这里主要说明一些注意事项。

1. 该语句只能针对 Partition 级别进行删除。如果一个表有多个 partition 含有需要删除的数据，则需要执行多次针对不同 Partition 的 delete 语句。而如果是没有使用 Partition 的表，partition 的名称即表名。

2. where 后面的条件谓词只能针对 Key 列，并且谓词之间，只能通过 AND 连接。如果想实现 OR 的语义，需要执行多条 delete。

3. delete 是一个同步命令，命令返回即表示执行成功。

4. 从代码实现角度，delete 是一种特殊的导入操作。该命令所导入的内容，也是一个新的数据版本，只是该版本中只包含命令中指定的删除条件。在实际执行查询时，会根据这些条件进行查询时过滤。所以，**不建议大量频繁使用 delete 命令**，因为这可能导致查询效率降低。

5. 数据的真正删除是在 BE 进行数据 Compaction 时进行的。所以执行完 delete 命令后，并不会立即释放磁盘空间。

6. delete 命令一个较强的限制条件是，在执行该命令时，对应的表，不能有正在进行的导入任务（包括 PENDING、ETL、LOADING）。而如果有 QUORUM\_FINISHED 状态的导入任务，则可能可以执行。

7. delete 也有一个隐含的类似 QUORUM\_FINISHED 的状态。即如果 delete 只在多数副本上完成了，也会返回用户成功。但是会在后台生成一个异步的 delete job（Async Delete Job），来继续完成对剩余副本的删除操作。如果此时通过 `show delete` 命令，可以看到这种任务在 state 一栏会显示 QUORUM\_FINISHED。

### DROP PARTITION Statement（删除分区）

该命令可以直接删除指定的分区。因为 Partition 是逻辑上最小的数据管理单元，所以使用 `DROP PARTITION` 命令可以很轻量的完成数据删除工作。并且该命令不受 load 以及任何其他操作的限制，同时不会影响查询效率。是**比较推荐的一种数据删除方式**。

该命令是同步命令，执行成功即生效。而后台数据真正删除的时间可能会延迟10分钟左右。

## 常见问题

### 建表操作常见问题

1. 如果在较长的建表语句中出现语法错误，可能会出现语法错误提示不全的现象。这里罗列可能的语法错误供手动纠错：
    * 语法结构错误。请仔细阅读 `help create table;`，检查相关语法结构。
    * 保留字。当用户自定义名称遇到保留字时，需要用 `` 符号引起来。建议所有自定义名称使用这个符号引起来。
    * 中文字符或全角字符。非 utf8 编码的中文字符，或隐藏的全角字符（空格，标点等）会导致语法错误。建议使用带有显示不可见字符的文本编辑器进行检查。

2. `Failed to create partition [xxx] . Timeout`

    Doris 建表是按照 Partition 粒度依次创建的。当一个 Partition 创建失败时，可能会报这个错误。即使不使用 Partition，当建表出现问题时，也会报 `Failed to create partition`，因为如前文所述，Doris 会为没有指定 Partition 的表创建一个不可更改的默认的 Partition。
    
    当遇到这个错误是，通常是 BE 在创建数据分片时遇到了问题。可以参照以下步骤排查：
    
    1. 在 fe.log 中，查找对应时间点的 `Failed to create partition` 日志。在该日志中，会出现一系列类似 `{10001-10010}` 字样的数字对儿。数字对儿的第一个数字表示 Backend ID，第二个数字表示 Tablet ID。如上这个数字对，表示 ID 为 10001 的 Backend 上，创建 ID 为 10010 的 Tablet 失败了。
    2. 前往对应 Backend 的 be.INFO 日志，查找对应时间段内，tablet id 相关的日志，可以找到错误信息。
    3. 以下罗列一些常见的 tablet 创建失败错误，包括但不限于：
        * BE 没有收到相关 task，此时无法在 be.INFO 中找到 tablet id 相关日志。或者 BE 创建成功，但汇报失败。以上问题，请参阅 [Doris 部署与升级文档](https://github.com/apache/incubator-doris/wiki/Doris-Deploy-%26-Upgrade) 检查 FE 和 BE 的连通性。
        * 预分配内存失败。可能是表中一行的字节长度超过了 100KB。
        * `Too many open files`。打开的文件句柄数超过了 Linux 系统限制。需修改 Linux 系统的句柄数限制。

    也可以通过在 fe.conf 中设置 tablet_create_timeout_second=xxx 来延长超时时间。默认是1秒，单位是秒。

3. 建表命令长时间不返回结果。

    Doris 的建表命令是同步命令。该命令的超时时间目前设置的比较简单，即（tablet num * replication num）秒。如果创建较多的数据分片，并且其中有分片创建失败，则可能导致等待较长超时后，才会返回错误。
    
    正常情况下，建表语句会在几秒或十几秒内返回。如果超过一分钟，建议直接取消掉这个操作，前往 FE 或 BE 的日志查看相关错误。
    
### 导入操作常见问题

1. `Same label xxx already used`

    使用了与已存在的导入作业相同的 label。label 必须在一个 database 内唯一。过期的 label 被自动删除、或者 label 对应的作业状态为 Cancelled，则可以重复使用。Doris 不支持手动删除 label。

2. `ETL_QUALITY_UNSATISFIED; msg:quality not good enough to cancel`

    数据质量出现问题。可以点击 `show load` 结果最后一列的 URL，查看具体的错误信息。可以参阅 [Doris FAQ](https://github.com/apache/incubator-doris/wiki/Doris-FAQ) 查看更多帮助。
    
3. `Submit etl job failed.`

    当导入作业尝试进入 ETL 状态时失败，具体失败原因，需要查看 fe.log 中，对应 load job id 相关的错误日志。
    
4. 导入作业堆积

    作业堆积通常有三种情况：
    
    1. ETL 阶段作业堆积（不常见）。通常现象是，所有提交的导入作业，ETL 的进度一直为 0%。这种情况通常是因为1）作业提交 ETL task 失败，如配置的 broker 参数不正确，FE 和 BE 的连通性不正常等。2）提交 ETL task 的线程被占用。FE 内部默认有 3 个线程负责提交 ELT task。在某些情况下，如网络问题，导致线程被长时间占用。以上两种情况，通常都可以通过查看 fe.log 发现相关异常日志。
    
    2. LOADING 阶段阻塞（可能出现）。通常现象是，大部分提交的导入作业，ETL 的进度为 100%，但是 LOAD 进度一直为 0%。由前文描述，同一个表的 LOADING 是串行的。所以这种情况，通常是因为同一张表的某个导入作业，卡在了 LOADING 阶段（通常是 BE 在下载 ETL 生成的文件时出现了问题）。可以通过 `show load where state='loading';` 来查找卡住的导入作业，获得 job id。然后通过 FE web 页面的 `jobs` 进入相关页面，查看具体哪些 BE 上的 LOADING 没有完成，然后前往对应 BE 的 be.INFO 中，查看相关日志。当然，通过直接 cancel 掉卡主的导入作业并重新提交，通常也可以解决问题。

    3. QUORUM\_FINIHSED 阶段堆积。通常现象是，可以看到大量 QUORUM\_FINIHSED 状态的导入作业。该问题通常是因为某一台 BE 宕机或磁盘损坏，导致副本缺失。而副本恢复机制还没有及时对这些副本进行补齐。Doris 设计为当一个导入作业在处于 QUORUM\_FINIHSED 状态长达 4 小时后，会强制启动对应副本的恢复流程，来减少作业堆积。但如果导入作业提交频率很高，可能需要手动调整副本恢复的参数以期更快的进行副本恢复。

5. 导入占用内存过高（查询同样适用）

    在进行导入时，可能会出现 `Memory limit exceeded` 的错误。因为目前导入在 ETL 阶段的计算操作全部在 BE 的内存中完成。而在数据内存格式上，不同的数据类型实际占用的内存空间不同。对于 VARCHAR 类型，每一数值都会有**长度**和**指针**两个数据，本身就占用了 16B。所以如果有较多的较短的 VARCHAR 列，则可能出现导入过程的实际内存占用远大于源数据文件大小的情况。Decimal 类型的列也同样存在这样的问题（Decimal 目前版本，在内存中占用 40B）。该情况在查询是类似，而实际的磁盘占用不会出现这个问题。

    当出现这种情况时，可以通过调节导入所能使用的内存尝试解决。连接到 Doris 后，通过 `show variables;` 命令可以查看到 `exec_mem_limit` 这个参数，默认是 2G，单位是字节。该参数表示一次导入任务（或一次查询），在单个 BE 上所能使用的内存上限。可以通过以下语句设置：

    `set exec_mem_limit = xxxxx;`

    `set global exec_mem_limit = xxxxx;`

    因为 exec_mem_limit 是一个 session 级别的变量。第一个语句，只在本次连接 session 中有效。连接断开则失效。第二个语句可以设置永久值，设置好后，重连即可永久生效。当出现 `Memory limit exceeded` 错误时，可以尝试修改这个变量，如修改为 4G 或 8G。然后重试查询或导入。

6. 使用 Broker 进行导入，任务长时间处于 ETL 阶段

    该问题可能的原因是导入期间，有 BE 进程挂掉。导致该 BE 上执行的 load 子任务处于未知状态。而 FE 只能等待超时才会进行下一步处理。所以**强烈建议**设置每一个导入任务的超时时间，并且将超时时间设置为合理范围，以保证导入能够在预期时间内，到达最终状态。

7. 使用 insert into select 方式导入数据，select 有数据，但是导入数据条目比 select 的少，或者显示 `all partitions have no load data` 错误。

    目前 insert into 存在这样一个问题：当 select 出来的结果，列格式不满足目的表列结构的话（比如 varchar 类型长度超长，int 类型给事不匹配等等），这些数据行会被直接过滤掉并且没有错误提示。所以如果遇到以上情况，建议先缩小 select 结果集，排查问题后（目前只能人工排查），再重新导入尝试。

### 删除操作常见问题

1. 执行 delete 命令后，依然可以查询到数据。

    该问题通常可能是：1）delete 命令中指定的 Partition 不对，不包含对应的删除条件的数据。2）where 后面的条件格式不正确。如删除整型时，条件值添加了引号；或删除日期数据时，没有添加引号等。
