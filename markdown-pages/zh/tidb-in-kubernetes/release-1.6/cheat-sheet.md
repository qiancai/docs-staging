---
title: 管理 TiDB 集群的 Command Cheat Sheet
summary: 介绍管理 TiDB 集群的 Command Cheat Sheet。
---

# 管理 TiDB 集群的 Command Cheat Sheet

本文提供管理 TiDB 集群的 Command Cheat Sheet。

## kubectl

### 查看资源

* 查看 CRD：

    
    ```shell
    kubectl get crd
    ```

* 查看 TidbCluster：

    
    ```shell
    kubectl -n ${namespace} get tc ${name}
    ```

* 查看 TidbMonitor：

    
    ```shell
    kubectl -n ${namespace} get tidbmonitor ${name}
    ```

* 查看 Backup：

    
    ```shell
    kubectl -n ${namespace} get bk ${name}
    ```

* 查看 BackupSchedule：

    
    ```shell
    kubectl -n ${namespace} get bks ${name}
    ```

* 查看 Restore：

    
    ```shell
    kubectl -n ${namespace} get restore ${name}
    ```

* 查看 TidbClusterAutoScaler：

    
    ```shell
    kubectl -n ${namespace} get tidbclusterautoscaler ${name}
    ```

* 查看 TidbInitializer：

    
    ```shell
    kubectl -n ${namespace} get tidbinitializer ${name}
    ```

* 查看 Advanced StatefulSet：

    
    ```shell
    kubectl -n ${namespace} get asts ${name}
    ```

* 查看 Pod：

    
    ```shell
    kubectl -n ${namespace} get pod ${name}
    ```

    查看 TiKV Pod：

    
    ```shell
    kubectl -n ${namespace} get pod -l app.kubernetes.io/component=tikv
    ```

    持续观察 Pod 状态变化：

    ```shell
    watch kubectl -n ${namespace} get pod
    ```

    查看 Pod 详细信息：

    ```shell
    kubectl -n ${namespace} describe pod ${name}
    ```

* 查看 Pod 所在 Node：

    
    ```shell
    kubectl -n ${namespace} get pods -l "app.kubernetes.io/component=tidb,app.kubernetes.io/instance=${cluster_name}" -ojsonpath="{range .items[*]}{.spec.nodeName}{'\n'}{end}"
    ```

* 查看 Service：

    
    ```shell
    kubectl -n ${namespace} get service ${name}
    ```

* 查看 ConfigMap：

    
    ```shell
    kubectl -n ${namespace} get cm ${name}
    ```

* 查看 PV：

    
    ```shell
    kubectl -n ${namespace} get pv ${name}
    ```

    查看集群使用的 PV:

    
    ```shell
    kubectl get pv -l app.kubernetes.io/namespace=${namespace},app.kubernetes.io/managed-by=tidb-operator,app.kubernetes.io/instance=${cluster_name}
    ```

* 查看 PVC：

    
    ```shell
    kubectl -n ${namespace} get pvc ${name}
    ```

* 查看 StorageClass：

    
    ```shell
    kubectl -n ${namespace} get sc
    ```

* 查看 StatefulSet：

    
    ```shell
    kubectl -n ${namespace} get sts ${name}
    ```

    查看 StatefulSet 详细信息：

    
    ```shell
    kubectl -n ${namespace} describe sts ${name}
    ```

### 更新资源

* 为 TiDBCluster 增加 Annotation：

    
    ```shell
    kubectl -n ${namespace} annotate tc ${cluster_name} ${key}=${value}
    ```

    为 TiDBCluster 增加强制升级 Annotation：

    
    ```shell
    kubectl -n ${namespace} annotate --overwrite tc ${cluster_name} tidb.pingcap.com/force-upgrade=true
    ```

    为 TiDBCluster 删除强制升级 Annotation：

    
    ```shell
    kubectl -n ${namespace} annotate tc ${cluster_name} tidb.pingcap.com/force-upgrade-
    ```

    为 Pod 开启 Debug 模式：

    
    ```shell
    kubectl -n ${namespace} annotate pod ${pod_name} runmode=debug
    ```

### 编辑资源

* 编辑 TidbCluster：

    
    ```shell
    kubectl -n ${namespace} edit tc ${name}
    ```

### Patch 资源

* Patch TidbCluster:

  
    ```shell
    kubectl -n ${namespace} patch tc ${name} --type merge -p '${json_path}'
    ```

* Patch PV ReclaimPolicy：

    
    ```shell
    kubectl patch pv ${name} -p '{"spec":{"persistentVolumeReclaimPolicy":"Delete"}}'
    ```

* Patch PVC：

    
    ```shell
    kubectl -n ${namespace} patch pvc ${name} -p '{"spec": {"resources": {"requests": {"storage": "100Gi"}}}'
    ```

* Patch StorageClass：

    
    ```shell
    kubectl patch storageclass ${name} -p '{"allowVolumeExpansion": true}'
    ```

