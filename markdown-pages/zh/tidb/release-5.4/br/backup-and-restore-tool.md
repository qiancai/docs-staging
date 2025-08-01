---
title: 备份与恢复工具 BR 简介
summary: 了解 BR 工具是什么、有什么用。
---

# 备份与恢复工具 BR 简介

[BR](https://github.com/pingcap/br) 全称为 Backup & Restore，是 TiDB **分布式备份恢复**的命令行工具，用于对 TiDB 集群进行数据备份和恢复。

相比 [Dumpling](/dumpling-overview.md)，BR 更适合**大数据量**的场景。

BR 除了可以用来进行常规备份恢复外，也可以在保证兼容性前提下用来做大规模的数据迁移。

本文介绍了 BR 的工作原理、推荐部署配置、使用限制以及几种使用方式。

## 工作原理

BR 将备份或恢复操作命令下发到各个 TiKV 节点。TiKV 收到命令后执行相应的备份或恢复操作。

在一次备份或恢复中，各个 TiKV 节点都会有一个对应的备份路径，TiKV 备份时产生的备份文件将会保存在该路径下，恢复时也会从该路径读取相应的备份文件。

![br-arch](https://docs-download.pingcap.com/media/images/docs-cn/br-arch.png)

更多信息请参阅[备份恢复设计方案](https://github.com/pingcap/br/blob/980627aa90e5d6f0349b423127e0221b4fa09ba0/docs/cn/2019-08-05-new-design-of-backup-restore.md)。

### 备份文件类型

备份路径下会生成以下两种类型文件：

- SST 文件：存储 TiKV 备份下来的数据信息
- `backupmeta` 文件：存储本次备份的元信息，包括备份文件数、备份文件的 Key 区间、备份文件大小和备份文件 Hash (sha256) 值
- `backup.lock` 文件：用于防止多次备份到同一目录

### SST 文件命名格式

SST 文件以 `storeID_regionID_regionEpoch_keyHash_cf` 的格式命名。格式名的解释如下：

- storeID：TiKV 节点编号
- regionID：Region 编号
- regionEpoch：Region 版本号
- keyHash：Range startKey 的 Hash (sha256) 值，确保唯一性
- cf：RocksDB 的 ColumnFamily（默认为 `default` 或 `write`）

## 部署使用 BR 工具

### 推荐部署配置

- 推荐 BR 部署在 PD 节点上。
- 推荐使用一块高性能 SSD 网盘，挂载到 BR 节点和所有 TiKV 节点上，网盘推荐万兆网卡，否则带宽有可能成为备份恢复时的性能瓶颈。

> **注意：**
>
> - 如果没有挂载网盘或者使用其他共享存储，那么 BR 备份的数据会生成在各个 TiKV 节点上。由于 BR 只备份 leader 副本，所以各个节点预留的空间需要根据 leader size 来预估。
> - 同时由于 TiDB 默认使用 leader count 进行平衡，所以会出现 leader size 差别大的问题，导致各个节点备份数据不均衡。

### 使用限制

下面是使用 BR 进行备份恢复的几条限制：

- BR 恢复到 TiCDC / Drainer 的上游集群时，恢复数据无法由 TiCDC / Drainer 同步到下游。
- BR 只支持在 `new_collations_enabled_on_first_bootstrap` [开关值](/character-set-and-collation.md#排序规则支持)相同的集群之间进行操作。这是因为 BR 仅备份 KV 数据。如果备份集群和恢复集群采用不同的排序规则，数据校验会不通过。所以恢复集群时，你需要确保 `select VARIABLE_VALUE from mysql.tidb where VARIABLE_NAME='new_collation_enabled';` 语句的开关值查询结果与备份时的查询结果相一致，才可以进行恢复。

### 兼容性

BR 和 TiDB 集群的兼容性问题分为以下两方面：

+ BR 部分版本和 TiDB 集群的接口不兼容

  BR 在 v5.4.0 之前不支持恢复 `charset=GBK` 的表。并且，任何版本的 BR 都不支持恢复 `charset=GBK` 的表到 5.4.0 之前的 TiDB 集群。
  
+ 某些功能在开启或关闭状态下，会导致 KV 格式发生变化，因此备份和恢复期间如果没有统一开启或关闭，就会带来不兼容的问题

下表整理了会导致 KV 格式发生变化的功能。

| 功能 | 相关 issue | 解决方式 |
|  ----  | ----  | ----- |
| 聚簇索引 | [#565](https://github.com/pingcap/br/issues/565)       | 确保备份时 `tidb_enable_clustered_index` 全局变量和恢复时一致，否则会导致数据不一致的问题，例如 `default not found` 和数据索引不一致。 |
| New collation  | [#352](https://github.com/pingcap/br/issues/352)       | 确保恢复时集群的 `new_collations_enabled_on_first_bootstrap` 变量值和备份时的一致，否则会导致数据索引不一致和 checksum 通不过。 |
| 恢复集群开启 TiCDC 同步 | [#364](https://github.com/pingcap/br/issues/364#issuecomment-646813965) |  TiKV 暂不能将 BR ingest 的 SST 文件下推到 TiCDC，因此使用 BR 恢复时候需要关闭 TiCDC。 |
| 全局临时表 | | 确保使用 BR v5.3.0 及以上版本进行备份和恢复，否则会导致全局临时表的表定义错误。 |

在上述功能确保备份恢复一致的**前提**下，BR 和 TiKV/TiDB/PD 还可能因为版本内部协议不一致/接口不一致出现不兼容的问题，因此 BR 内置了版本检查。

#### 版本检查

BR 内置版本会在执行备份和恢复操作前，对 TiDB 集群版本和自身版本进行对比检查。如果大版本不匹配（比如 BR v4.x 和 TiDB v5.x 上），BR 会提示退出。如要跳过版本检查，可以通过设置 `--check-requirements=false` 强行跳过版本检查。

需要注意的是，跳过检查可能会遇到版本不兼容的问题。BR 和 TiDB 各版本兼容情况如下表所示：

| 恢复版本（横向）\ 备份版本（纵向）   | 用 BR v5.4 恢复 TiDB v5.4 | 用 BR v5.0 恢复 TiDB v5.0| 用 BR v4.0 恢复 TiDB v4.0 |
|  ----  |  ----  | ---- | ---- |
| 用 BR v5.4 备份 TiDB v5.4 | ✅ | ✅ | ❌（如果恢复了使用非整数类型聚簇主键的表到 v4.0 的 TiDB 集群，BR 会无任何警告地导致数据错误） |
| 用 BR v5.0 备份 TiDB v5.0 | ✅ | ✅ | ❌（如果恢复了使用非整数类型聚簇主键的表到 v4.0 的 TiDB 集群，BR 会无任何警告地导致数据错误）
| 用 BR v4.0 备份 TiDB v4.0 | ✅ | ✅  | ✅（如果 TiKV >= v4.0.0-rc.1，BR 包含 [#233](https://github.com/pingcap/br/pull/233) Bug 修复，且 TiKV 不包含 [#7241](https://github.com/tikv/tikv/pull/7241) Bug 修复，那么 BR 会导致 TiKV 节点重启) |
| 用 BR v5.4 或 v5.0 备份 TiDB v4.0 | ❌（当 TiDB 版本小于 v4.0.9 时会出现 [#609](https://github.com/pingcap/br/issues/609) 问题) | ❌（当 TiDB 版本小于 v4.0.9 会出现 [#609](https://github.com/pingcap/br/issues/609) 问题) | ❌（当 TiDB 版本小于 v4.0.9 会出现 [#609](https://github.com/pingcap/br/issues/609) 问题) |

### 备份和恢复 `mysql` 系统库下的表数据（实验特性）

> **警告：**
>
> 该功能目前为实验性功能，尚未经过充分测试。**不建议**在生产环境中使用该功能。

在 v5.1.0 之前，BR 备份时会过滤掉系统库 `mysql.*` 的表数据。自 v5.1.0 起，BR 默认**备份**集群内的全部数据，包括系统库 `mysql.*` 中的数据。但由于恢复 `mysql.*` 中系统表数据的技术实现尚不完善，因此 BR 默认**不恢复**系统库 `mysql` 中的表数据。

如果你希望将系统表（如 `mysql.usertable1`）的数据恢复到系统库 `mysql` 中，可以设置 [`filter` 参数](/br/use-br-command-line-tool.md#使用表库过滤功能备份多张表的数据) 来过滤表名 (`-f "mysql.usertable1"`)。完成设置后，系统库下的表数据会恢复到临时库中，然后通过对临时库表进行重命名的方式恢复到系统库。

由于技术原因，以下系统表不能被正确恢复。即使你指定了 `-f "mysql.*"`，以下表也不会被恢复：

- 统计信息相关的表："stats_buckets"，"stats_extended"，"stats_feedback"，"stats_fm_sketch"，"stats_histograms"，"stats_meta"，"stats_top_n"
- 权限或系统相关的表："tidb"，"global_variables"，"columns_priv"，"db"，"default_roles"，"global_grants"，"global_priv"，"role_edges"，"tables_priv"，"user"，"gc_delete_range"，"gc_delete_range_done"，"schema_index_usage"

### 运行 BR 的最低机型配置要求

运行 BR 的最低机型配置要求如下：

| CPU | 内存 | 硬盘类型 | 网络 |
| --- | --- | --- | --- |
| 1 核 | 4 GB | HDD | 千兆网卡 |

一般场景下（备份恢复的表少于 1000 张），BR 在运行期间的 CPU 消耗不会超过 200%，内存消耗不会超过 4 GB。但在备份和恢复大量数据表时，BR 的内存消耗可能会上升到 4 GB 以上。在实际测试中，备份 24000 张表大概需要消耗 2.7 GB 内存，CPU 消耗维持在 100% 以下。

### 最佳实践

下面是使用 BR 进行备份恢复的一些建议：

- 推荐在业务低峰时执行备份操作，这样能最大程度地减少对业务的影响；
- BR 恢复数据时会尽可能多地占用恢复集群的资源，因此推荐恢复数据到新集群或离线集群。应避免向正在提供服务的生产集群执行恢复，否则，恢复期间会对业务产生不可避免的影响；
- 不推荐多个 BR 备份和恢复任务并行运行。不同的任务并行，不仅会导致备份或恢复的性能降低，影响在线业务，还会因为任务之间缺少协调机制造成任务失败，甚至对集群的状态产生影响；
- 推荐使用支持 S3/GCS/Azure Blob Storage 协议的存储系统保存备份数据；
- 应确保 BR、TiKV 节点和备份存储系统有足够的网络带宽，备份存储系统能提供足够的写入和读取性能，否则，它们有可能成为备份恢复时的性能瓶颈。

### 使用方式

目前支持以下几种方式来运行 BR 工具，分别是通过 SQL 语句、命令行工具或在 Kubernetes 环境下进行备份恢复。

#### 通过 SQL 语句

TiDB 支持使用 SQL 语句 [`BACKUP`](/sql-statements/sql-statement-backup.md#backup) 和 [`RESTORE`](/sql-statements/sql-statement-restore.md#restore) 进行备份恢复。如果要查看备份恢复的进度，你可以使用 [`SHOW BACKUPS|RESTORES`](/sql-statements/sql-statement-show-backups.md) 语句。

#### 通过命令行工具

TiDB 支持使用 BR 命令行工具进行备份恢复（需[手动下载](/download-ecosystem-tools.md#备份和恢复-br-工具)）。关于 BR 命令行工具的具体使用方法，请参阅[使用备份与恢复工具 BR](/br/use-br-command-line-tool.md)。

#### 在 Kubernetes 环境下

目前支持使用 BR 工具备份 TiDB 集群数据到兼容 S3 的存储、Google Cloud Storage 以及持久卷，并作恢复：

> **注意：**
>
> Amazon S3 和 Google Cloud Storage (GCS) 参数描述见[外部存储](/br/backup-and-restore-storages.md#url-参数)文档。

- [备份 TiDB 集群数据到兼容 S3 的存储](https://docs.pingcap.com/zh/tidb-in-kubernetes/stable/backup-to-aws-s3-using-br)
- [恢复 S3 兼容存储上的备份数据](https://docs.pingcap.com/zh/tidb-in-kubernetes/stable/restore-from-aws-s3-using-br)
- [备份 TiDB 集群到 Google Cloud Storage](https://docs.pingcap.com/zh/tidb-in-kubernetes/stable/backup-to-gcs-using-br)
- [恢复 Google Cloud Storage 上的备份数据](https://docs.pingcap.com/zh/tidb-in-kubernetes/stable/restore-from-gcs-using-br)
- [备份 TiDB 集群到持久卷](https://docs.pingcap.com/zh/tidb-in-kubernetes/stable/backup-to-pv-using-br)
- [恢复持久卷上的备份数据](https://docs.pingcap.com/zh/tidb-in-kubernetes/stable/restore-from-pv-using-br)

## BR 相关文档

+ [使用 BR 命令行备份恢复](/br/use-br-command-line-tool.md)
+ [BR 备份与恢复场景示例](/br/backup-and-restore-use-cases.md)
+ [BR 常见问题](/br/backup-and-restore-faq.md)
+ [外部存储](/br/backup-and-restore-storages.md)
