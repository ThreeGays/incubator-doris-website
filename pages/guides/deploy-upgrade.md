title=部署和升级
date=2018-11-06
type=guide
status=published
~~~~~~

# Doris 部署和升级文档

该文档主要介绍了部署 Doris 所需软硬件环境、建议的部署方式、集群扩容缩容、升级以及集群搭建到运行过程中的常见问题。  
在阅读本文档前，**强烈建议**先阅读 [安装编译文档](https://github.com/apache/incubator-doris/wiki/Doris-Install)

## 软硬件需求

### 概述

Doris 作为一款开源的 MPP 架构 OLAP 数据库，能够运行在绝大多数主流的商用服务器上。为了能够充分运用 MPP 架构的并发优势，以及 Doris 的高可用特性，我们建议 Doris 的部署遵循以下需求：

#### Linux 操作系统版本需求

| Linux 系统 | 版本 | 
|---|---|
| CentOS | 7.1 及以上 |
| Ubuntu | 16.04 及以上 |

#### 软件需求

| 软件 | 版本 | 
|---|---|
| Java | 1.8 及以上 |
| GCC  | 4.8.2 及以上 |

#### 开发测试环境

| 模块 | CPU | 内存 | 磁盘 | 网络 | 实例数量 |
|---|---|---|---|---|---|
| Frontend | 8核+ | 8GB+ | SSD 或 SATA，10GB+ * | 千兆网卡 | 1 |
| Backend | 8核+ | 16GB+ | SSD 或 SATA，50GB+ * | 千兆网卡 | 1-3 * |

#### 生产环境

| 模块 | CPU | 内存 | 磁盘 | 网络 | 实例数量（最低要求） |
|---|---|---|---|---|---|
| Frontend | 16核+ | 64GB+ | SSD 或 RAID 卡，100GB+ * | 万兆网卡 | 1-5 * |
| Backend | 16核+ | 64GB+ | SSD 或 SATA，100G+ * | 万兆网卡 | 10-100 * |

> 注1：  
> 1. FE 的磁盘空间主要用于存储元数据，包括日志和 image。通常从几百 MB 到几个 GB 不等。  
> 2. BE 的磁盘空间主要用于存放用户数据，总磁盘空间按用户总数据量 * 3（3副本）计算，然后再预留额外 40% 的空间用作后台 compaction 以及一些中间数据的存放。  
> 3. 一台机器上可以部署多个 BE 实例，但是**只能部署一个 FE**。如果需要 3 副本数据，那么至少需要 3 台机器各部署一个 BE 实例（而不是1台机器部署3个BE实例）。**多个FE所在服务器的时钟必须保持一致（允许最多5秒的时钟偏差）**
> 4. 测试环境也可以仅适用一个 BE 进行测试。实际生产环境，BE 实例数量直接决定了整体查询延迟。

> 注2：FE 节点的数量
> 1. FE 角色分为 Follower 和 Observer，（Leader 为 Follower 组中选举出来的一种角色，以下统称 Follower，具体含义见[元数据设计文档](https://github.com/apache/incubator-doris/wiki/Metadata-Design)）
> 2. FE 节点数据至少为1（1 个 Follower）。当部署 1 个 Follower 和 1 个 Observer 时，可以实现读高可用。当部署 3 个 Follower 时，可以实现读写高可用（HA）。
> 3. Follower 的数量**必须**为奇数，Observer 数量随意。
> 4. 根据以往经验，当集群可用性要求很高是（比如提供在线业务），可以部署 3 个 Follower 和 1-3 个 Observer。如果是离线业务，建议部署 1 个 Follower 和 1-3 个 Observer。

* **通常我们建议 10 ~ 100 台左右的机器，来充分发挥 Doris 的性能（其中 3 台部署 FE（HA），剩余的部署 BE）**  
* **当然，Doris的性能与节点数量及配置正相关。在最少4台机器（一台 FE，三台 BE，其中一台 BE 混部一个 Observer FE 提供元数据备份），以及较低配置的情况下，依然可以平稳的运行 Doris。**  
* **不建议 FE 和 BE 混部，可能会造成资源竞争。**

#### Broker 部署

Broker 是用于访问外部数据源（如 hdfs）的进程。通常，在每台机器上部署一个 broker 实例即可。

#### 网络需求

Doris 各个实例直接通过网络进行通讯。以下表格展示了所有需要的端口

| 实例名称 | 端口名称 | 默认端口 | 通讯方向 | 说明 | 
|---|---|---|---| ---|
| BE | be_port | 9060 | FE --> BE | BE 上 thrift server 的端口，用于接收来自 FE 的请求 |
| BE | be\_rpc_port | 9070 | BE <--> BE | BE 之间 rpc 使用的端口 |
| BE | webserver_port | 8040 | BE <--> BE | BE 上的 http server 的端口 |
| BE | heartbeat\_service_port | 9050 | FE --> BE | BE 上心跳服务端口（thrift），用户接收来自 FE 的心跳 |
| FE | http_port * | 8030 | FE <--> FE，用户 |FE 上的 http server 端口 |
| FE | rpc_port | 9020 | BE --> FE, FE <--> FE | FE 上的 thrift server 端口 |
| FE | query_port | 9030 | 用户 | FE 上的 mysql server 端口 |
| FE | edit\_log_port | 9010 | FE <--> FE | FE 上的 bdbje 之间通信用的端口 |
| Broker | broker\_ipc_port | 8000 | FE --> Broker, BE --> Broker | Broker 上的 thrift server，用于接收请求 |

> 注：  
> 1. 当部署多个 FE 实例时，要保证 FE 的 http_port 配置相同。  
> 2. 部署前请确保各个端口在应有方向上的访问权限。

#### IP 绑定

因为有多网卡的存在，或因为安装过 docker 等环境导致的虚拟网卡的存在，同一个主机可能存在多个不同的 ip。当前 Doris 并不能自动识别可用 IP。所以当遇到部署主机上有多个 IP 时，必须通过 priority\_networks 配置项来强制指定正确的 IP。

priority\_networks 是 FE 和 BE 都有的一个配置，配置项需写在 fe.conf 和 be.conf 中。该配置项用于在 FE 或 BE 启动时，告诉进程应该绑定哪个IP。示例如下：

`priority_networks=10.1.3.0/24`

这是一种 [CIDR](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing) 的表示方法。FE 或 BE 会根据这个配置项来寻找匹配的IP，作为自己的 localIP。

**注意**：当配置完 priority\_networks 并启动 FE 或 BE 后，只是保证了 FE 或 BE 自身的 IP 进行了正确的绑定。而在使用 ADD BACKEND 或 ADD FRONTEND 语句中，也需要指定和 priority\_networks 配置匹配的 IP，否则集群无法建立。举例：

BE 的配置为：`priority_networks=10.1.3.0/24`

但是在 ADD BACKEND 时使用的是：`ALTER SYSTEM ADD BACKEND "192.168.0.1:9050";` 

则 FE 和 BE 将无法正常通信。

这时，必须 DROP 掉这个添加错误的 BE，重新使用正确的 IP 执行 ADD BACKEND。

FE 同理。

BROKER 当前没有，也不需要 priority\_networks 这个选项。Broker 的服务默认绑定在 0.0.0.0 上。只需在 ADD BROKER 时，执行正确可访问的 BROKER IP 即可。

## 集群部署

### 手动部署

手动部署请参阅 [安装编译文档](https://github.com/apache/incubator-doris/wiki/Doris-Install)

**注：在生产环境中，所有实例都应使用守护进程启动，以保证进程退出后，会被自动拉起，如 [Supervisor](http://supervisord.org/)。如需使用守护进程启动，需要修改各个 start_xx.sh 脚本，去掉最后的 & 符号**

### Kubernetes 部署
（TODO）

## 扩容缩容

Doris 可以很方便的扩容和缩容 FE、BE、Broker 实例。

### FE 扩容和缩容

用户可以通过 mysql 客户端登陆 Master FE。通过:

```SHOW PROC '/frontends';```

来查看当前 FE 的节点情况。

也可以通过前端页面连接：```http://fe_hostname:fe_http_port/frontend``` 或者 ```http://fe_hostname:fe_http_port/system?path=//frontends``` 来查看 FE 节点的情况。

以上方式，都需要 Doris 的 root 用户权限。

FE 节点的扩容和缩容过程，不影响当前系统运行。

#### 增加 FE 节点

Doris 支持增加两种 FE 节点：Follower 和 Observer。具体说明见：
[Doris 元数据设计文档](https://github.com/apache/incubator-doris/wiki/Metadata-Design)

FE 节点的增加方式同 [安装编译文档](https://github.com/apache/incubator-doris/wiki/Doris-Install) 中 **FE 高可用** 一节中的方式相同。

> FE 扩容注意事项：  
> 1. Follower FE（包括 Master）的数量必须为奇数，建议最多部署 3 个组成高可用（HA）模式即可。  
> 2. 当 FE 处于高可用部署时（1个Master，2个Follower），我们建议通过增加 Observer FE 来扩展 FE 的读服务能力。当然也可以继续增加 Follower FE，但几乎是不必要的。  
> 3. 通常一个 FE 节点可以应对 10-20 台 BE 节点。建议总的 FE 节点数量在 10 个以下。而通常 3 个即可满足绝大部分需求。  

#### 删除 FE 节点

使用以下命令删除对应的 FE 节点：

```ALTER SYSTEM DROP FOLLOWER[OBSERVER] "fe_host:edit_log_port";```

> FE 缩容注意事项：  
> 1. 删除 Follower FE 时，确保最终剩余的 Follower（包括 Master）节点为奇数。  
> 2. 当删除一个 FE 后，不能再使用相同的 fe_host:edit_log_port 添加 FE。新的 FE 必须使用新的 ip 或者新的 edit_log_port。

### BE 扩容缩容

用户可以通过 mysql 客户端登陆 Master FE。通过:

```SHOW PROC '/backends';```

来查看当前 BE 的节点情况。

也可以通过前端页面连接：```http://fe_hostname:fe_http_port/backend``` 或者 ```http://fe_hostname:fe_http_port/system?path=//backends``` 来查看 BE 节点的情况。

以上方式，都需要 Doris 的 root 用户权限。

BE 节点的扩容和缩容过程，不影响当前系统运行以及正在执行的任务，并且不会影响当前系统的性能。数据均衡会自动进行。根据集群现有数据量的大小，集群会在几个小时到1天不等的时间内，恢复到负载均衡的状态。集群负载情况，可以参见 **前端页面** 一节中， **system** 小节的 **backends** 介绍。

#### 增加 BE 节点

BE 节点的增加方式同 [安装编译文档](https://github.com/apache/incubator-doris/wiki/Doris-Install) 中 **多 BE 部署** 一节中的方式相同。

> BE 扩容注意事项：  
> 1. BE 扩容后，Doris 会自动根据负载情况，进行数据均衡，期间不影响使用。

#### 删除 BE 节点

删除 BE 节点有两种方式：DROP 和 DECOMMISSION

DROP 语句如下：

```ALTER SYSTEM DROP BACKEND "be_host:be_heartbeat_service_port";```

**注意：DROP BACKEND 会直接删除该 BE，并且其上的数据将不能再恢复！！！所以我们强烈不推荐使用 DROP BACKEND 这种方式删除 BE 节点。当你使用这个语句时，会有对应的防误操作提示。**

DECOMMISSION 语句如下：

```ALTER SYSTEM DECOMMISSION BACKEND "be_host:be_heartbeat_service_port";```

> DECOMMISSION 命令说明：  
> 1. 该命令用于安全删除 BE 节点。命令下发后，Doris 会尝试将该 BE 上的数据向其他 BE 节点迁移，当所有数据都迁移完成后，Doris 会自动删除该节点。  
> 2. 该命令是一个异步操作。执行后，可以通过 ```SHOW PROC '/backends';``` 看到该 BE 节点的 isDecommission 状态为 true。表示该节点正在进行下线。  
> 3. 该命令**不一定执行成功**。比如剩余 BE 存储空间不足以容纳下线 BE 上的数据，或者剩余机器数量不满足最小副本数时，该命令都无法完成，并且 BE 会一直处于 isDecommission 为 true 的状态。  
> 4. DECOMMISSION 的进度，可以通过 ```SHOW PROC '/backends';``` 中的 TabletNum 查看，如果正在进行，TabletNum 将不断减少。  
> 5. 该操作可以通过:  
> 		```CANCEL ALTER SYSTEM DECOMMISSION BACKEND "be_host:be_heartbeat_service_port";```  
> 	命令取消。取消后，该 BE 上的数据将维持当前剩余的数据量。后续 Doris 重新进行负载均衡

**对于多租户部署环境下，BE 节点的扩容和缩容，请参阅 [多租户设计文档](https://github.com/apache/incubator-doris/wiki/Multi-Tenant)。**

### Broker 扩容缩容

Broker 实例的数量没有硬性要求。通常每台物理机部署一个即可。Broker 的添加和删除可以通过以下命令完成：

```ALTER SYSTEM ADD BROKER broker_name "broker_host:broker_ipc_port";```
```ALTER SYSTEM DROP BROKER broker_name "broker_host:broker_ipc_port";```
```ALTER SYSTEM DROP ALL BROKER broker_name;```

Broker 是无状态的进程，可以随意启停。当然，停止后，正在其上运行的作业会失败，重试即可。

## Doris 升级

Doris 可以通过滚动升级的方式，平滑进行升级。建议按照以下步骤进行安全升级。

> 注：  
> 1. 以下方式均建立在高可用部署的情况下。即数据 3 副本，FE 高可用情况下。  
> 2. 进程是否启动成功的判断，请参阅后面 **常见问题** 章节。

1. 测试 BE 升级正确性
	1. 任意选择一个 BE 节点，部署最新的 palo_be 二进制文件。
	2. 重启 BE 节点，通过 BE 日志 be.INFO，查看是否启动成功。
	3. 如果启动失败，可以先排查原因。如果错误不可恢复，可以直接通过 DROP BACKEND 删除该 BE、清理数据后，使用上一个版本的 palo_be 重新启动 BE。然后重新 ADD BACKEND。（**该方法会导致丢失一个数据副本，请务必确保3副本完整的情况下，执行这个操作！！！**）

2. 测试 FE 元数据兼容性（重要！!元数据兼容性异常很可能导致数据无法恢复！！）
	1. 单独使用新版本部署一个测试用的 FE 进程（比如自己本地的开发机）。
	2. 修改测试用的 FE 的配置文件 fe.conf，将所有端口设置为**与线上不同**。
	3. 在 fe.conf 添加配置：cluster_id=123456
	4. 在 fe.conf 添加配置：metadata\_failure_recovery=true
	5. 拷贝线上环境 Master FE 的元数据目录 palo-meta 到测试环境
	6. 将拷贝到测试环境中的 palo-meta/image/VERSION 文件中的 cluster_id 修改为 123456（即与第3步中相同）
	7. 在测试环境中，运行 sh bin/start_fe.sh 启动 FE
	8. 通过 FE 日志 fe.log 观察是否启动成功。
	9. 如果启动成功，运行 sh bin/stop_fe.sh 停止测试环境的 FE 进程。
	10. **以上 2-6 步的目的是防止测试环境的FE启动后，错误连接到线上环境中。**

3. 升级准备
	1. 在完成数据正确性验证后，将 BE 和 FE 新版本的二进制文件分发到各自目录下。
	2. 通常小版本升级，BE 只需升级 palo_be；而 FE 只需升级 palo-fe.jar。如果是大版本升级，则可能需要升级其他文件（包括但不限于 bin/ lib/ 等等）如果您不清楚是否需要替换其他文件，建议全部替换即可。

4. 升级
	1. 确认新版本的文件部署完成后。逐台重启 FE 和 BE 实例即可。
	2. 建议逐台重启 FE 后，再逐台重启 BE。因为如果升级导致新旧 FE、BE 不兼容，从新 FE 发出的命令可能会导致旧的 BE 挂掉。但是因为已经部署了新的 BE 文件，BE 通过守护进程自动重启后，即已经是新的 BE 了。
	3. 建议确认前一个实例启动成功后，在重启下一个实例。实例启动成功的标识，请参阅 **常见问题** 一节
	
## 前端页面

Doris 可以通过 FE 的 http_port，在浏览器显示系统展示页面。使用浏览器登陆：

```http://fe_host:http_port```

然后输入 root 用户名及密码即可登录页面（该页面仅支持 root 用户登录）。以下简要介绍导航栏的各个标签页

1. Doris（首页）

	![](https://github.com/apache/incubator-doris/blob/master/docs/resources/fe_page_index.png)
	
	首页主要展示一些版本信息和硬件信息（待完善）
	
2. System

	![](https://github.com/apache/incubator-doris/blob/master/docs/resources/fe_page_system.png)

	System 页面主要展示各类元数据信息。下面按字母序分别介绍各个子条目的内容。
	> 注：  
	> 1. 点击任意表格的标题列，可以按照该列排序。  
	> 2. 修改表格左上角 `records per page` 可以修改每页展示的条目。  
	> 3. 在表格右上角的 `Search:` 中可以按关键词过滤表格。  
	> 4. `Current path:` 后面的路径称为 *PROC*。如果无法通过浏览器显示页面，则可以在 mysql 客户端执行 `SHOW PROC '/current_path';` 来显示结果页的表格。
	
	1. access_resource
	
		![](https://github.com/apache/incubator-doris/blob/master/docs/resources/fe_page_system_access.png)
		
		该页面展示系统内所有用户及其权限信息、以及每个用户最大连接数：`MaxConn`。该连接数表示一个用户在一台 FE 上的最大连接数量。
		
	2. backends
	
		![](https://github.com/apache/incubator-doris/blob/master/docs/resources/fe_page_system_backends.png)
		
		* 该页面展示所有 BE 节点，包括 IP、host、各种端口、心跳信息、状态、数据分片数量、存储空间使用量等。
		* `LastStartTime` 表示 BE 最近一次启动时间。该时间可以用于判断 **BE 是否重启过。**
		* `LastHeartbeat` 表示 BE 最近一次收到心跳的时间。
		* `SystemDecommissioned` 为 `true`，表示该 BE 正在从整个 Doris 集群中下线。下线完成后，BE 将会被彻底删除。
		* `ClusterDecommissioned` 为 `true`，表示该 BE 正在从所属的 cluster 下线。下线完成后，BE 将会变成 free 状态，等待加入某一 cluster。
		* `TabletNum` 表示该 BE 上数据分片的总数。`FreeSpace` 表示该 BE 剩余磁盘空间占比。通过这两个数值，可以查看整个集群的存储负载均衡情况。
		* 点击第一列 `BanckendId` 可以在子页面中查看该 BE 的各个数据目录磁盘空间使用情况。
		* 当数据分片数量较多时，该页面打开的时间可能会较长（10几秒）。

	3. brokers

		![](https://github.com/apache/incubator-doris/blob/master/docs/resources/fe_page_system_brokers.png)
		
		* 该页面展示所有已添加的 brokers

	4. dbs

		![](https://github.com/apache/incubator-doris/blob/master/docs/resources/fe_page_system_dbs.png)

		* 该页面展示所有数据库的元信息。
		* 顶级页面包括数据库名称、table 数量、quota（数据库数据量上限）。这里，包括后续子页面的 `LastConsistencyCheckTime` 表示对应元素一致性检查的时间。
		* 点击第一列 `DbId` 可以进入子页面（后续所有子页面，都可以点击**第一列**进入下一级子页面）。
		* **第一级**子页面（点击 `DbId` 后进入的页面，后面子页面层级类推）展示 Table 信息。其中 `State` 列，以及后续子页面出现的 `State` 列。在正常情况下都应为 **NORMAL**，如果对应的元素在进行 SchemaChange 或者 Rollup，则对应的状态应为 **SCHEMA_CHANGE** 或者 **ROLLUP**。假设 Table 的 `STATE` 列为 **X**，则后续的该 Table 的 Partition、Index 以及 Tablet 的状态都应为 **X**。而 Tablet 下的 Replica 状态可能为 **NORMAL**、**X** 或 **CLONE**。
		* **第二级**子页面可以选择查看 **partitions** 或者 **index_schema**。后者展示 Base 表及其 Rollup 表的列信息。
		* **第三级**子页面（**partitions** 页面）显示该表的所有 partition 信息，包括导入版本、分区列、Range 范围、分桶列、桶数、副本数、存储介质等信息。如果是单分区表，则可能没有分区列和 Range 信息。
		* **第四级**子页面展示 partition 下所有 index（既 Base 表和 Rollup 表）。正常情况下，IndexId 最小的，既是 Base 表。
		* **第五级**子页面展示一个 Index 下所有 tablet 的信息。包括其副本状态、所在 BE，大小、行数等。`CheckVersion` 和 `CheckVersionHash` 是最后一次一致性检查所检查的版本。
		* **第六级**子页面即单个 tablet 的副本信息。

	5. frontends
	
		![](https://github.com/apache/incubator-doris/blob/master/docs/resources/fe_page_system_frontends.png)
		
		* 该页面展示所有 FE 节点。包括 Ip、edit\_log_port、角色、cluster id 等信息。
		* `Join` 列表示是否已经通过 bdbje 加入集群。当通过 `ADD FRONTEND` 添加 FE，并且 FE 未启动前，该列应为 false。FE 正常启动并加入集群后，该列会变为 true。
	
	6. jobs

		![](https://github.com/apache/incubator-doris/blob/master/docs/resources/fe_page_system_jobs.png)
		
		* 点击 `jobs` 首先会看到 database 列表。点击某一 database，可以出现如上页面，既该 database 相关的 job 信息汇总页面。
		* 汇总页面展示了所有类型的 job 的统计信息。**注意** Doris 通常会在一定时间后删除历史 job。所以 `Total` 展示的只是所有当前还未被删除的 job 的汇总数量。
		* 点击 `JobType` 列可以进入各类作业的详细信息页。除了 `Clone` 作业，其他作业都有相应的 `SHOW XXXX`(比如 `SHOW LOAD`) 语句对应。页面所展示的信息与语句结果基本一致。
		* `Clone` 作业是这些作业中唯一由系统、而不是用户触发的作业。如果 `Clone` 作业的状态为 `SUPPLEMENT`，表示该数据库有副本不齐的情况。如果为 `MIGRATION`，则表示正在进行负载均衡。

	7. load\_error\_hub_url
	
		![](https://github.com/apache/incubator-doris/blob/master/docs/resources/fe_page_system_error_hub.png)

		* 该页面展示 load_error_hub 的信息。Doris 支持将 load 作业产生的错误信息集中存储到一个 error hub 中（既 mysql）。然后直接通过 `SHOW LOAD WARNINGS;` 语句查看错误信息。这里展示的就是 error hub 的配置信息。
		
	8. statistic
	
		![](https://github.com/apache/incubator-doris/blob/master/docs/resources/fe_page_system_statistic.png)

		* 该页面展示所有数据分片及副本的统计信息。包括每个 database 的 table 数量、partition 数量、index 数据量、分片（tablet）数量、副本（replica）数据量，以及数量总和。  
		* IncompleteTabletNum 表示对应数据库中有多少 tablet 处于副本不齐的状态。  
		* InconsistentTabletNum 表示对应数据库中有多少 tablet 处于副本不一致的状态。  
		* 正常情况下，这两个数值应该为**零**。如果非零，可以点击对应的 **DbId**，子页面中会显示具体的副本不齐或者副本不一致的 tablet 的 id 列表。
		* 通过 mysql 使用 root 用户登录，使用 `SHOW TABLET tablet_id;` 命令可以查看一个 tablet 的具体信息。也可以通过 `SHOW TABLET FROM table_name;` 查看一个 table 所有的数据分片
	
	9. tasks

		![](https://github.com/apache/incubator-doris/blob/master/docs/resources/fe_page_system_tasks.png)

		* 该页面展示所有正在运行的 task 的信息。task 既对应 job 的子任务。该页面通常用于查看这些 task 是否**运行正常**，或者 task 是否有**堆积**。
		* `FailedNum` 表示失败的 task 的数量。`TotalNum` 表示总数。
		* 正常情况下，`FailedNum` 应该为**零**，表示没有失败的情况。`TotalNum` 应该维持在可接受的数量内，表示 task 没有堆积。
		* 点击第一列（`TaskType`）可以查看正在运行的 task 分布在哪些 BE 上。
		* 点击子页面的 `BackendId` 可以查看该类型的 task，在指定 BE 上，失败的 task 以及其重试次数。通常一个 task 对应的是一个 tablet，所以 `TaskSignature` 列即为 tablet id。可以前往该 BE，查看 BE 日志中有关该 tablet 的错误信息。

3. logs

	![](https://github.com/apache/incubator-doris/blob/master/docs/resources/fe_page_logs.png)
	
	* 该页面展示最新的 10MB 的 fe.warn.log 日志内容。
	* `ADD` 输入框可以动态的打开或关闭某个 java package 或文件的 DEBUG 日志。（仅限 FE）
	* 比如想打开 Clone.java 这个文件中的 DEBUG 日志，则在 ADD 输入框中输入 `org.apache.doris.clone.Clone`（既文件对应包的路径+文件名），然后点击 `ADD`，即可打开 Clone.java 这个文件中的 DEBUG 日志。DEBUG 日志在 fe.log 中，不在 fe.warn.log 中，所以该页面看不到 DEBUG 日志。
	* 比如想打开 load 这个 pakage 下所有文件的 DEBUG 日志，则添加 `org.apache.doris.load`（即仅包路径）即可。
	* 在 `Delete` 输入框可以输入对应 `ADD` 的内容，即可删除（关闭）相应 DEBUG 日志。

4. queries

	![](https://github.com/apache/incubator-doris/blob/master/docs/resources/fe_page_queries.png)

	* 该页面显示最新 100 个 query 的详细信息。**注意**，只有在 mysql 中设置 **session 变量：`set is_report_success=true;`** 之后，后续的查询才会显示在这里。
	* 对于 query 查询，`Query State` 为 EOF 即表示正常。
	* 点击最后一列 `Profile`，可以查看该 query 的详细执行情况。

5. sessions

	![](https://github.com/apache/incubator-doris/blob/master/docs/resources/fe_page_queries.png)
	
	* 该页面展示当前连接到这个 FE 的连接信息，包括 session id、用户、来自哪个主机等。
	* 该页面的信息，可以通过执行 mysql 命令 `SHOW PROCESSLIST;` 查看。
	* 如果需要杀死某个 session，可以通过 mysql 命令 `KILL session_id;` 执行。

6. variables

	* 该页面分为 `Configure Info` 和 `Variable Info` 两部分。
	* `Configure Info` 展示了当前 FE 的配置信息。
	* `Variable Info` 为当前 mysql 的(global)session variable 信息。
	* `Configure Info` 中的配置项可以通过以下 http 接口动态修改而不用重启 FE：
		`fe_host:http:port/api/_set_config?config_name=config_value`  
		但参数是否能生效，要看具体的实现。并且这种修改方式，只作用于该 FE，且在 FE 重启后失效。
		
7. ha

	* 该页面展示了 FE 高可用的一些信息。包括当前 FE 的角色、当前写入的 journal id、checkpoint、其他 FE 节点的信息。
	* `Electable nodes` 中是所有 Follower（包括 Master）FE。
	* `Observer nodes` 中是所有 Observer FE。
	* `Checkpoint Info` 的 `last checkpoint version` 是当前 image（checkpoint） 中包含的最大的 journal id。而 `last checkpoint time` 是最新的 image 生成时间。默认每 100000 条日志生成一次 image。如果发现很久都没有生成新的 image 了，有可能是 **checkpoint 遇到了问题**。需要到 fe 的 fe.log 日志中，搜索由 `Checkpoint.java` 文件中打印出来的日志信息，查看具体问题。
	* `Database names` 是 BDBJE 内部的 db 名称，和用户数据库名称无关。
	* `Allowed Frontends` 包含所有当前的 FE 节点。
	* `Removed Frontends` 包含所有曾经移除过的 FE 节点。这些节点对应的 ip:port 不能在重复使用。需要更换 ip 或者 port 后，才能重新加入。

8. help

	该页面用户搜索和展示各类语句帮助。
	
	* Help 是**唯一一个不需要 root 权限即可查看**的页面。 可以直接通过：
	`fe_host:http_port/help` 访问。
	* 在输入框中输入想要搜索的语句的关键词，比如 `load`，即可查找相关关键词对应的语法的帮助。
		
## 常见问题

### 进程相关

1. 如何确定 FE 进程启动成功
	
	FE 进程启动后，会首先加载元数据，根据 FE 角色的不同，在日志中会看到 ```transfer from UNKNOWN to MASTER/FOLLOWER/OBSERVER```。最终会看到 ```thrift server started``` 日志，并且可以通过 mysql 客户端连接到 FE，则表示 FE 启动成功。

	也可以通过如下连接查看是否启动成功：  
	`http://fe_host:fe_http_port/api/bootstrap`

	如果返回：  
	`{"status":"OK","msg":"Success"}`

	则表示启动成功，其余情况，则可能存在问题。
	
	> 注：如果在 fe.log 中查看不到启动失败的信息，也许在 fe.out 中可以看到。

2. 如何确定 BE 进程启动成功

	BE 进程启动后，如果之前有数据，则可能有数分钟不等的数据索引加载时间。  
	
	如果是 BE 的第一次启动，或者该 BE 尚未加入任何集群，则 BE 日志会定期滚动 ```waiting to receive first heartbeat from frontend``` 字样。表示 BE 还未通过 FE 的心跳收到 Master 的地址，正在被动等待。这种错误日志，在 FE 中 ADD BACKEND 并发送心跳后，就会消失。如果在接到心跳后，又重复出现 ``````master client, get client from cache failed.host: , port: 0, code: 7`````` 字样，说明 FE 成功连接了 BE，但 BE 无法主动连接 FE。可能需要检查 BE 到 FE 的 rpc_port 的连通性。  
	
	如果 BE 已经被加入集群，日志中应该每隔 5 秒滚动来自 FE 的心跳日志：```get heartbeat, host: xx.xx.xx.xx, port: 9020, cluster id: xxxxxx```，表示心跳正常。  
	
	其次，日志中应该每隔 10 秒滚动 ```finish report task success. return code: 0``` 的字样，表示 BE 向 FE 的通信正常。
	
	同时，如果有数据查询，应该能看到不停滚动的日志，并且有 ```execute time is xxx``` 日志，表示 BE 启动成功，并且查询正常。  

	也可以通过如下连接查看是否启动成功：  
	`http://be_host:be_http_port/api/health`

	如果返回：  
	`{"status": "OK","msg": "To Be Added"}`

	则表示启动成功，其余情况，则可能存在问题。
	
	> 注：如果在 be.INFO 中查看不到启动失败的信息，也许在 be.out 中可以看到。
	
3. 搭建系统后，如何确定 FE、BE 连通性正常
	
	首先确认 FE 和 BE 进程都已经单独正常启动，并确认已经通过 `ADD BACKEND` 或者 `ADD FOLLOWER/OBSERVER` 语句添加了所有节点。
	
	如果心跳正常，BE 的日志中会显示 ```get heartbeat, host: xx.xx.xx.xx, port: 9020, cluster id: xxxxxx```。如果心跳失败，在 FE 的日志中会出现 ```backend[10001] got Exception: org.apache.thrift.transport.TTransportException``` 类似的字样，或者其他 thrift 通信异常日志，表示 FE 向 10001 这个 BE 的心跳失败。这里需要检查 FE 向 BE host 的心跳端口的连通性。
	
	如果 BE 向 FE 的通信正常，则 BE 日志中会显示 ```finish report task success. return code: 0``` 的字样。否则会出现 ```master client, get client from cache failed``` 的字样。这种情况下，需要检查 BE 向 FE 的 rpc_port 的连通性。
	
4. Doris 各节点认证机制

	除了 Master FE 以外，其余角色节点（Follower FE，Observer FE，Backend），都需要通过 `ALTER SYSTEM ADD` 语句先注册到集群，然后才能加入集群。
	
	Master FE 在第一次启动时，会在 palo-meta/image/VERSION 文件中生成一个 cluster_id。
	
	FE 在第一次加入集群时，会首先从 Master FE 获取这个文件。之后每次 FE 之间的重新连接（FE 重启），都会校验自身 cluster id 是否与已存在的其它 FE 的 cluster id 相同。如果不同，则该 FE 会自动退出。
	
	BE 在第一次接收到 Master FE 的心跳时，会从心跳中获取到 cluster id，并记录到数据目录的 `cluster_id` 文件中。之后的每次心跳都会比对 FE 发来的 cluster id。如果 cluster id 不相等，则 BE 会拒绝响应 FE 的心跳。
	
	心跳中同时会包含 Master FE 的 ip。当 FE 切主时，新的 Master FE 会携带自身的 ip 发送心跳给 BE，BE 会更新自身保存的 Master FE 的 ip。

	> **priority\_network**  
	> priority\_network 是 FE 和 BE 都有一个配置，其主要目的是在多网卡的情况下，协助 FE 或 BE 识别自身 ip 地址。priority\_network 采用 CIDR 表示法：[RFC 4632](https://tools.ietf.org/html/rfc4632)  
	> 当确认 FE 和 BE 连通性正常后，如果仍然出现建表 Timeout 的情况，并且 FE 的日志中有 `backend does not found. host: xxx.xxx.xxx.xxx` 字样的错误信息。则表示 Doris 自动识别的 IP 地址有问题，需要手动设置 priority\_network 参数。  
	> 出现这个问题的主要原因是：当用户通过 `ADD BACKEND` 语句添加 BE 后，FE 会识别该语句中指定的是 hostname 还是 IP。如果是 hostname，则 FE 会自动将其转换为 IP 地址并存储到元数据中。当 BE 在汇报任务完成信息时，会携带自己的 IP 地址。而如果 FE 发现 BE 汇报的 IP 地址和元数据中不一致时，就会出现如上错误。  
	> 这个错误的解决方法：1）分别在 FE 和 BE 设置 **priority\_network** 参数。通常 FE 和 BE 都处于一个网段，所以该参数设置为相同即可。2）在 `ADD BACKEND` 语句中直接填写 BE 正确的 IP 地址而不是 hostname，以避免 FE 获取到错误的 IP 地址。