### 创建资源

* 通过 Yaml 文件创建集群：

    
    ```shell
    kubectl -n ${namespace} apply -f ${file}
    ```

* 创建 Namespace：

    
    ```shell
    kubectl create ns ${namespace}
    ```

* 创建 Secret：

    创建证书的 Secret：

    
    ```shell
    kubectl -n ${namespace} create secret generic ${secret_name} --from-file=tls.crt=${cert_path} --from-file=tls.key=${key_path} --from-file=ca.crt=${ca_path}
    ```

    创建用户名、密码的 Secret：

    
    ```shell
    kubectl -n ${namespace} create secret generic ${secret_name} --from-literal=user=${user} --from-literal=password=${password}
    ```

### 与 Running Pod 交互

* 查看 PD 配置文件：

    
    ```shell
    kubectl -n ${namespace} -it exec ${pod_name} -- cat /etc/pd/pd.toml
    ```

* 查看 TiDB 配置文件：

    
    ```shell
    kubectl -n ${namespace} -it exec ${pod_name} -- cat /etc/tidb/tidb.toml
    ```

* 查看 TiKV 配置文件：

    
    ```shell
    kubectl -n ${namespace} -it exec ${pod_name} -- cat /etc/tikv/tikv.toml
    ```

* 查看 Pod Log：

    
    ```shell
    kubectl -n ${namespace} logs ${pod_name} -f
    ```

    查看上一次容器的 Log：

    
    ```shell
    kubectl -n ${namespace} logs ${pod_name} -p
    ```

    如果 Pod 内有多个容器，查看某一个容器的 Log：

    
    ```shell
    kubectl -n ${namespace} logs ${pod_name} -c ${container_name}
    ```

* 暴露服务：

    
    ```shell
    kubectl -n ${namespace} port-forward svc/${service_name} ${local_port}:${port_in_pod}
    ```

    暴露 PD 服务：

    
    ```shell
    kubectl -n ${namespace} port-forward svc/${cluster_name}-pd 2379:2379
    ```

### 与 Node 交互

* 把 Node 设置为不可调度：

    
    ```shell
    kubectl cordon ${node_name}
    ```

* 取消 Node 不可调度：

    
    ```shell
    kubectl uncordon ${node_name}
    ```

### 删除资源

* 删除 Pod：

    
    ```shell
    kubectl delete -n ${namespace} pod ${pod_name}
    ```

* 删除 PVC：

    
    ```shell
    kubectl delete -n ${namespace} pvc ${pvc_name}
    ```

* 删除 TidbCluster：

    
    ```shell
    kubectl delete -n ${namespace} tc ${tc_name}
    ```

* 删除 TidbMonitor：

    
    ```shell
    kubectl delete -n ${namespace} tidbmonitor ${tidb_monitor_name}
    ```

* 删除 TidbClusterAutoScaler：

    
    ```shell
    kubectl -n ${namespace} delete tidbclusterautoscaler ${name}
    ```

### 更多

其他更多 kubectl 的使用，请参考 [Kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)。

## Helm

### 添加 Helm repository


```shell
helm repo add pingcap https://charts.pingcap.org/
```

### 更新 Helm repository


```shell
helm repo update
```

### 查看可用的 Helm Chart

- 查看 Helm Hub 中的 Chart

    
    ```shell
    helm search hub ${chart_name}
    ```

    示例：

    
    ```shell
    helm search hub mysql
    ```

- 查看其他 repository 中的 Chart

    
    ```shell
    helm search repo ${chart_name} -l --devel
    ```

    示例：

    
    ```shell
    helm search repo tidb-operator -l --devel
    ```

### 获取 Helm Chart 默认 values.yaml


```shell
helm inspect values ${chart_name} --version=${chart_version} > values.yaml
```

示例：


```shell
helm inspect values pingcap/tidb-operator --version=v1.6.2 > values-tidb-operator.yaml
```

### 使用 Helm Chart 部署


```shell
helm install ${name} ${chart_name} --namespace=${namespace} --version=${chart_version} -f ${values_file}
```

示例：


```shell
helm install tidb-operator pingcap/tidb-operator --namespace=tidb-admin --version=v1.6.2 -f values-tidb-operator.yaml
```

### 查看已经部署的 Helm Release


```shell
helm ls
```

### 升级 Helm Release


```shell
helm upgrade ${name} ${chart_name} --version=${chart_version} -f ${values_file}
```

示例：


```shell
helm upgrade tidb-operator pingcap/tidb-operator --version=v1.6.2 -f values-tidb-operator.yaml
```

### 删除 Helm Release


```shell
helm uninstall ${name} -n ${namespace}
```

示例：


```shell
helm uninstall tidb-operator -n tidb-admin
```

### 更多

其他更多 Helm 的使用，请参考 [Helm Commands](https://helm.sh/docs/helm/)。
