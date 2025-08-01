---
title: 使用 TiDB Lightning 恢复 GCS 上的备份数据 (Helm)
summary: 介绍如何使用 TiDB Lightning 将存储在 GCS 上的备份数据恢复到 TiDB 集群。
---

# 使用 TiDB Lightning 恢复 GCS 上的备份数据 (Helm)

> **警告：**
>
> 本文介绍的 Helm 部署方式已弃用，建议使用 [Job 方式](restore-from-gcs-using-job.md)进行数据恢复操作。

本文档介绍如何将 Kubernetes 上通过 TiDB Operator 备份的数据恢复到 TiDB 集群。

本文使用的恢复方式基于 TiDB Operator v1.1 及以上的 CustomResourceDefinition (CRD) 实现，底层通过使用 [TiDB Lightning TiDB-backend](https://docs.pingcap.com/zh/tidb/stable/tidb-lightning-backends#tidb-lightning-tidb-backend) 来恢复数据。

TiDB Lightning 是一款将全量数据高速导入到 TiDB 集群的工具，可用于从本地盘、Google Cloud Storage (GCS) 或 Amazon S3 云盘读取数据。目前，TiDB Lightning 支持三种后端：`Importer-backend`、`Local-backend`、`TiDB-backend`。本文介绍的方法使用 `TiDB-backend`。关于这三种后端的区别和选择，请参阅 [TiDB Lightning 文档](https://docs.pingcap.com/zh/tidb/stable/tidb-lightning-backends)。如果要使用 `Importer-backend` 或者 `Local-backend` 导入数据，请参阅[使用 TiDB Lightning 导入集群数据](restore-data-using-tidb-lightning.md)。

以下示例将存储在 [GCS](https://cloud.google.com/storage/docs/) 上指定路径上的集群备份数据恢复到 TiDB 集群。

## 使用场景

如果你需要从 GCS 导出备份数据到 TiDB 集群，并对数据恢复有以下要求，可使用本文介绍的恢复方案：

- 希望以较低资源占用率和较低网络带宽占用进行恢复，并能接受 50 GB/小时的恢复速度
- 要求导入集群时满足 ACID
- 要求备份期间 TiDB 集群仍可对外提供服务

## 恢复前的准备

在进行数据恢复前，你需要准备恢复环境，并拥有数据库的相关权限。

### 环境准备

1. 下载文件 [`backup-rbac.yaml`](<https://github.com/pingcap/tidb-operator/blob/v1.6.2/manifests/backup/backup-rbac.yaml>)，并执行以下命令在 `test2` 这个 namespace 中创建恢复所需的 RBAC 相关资源：

    
    ```shell
    kubectl apply -f backup-rbac.yaml -n test2
    ```

2. 远程存储访问授权。

    参考 [GCS 账号授权](grant-permissions-to-remote-storage.md#gcs-账号授权)授权访问 GCS 远程存储。

3. 创建 `restore-demo2-tidb-secret` secret，该 secret 存放用来访问 TiDB 集群的 root 账号和密钥：

    
    ```shell
    kubectl create secret generic restore-demo2-tidb-secret --from-literal=user=root --from-literal=password=${password} --namespace=test2
    ```

### 所需的数据库权限

使用 TiDB Lightning 将 GCS 上的备份数据恢复至 TiDB 集群前，确保你拥有备份数据库的以下权限：

| 权限 | 作用域 |
|:----|:------|
| SELECT | Tables |
| INSERT | Tables |
| UPDATE | Tables |
| DELETE | Tables |
| CREATE | Databases, tables |
| DROP | Databases, tables |
| ALTER | Tables |

## 将指定备份数据恢复到 TiDB 集群

1. 创建 restore custom resource (CR)，将指定的备份数据恢复至 TiDB 集群：

    
    ```shell
    kubectl apply -f restore.yaml
    ```

    `restore.yaml` 文件内容如下：

    ```yaml
    ---
    apiVersion: pingcap.com/v1alpha1
    kind: Restore
    metadata:
      name: demo2-restore
      namespace: test2
    spec:
      to:
        host: ${tidb_host}
        port: ${tidb_port}
        user: ${tidb_user}
        secretName: restore-demo2-tidb-secret
      gcs:
        projectId: ${project_id}
        secretName: gcs-secret
        path: gcs://${backup_path}
      # storageClassName: local-storage
      storageSize: 1Gi
    ```

    以上示例将存储在 GCS 上指定路径 `spec.gcs.path` 的备份数据恢复到 TiDB 集群 `spec.to.host`。关于 GCS 的配置项可以参考 [GCS 字段介绍](backup-restore-cr.md#gcs-存储字段介绍)。

    更多 `Restore` CR 字段的详细解释参考 [Restore CR 字段介绍](backup-restore-cr.md#restore-cr-字段介绍)。

2. 创建好 `Restore` CR 后可通过以下命令查看恢复的状态：

    
     ```shell
     kubectl get rt -n test2 -owide
     ```

> **注意：**
>
> TiDB Operator 会创建一个 PVC，用于数据恢复，备份数据会先从远端存储下载到 PV，然后再进行恢复。如果恢复完成后想要删掉这个 PVC，可以参考[删除资源](cheat-sheet.md#删除资源)先把恢复 Pod 删掉，然后再把 PVC 删掉。

## 故障诊断

在使用过程中如果遇到问题，可以参考[故障诊断](deploy-failures.md)。
