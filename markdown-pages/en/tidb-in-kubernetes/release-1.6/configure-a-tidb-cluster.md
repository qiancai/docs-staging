---
title: Configure a TiDB Cluster on Kubernetes
summary: Learn how to configure a TiDB cluster on Kubernetes.
---

# Configure a TiDB Cluster on Kubernetes

This document introduces how to configure a TiDB cluster for production deployment. It covers the following content:

- [Configure resources](#configure-resources)

- [Configure TiDB deployment](#configure-tidb-deployment)

- [Configure high availability](#configure-high-availability)

## Configure resources

Before deploying a TiDB cluster, it is necessary to configure the resources for each component of the cluster depending on your needs. PD, TiKV, and TiDB are the core service components of a TiDB cluster. In a production environment, you need to configure resources of these components according to their needs. For details, refer to [Hardware Recommendations](https://docs.pingcap.com/tidb/stable/hardware-and-software-requirements).

To ensure the proper scheduling and stable operation of the components of the TiDB cluster on Kubernetes, it is recommended to set Guaranteed-level quality of service (QoS) by making `limits` equal to `requests` when configuring resources. For details, refer to [Configure Quality of Service for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/).

If you are using a NUMA-based CPU, you need to enable `Static`'s CPU management policy on the node for better performance. In order to allow the TiDB cluster component to monopolize the corresponding CPU resources, the CPU quota must be an integer greater than or equal to `1`, apart from setting Guaranteed-level QoS as mentioned above. For details, refer to [Control CPU Management Policies on the Node](https://kubernetes.io/docs/tasks/administer-cluster/cpu-management-policies).

## Configure TiDB deployment

To configure a TiDB deployment, you need to configure the `TiDBCluster` CR. Refer to the [TidbCluster example](<https://github.com/pingcap/tidb-operator/blob/v1.6.2/examples/advanced/tidb-cluster.yaml>) for an example. For the complete configurations of `TiDBCluster` CR, refer to [API documentation](<https://github.com/pingcap/tidb-operator/blob/v1.6.2/docs/api-references/docs.md>).

> **Note:**
>
> It is recommended to organize configurations for a TiDB cluster under a directory of `cluster_name` and save it as `${cluster_name}/tidb-cluster.yaml`.
The modified configuration is not automatically applied to the TiDB cluster by default. The new configuration file is loaded only when the Pod restarts.

### Cluster name

The cluster name can be configured by changing `metadata.name` in the `TiDBCuster` CR.

### Version

Usually, components in a cluster are in the same version. It is recommended to configure `spec.<pd/tidb/tikv/pump/tiflash/ticdc>.baseImage` and `spec.version`, if you need to configure different versions for different components, you can configure `spec.<pd/tidb/tikv/pump/tiflash/ticdc>.version`.

Here are the formats of the parameters:

- `spec.version`: the format is `imageTag`, such as `v8.5.2`

- `spec.<pd/tidb/tikv/pump/tiflash/ticdc>.baseImage`: the format is `imageName`, such as `pingcap/tidb`

- `spec.<pd/tidb/tikv/pump/tiflash/ticdc>.version`: the format is `imageTag`, such as `v8.5.2`

### Recommended configuration

#### configUpdateStrategy

The default value of the `spec.configUpdateStrategy` field is `InPlace`, which means that when you modify `config` of a component, you need to manually trigger a rolling update to apply the new configurations to the cluster.

It is recommended that you configure `spec.configUpdateStrategy: RollingUpdate` to enable automatic update of configurations. In this way, every time the `config` of a component is updated, TiDB Operator automatically triggers a rolling update for the component and applies the modified configuration to the cluster.

#### enableDynamicConfiguration

It is recommended that you configure `spec.enableDynamicConfiguration: true` to enable the `--advertise-status-addr` startup parameter for TiKV.

Versions required:

- TiDB 4.0.1 or later versions

#### pvReclaimPolicy

It is recommended that you configure `spec.pvReclaimPolicy: Retain` to ensure that the PV is retained even if the PVC is deleted. This is to ensure your data safety.

#### mountClusterClientSecret

PD and TiKV supports configuring `mountClusterClientSecret`. If [TLS is enabled between cluster components](enable-tls-between-components.md), it is recommended to configure `spec.pd.mountClusterClientSecret: true` and `spec.tikv.mountClusterClientSecret: true`. Under such configuration, TiDB Operator automatically mounts the `${cluster_name}-cluster-client-secret` certificate to the PD and TiKV container, so you can conveniently [use `pd-ctl` and `tikv-ctl`](enable-tls-between-components.md#step-3-configure-pd-ctl-tikv-ctl-and-connect-to-the-cluster).

#### startScriptVersion

To choose the different versions of the startup scripts for each component, you can configure the `spec.startScriptVersion` field in the cluster spec.

The supported versions of the start script are as follows:

* `v1` (default): the original version of the startup script.

* `v2`: to optimize the start script for each component and make sure that upgrading TiDB Operator does not result in cluster rolling restart, TiDB Operator v1.4.0 introduces `v2`. Compared to `v1`, `v2` has the following optimizations:

    * Use `dig` instead of `nslookup` to resolve DNS.
    * All components support [debug mode](tips.md#use-the-debug-mode).

It is recommended that you configure `spec.startScriptVersion` as the latest version (`v2`) for the new cluster.

> **Warning:**
>
> Modify the `startScriptVersion` field of the deployed cluster will cause the rolling restart.

### Storage

#### Storage Class

You can set the storage class by modifying `storageClassName` of each component in `${cluster_name}/tidb-cluster.yaml` and `${cluster_name}/tidb-monitor.yaml`. For the [storage classes](https://kubernetes.io/docs/concepts/storage/storage-classes/) supported by the Kubernetes cluster, check with your system administrator.

Different components of a TiDB cluster have different disk requirements. Before deploying a TiDB cluster, refer to the [Storage Configuration document](configure-storage-class.md) to select an appropriate storage class for each component according to the storage classes supported by the current Kubernetes cluster and usage scenario.

> **Note:**
>
> When you create the TiDB cluster, if you set a storage class that does not exist in the Kubernetes cluster, then the TiDB cluster creation goes to the Pending state. In this situation, you must [destroy the TiDB cluster on Kubernetes](destroy-a-tidb-cluster.md) and retry the creation.

#### Multiple disks mounting

TiDB Operator supports mounting multiple PVs for PD, TiDB, TiKV, and TiCDC, which can be used for data writing for different purposes.

You can configure the `storageVolumes` field for each component to describe multiple user-customized PVs.

The meanings of the related fields are as follows:

- `storageVolume.name`: The name of the PV.
- `storageVolume.storageClassName`: The StorageClass that the PV uses. If not configured, `spec.pd/tidb/tikv/ticdc.storageClassName` will be used.
- `storageVolume.storageSize`: The storage size of the requested PV.
- `storageVolume.mountPath`: The path of the container to mount the PV to.

For example:

<SimpleTab>

<div label="TiKV">

To mount multiple PVs for TiKV:


```yaml
  tikv:
    ...
    config: |
      [rocksdb]
        wal-dir = "/data_sbi/tikv/wal"
      [titan]
        dirname = "/data_sbj/titan/data"
    storageVolumes:
    - name: wal
      storageSize: "2Gi"
      mountPath: "/data_sbi/tikv/wal"
    - name: titan
      storageSize: "2Gi"
      mountPath: "/data_sbj/titan/data"
```

</div>

<div label="TiDB">

To mount multiple PVs for TiDB:


```yaml
  tidb:
    config: |
      path = "/tidb/data"
      [log.file]
        filename = "/tidb/log/tidb.log"
    storageVolumes:
    - name: data
      storageSize: "2Gi"
      mountPath: "/tidb/data"
    - name: log
      storageSize: "2Gi"
      mountPath: "/tidb/log"
```

</div>

<div label="PD">

To mount multiple PVs for PD:


```yaml
  pd:
    config: |
      data-dir = "/pd/data"
      [log.file]
        filename = "/pd/log/pd.log"
    storageVolumes:
    - name: data
      storageSize: "10Gi"
      mountPath: "/pd/data"
    - name: log
      storageSize: "10Gi"
      mountPath: "/pd/log"
```

</div>

<div label="TiCDC">

To mount multiple PVs for TiCDC:


```yaml
  ticdc:
    ...
    config:
      dataDir: /ticdc/data
      logFile: /ticdc/log/cdc.log
    storageVolumes:
    - name: data
      storageSize: "10Gi"
      storageClassName: local-storage
      mountPath: "/ticdc/data"
    - name: log
      storageSize: "10Gi"
      storageClassName: local-storage
      mountPath: "/ticdc/log"
```

</div>

<div label="PD microservices">

To mount multiple PVs for PD microservices (taking the `tso` microservice as an example):

> **Note:**
>
> Starting from v8.0.0, PD supports the [microservice mode](https://docs.pingcap.com/tidb/dev/pd-microservices) (experimental).

```yaml
  pd:
    mode: "ms"
  pdms:
  - name: "tso"
    config: |
      [log.file]
        filename = "/pdms/log/tso.log"
    storageVolumes:
    - name: log
      storageSize: "10Gi"
      mountPath: "/pdms/log"
```

</div>
</SimpleTab>

> **Note:**
>
> TiDB Operator uses some mount paths by default. For example, it mounts `EmptyDir` to the `/var/log/tidb` directory for the TiDB Pod. Therefore, avoid duplicate `mountPath` when you configure `storageVolumes`.

### HostNetwork

For PD, TiKV, TiDB, TiFlash, TiProxy, TiCDC, and Pump, you can configure the Pods to use the host namespace [`HostNetwork`](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-s-dns-policy).

To enable `HostNetwork` for all supported components, configure `spec.hostNetwork: true`.

To enable `HostNetwork` for specified components, configure `hostNetwork: true` for the components.

### Discovery

TiDB Operator starts a Discovery service for each TiDB cluster. The Discovery service can return the corresponding startup parameters for each PD Pod to support the startup of the PD cluster. You can configure resources of the Discovery service using `spec.discovery`. For details, see [Managing Resources for Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/).

A `spec.discovery` configuration example is as follows:

```yaml
spec:
  discovery:
    limits:
      cpu: "0.2"
    requests:
      cpu: "0.2"
  ...
```

### Cluster topology

#### PD/TiKV/TiDB

The deployed cluster topology by default has three PD Pods, three TiKV Pods, and two TiDB Pods. In this deployment topology, the scheduler extender of TiDB Operator requires at least three nodes in the Kubernetes cluster to provide high availability. You can modify the `replicas` configuration to change the number of pods for each component.

> **Note:**
>
> If the number of Kubernetes cluster nodes is less than three, one PD Pod goes to the Pending state, and neither TiKV Pods nor TiDB Pods are created. When the number of nodes in the Kubernetes cluster is less than three, to start the TiDB cluster, you can reduce the number of PD Pods in the default deployment to `1`.

#### Enable PD microservices

> **Note:**
>
> Starting from v8.0.0, PD supports the [microservice mode](https://docs.pingcap.com/tidb/dev/pd-microservices) (experimental).

To enable PD microservices in your cluster, configure `spec.pd.mode` and `spec.pdms` in the `${cluster_name}/tidb-cluster.yaml` file:

```yaml
spec:
  pd:
    mode: "ms"
  pdms:
  - name: "tso"
    baseImage: pingcap/pd
    replicas: 2
  - name: "scheduling"
    baseImage: pingcap/pd
    replicas: 1
```

- `spec.pd.mode` is used to enable or disable PD microservices. Setting it to `"ms"` enables PD microservices, while setting it to `""` or removing this field disables PD microservices.
- `spec.pdms.config` is used to configure PD microservices, and the specific configuration parameters are the same as `spec.pd.config`. To get all the parameters that can be configured for PD microservices, see the [PD configuration file](https://docs.pingcap.com/tidb/stable/pd-configuration-file).

#### Enable TiProxy

The deployment method is the same as that of PD. In addition, you need to modify `spec.tiproxy` to manually specify the number of TiProxy components.

```yaml
  tiproxy:
    baseImage: pingcap/tiproxy
    replicas: 3
    config:
```

When deploying TiProxy, you also need to configure additional parameters for TiDB. For detailed configuration steps, refer to [Deploy TiProxy Load Balancer for an Existing TiDB Cluster](deploy-tiproxy.md).

#### Enable TiFlash

If you want to enable TiFlash in the cluster, configure `spec.pd.config.replication.enable-placement-rules: true` and configure `spec.tiflash` in the `${cluster_name}/tidb-cluster.yaml` file as follows:

```yaml
  pd:
    config: |
      ...
      [replication]
      enable-placement-rules = true
  tiflash:
    baseImage: pingcap/tiflash
    maxFailoverCount: 0
    replicas: 1
    storageClaims:
    - resources:
        requests:
          storage: 100Gi
      storageClassName: local-storage
```

TiFlash supports mounting multiple Persistent Volumes (PVs). If you want to configure multiple PVs for TiFlash, configure multiple `resources` in `tiflash.storageClaims`, each `resources` with a separate `storage request` and `storageClassName`. For example:

```yaml
  tiflash:
    baseImage: pingcap/tiflash
    maxFailoverCount: 0
    replicas: 1
    storageClaims:
    - resources:
        requests:
          storage: 100Gi
      storageClassName: local-storage
    - resources:
        requests:
          storage: 100Gi
      storageClassName: local-storage
```

TiFlash mounts all PVs to directories such as `/data0` and `/data1` in the container in the order of configuration. TiFlash has four log files. The proxy log is printed in the standard output of the container. The other three logs are stored in the disk under the `/data0` directory by default, which are `/data0/logs/flash_cluster_manager.log`, `/ data0/logs/error.log`, `/data0/logs/server.log`. To modify the log storage path, refer to [Configure TiFlash parameters](#configure-tiflash-parameters).

> **Warning:**
>
> Since TiDB Operator will mount PVs automatically in the **order** of the items in the `storageClaims` list, if you need to add more disks to TiFlash, make sure to append the new item only to the **end** of the original items, and **DO NOT** modify the order of the original items.

#### Enable TiCDC

If you want to enable TiCDC in the cluster, you can add TiCDC spec to the `TiDBCluster` CR. For example:

```yaml
  spec:
    ticdc:
      baseImage: pingcap/ticdc
      replicas: 3
```

### Configure TiDB components

This section introduces how to configure the parameters of TiDB/TiKV/PD/TiProxy/TiFlash/TiCDC.

#### Configure TiDB parameters

TiDB parameters can be configured by `spec.tidb.config` in TidbCluster Custom Resource.

For example:

```yaml
spec:
  tidb:
    config: |
      split-table = true
      oom-action = "log"
```

For all the configurable parameters of TiDB, refer to [TiDB Configuration File](https://docs.pingcap.com/tidb/stable/tidb-configuration-file).

> **Note:**
>
> If you deploy your TiDB cluster using CR, make sure that `Config: {}` is set, no matter you want to modify `config` or not. Otherwise, TiDB components might not be started successfully. This step is meant to be compatible with `Helm` deployment.

#### Configure TiKV parameters

TiKV parameters can be configured by `spec.tikv.config` in TidbCluster Custom Resource.

For example:

```yaml
spec:
  tikv:
    config: |
      [storage]
        [storage.block-cache]
          capacity = "16GB"
      [log.file]
        max-days = 30
        max-backups = 30
```

For all the configurable parameters of TiKV, refer to [TiKV Configuration File](https://docs.pingcap.com/tidb/stable/tikv-configuration-file).

> **Note:**
>
> - If you deploy your TiDB cluster using CR, make sure that `Config: {}` is set, no matter you want to modify `config` or not. Otherwise, TiKV components might not be started successfully. This step is meant to be compatible with `Helm` deployment.
> - TiKV RocksDB logs are stored in the `/var/lib/tikv` data directory by default. It is recommended that you configure `max-days` and `max-backups` to automatically clean log files.
> - You can also use the `separateRocksDBLog` configuration item to configure TiKV to output RocksDB logs to stdout through a sidecar container. For more information, see the [TiDB Cluster example](https://github.com/pingcap/tidb-operator/blob/master/examples/advanced/tidb-cluster.yaml).

#### Configure PD parameters

PD parameters can be configured by `spec.pd.config` in TidbCluster Custom Resource.

For example:

```yaml
spec:
  pd:
    config: |
      lease = 3
      enable-prevote = true
```

For all the configurable parameters of PD, refer to [PD Configuration File](https://docs.pingcap.com/tidb/stable/pd-configuration-file).

> **Note:**
>
> - If you deploy your TiDB cluster using CR, make sure that `Config: {}` is set, no matter you want to modify `config` or not. Otherwise, PD components might not be started successfully. This step is meant to be compatible with `Helm` deployment.
> - After the cluster is started for the first time, some PD configuration items are persisted in etcd. The persisted configuration in etcd takes precedence over that in PD. Therefore, after the first start, you cannot modify some PD configuration using parameters. You need to dynamically modify the configuration using SQL statements, pd-ctl, or PD server API. Currently, among all the configuration items listed in [Modify PD configuration online](https://docs.pingcap.com/tidb/stable/dynamic-config#modify-pd-configuration-online), except `log.level`, all the other configuration items cannot be modified using parameters after the first start.

##### Configure PD microservices

> **Note:**
>
> Starting from v8.0.0, PD supports the [microservice mode](https://docs.pingcap.com/tidb/dev/pd-microservices) (experimental).

You can configure PD microservice using the `spec.pd.mode` and `spec.pdms` parameters of the TidbCluster CR. Currently, PD supports two microservices: the `tso` microservice and the `scheduling` microservice. The configuration example is as follows:

```yaml
spec:
  pd:
    mode: "ms"
  pdms:
  - name: "tso"
    baseImage: pingcap/pd
    replicas: 2
    config: |
      [log.file]
        filename = "/pdms/log/tso.log"
  - name: "scheduling"
    baseImage: pingcap/pd
    replicas: 1
    config: |
      [log.file]
        filename = "/pdms/log/scheduling.log"
```

In the preceding configuration, `spec.pdms` is used to configure PD microservices, and the specific configuration parameters are the same as `spec.pd.config`. To get all the parameters that can be configured for PD microservices, see the [PD configuration file](https://docs.pingcap.com/tidb/stable/pd-configuration-file).

> **Note:**
>
> - If you deploy your TiDB cluster using CR, make sure that `config: {}` is set, no matter you want to modify `config` or not. Otherwise, PD microservice components might fail to start. This step is meant to be compatible with `Helm` deployment.
> - If you enable the PD microservice mode when you deploy a TiDB cluster, some configuration items of PD microservices are persisted in etcd. The persisted configuration in etcd takes precedence over that in PD.
> - If you enable the PD microservice mode for an existing TiDB cluster, some configuration items of PD microservices adopt the same values in PD configuration and are persisted in etcd. The persisted configuration in etcd takes precedence over that in PD.
> - Hence, after the first startup of PD microservices, you cannot modify these configuration items using parameters. Instead, you can modify them dynamically using [SQL statements](https://docs.pingcap.com/tidb/stable/dynamic-config#modify-pd-configuration-dynamically), [pd-ctl](https://docs.pingcap.com/tidb/stable/pd-control#config-show--set-option-value--placement-rules), or PD server API. Currently, among all the configuration items listed in [Modify PD configuration dynamically](https://docs.pingcap.com/tidb/stable/dynamic-config#modify-pd-configuration-dynamically), except `log.level`, all the other configuration items cannot be modified using parameters after the first startup of PD microservices.

#### Configure TiProxy parameters

TiProxy parameters can be configured by `spec.tiproxy.config` in TidbCluster Custom Resource.

For example:

```yaml
spec:
  tiproxy:
    config: |
      [log]
      level = "info"
```

For all the configurable parameters of TiProxy, refer to [TiProxy Configuration File](https://docs.pingcap.com/tidb/stable/tiproxy-configuration).

#### Configure TiFlash parameters

TiFlash parameters can be configured by `spec.tiflash.config` in TidbCluster Custom Resource.

For example:

```yaml
spec:
  tiflash:
    config:
      config: |
        [flash]
          [flash.flash_cluster]
            log = "/data0/logs/flash_cluster_manager.log"
        [logger]
          count = 10
          level = "information"
          errorlog = "/data0/logs/error.log"
          log = "/data0/logs/server.log"
```

For all the configurable parameters of TiFlash, refer to [TiFlash Configuration File](https://docs.pingcap.com/tidb/stable/tiflash-configuration).

#### Configure TiCDC start parameters

You can configure TiCDC start parameters through `spec.ticdc.config` in TidbCluster Custom Resource.

For example:

For TiDB Operator v1.2.0-rc.2 and later versions, configure the parameters in the TOML format as follows:

```yaml
spec:
  ticdc:
    config: |
      gc-ttl = 86400
      log-level = "info"
```

For TiDB Operator versions earlier than v1.2.0-rc.2, configure the parameters in the YAML format as follows:

```yaml
spec:
  ticdc:
    config:
      timezone: UTC
      gcTTL: 86400
      logLevel: info
```

For all configurable start parameters of TiCDC, see [TiCDC configuration](https://github.com/pingcap/tiflow/blob/bf29e42c75ae08ce74fbba102fe78a0018c9d2ea/pkg/cmd/util/ticdc.toml).

#### Configure automatic failover thresholds of PD, TiDB, TiKV, and TiFlash

The [automatic failover](use-auto-failover.md) feature is enabled by default in TiDB Operator. When the Pods of PD, TiDB, TiKV, TiFlash fail or the corresponding nodes fail, TiDB Operator performs failover automatically and replenish the number of Pod replicas by scaling the corresponding components.

To avoid that the automatic failover feature creates too many Pods, you can configure the threshold of the maximum number of Pods that TiDB Operator can create during failover for each component. The default threshold is `3`. If the threshold for a component is configured to `0`, it means that the automatic failover feature is disabled for this component. An example configuration is as follows:

```yaml
  pd:
    maxFailoverCount: 3
  tidb:
    maxFailoverCount: 3
  tikv:
    maxFailoverCount: 3
  tiflash:
    maxFailoverCount: 3
```

> **Note:**
>
> For the following cases, configure `maxFailoverCount: 0` explicitly:
>
> - The Kubernetes cluster does not have enough resources for TiDB Operator to scale out the new Pod. In such cases, the new Pod will be in the Pending state.
> - You do not want to enable the automatic failover function.

### Configure graceful upgrade for TiDB cluster

When you perform a rolling update to the TiDB cluster, Kubernetes sends a [`TERM`](https://kubernetes.io/docs/concepts/workloads/pods/pod/#termination-of-pods) signal to the TiDB server before it stops the TiDB Pod. When the TiDB server receives the `TERM` signal, it tries to wait for all connections to close. After 15 seconds, the TiDB server forcibly closes all the connections and exits the process.

You can enable this feature by configuring the following items:

- `spec.tidb.terminationGracePeriodSeconds`: The longest tolerable duration to delete the old TiDB Pod during the rolling upgrade. If this duration is exceeded, the TiDB Pod will be deleted forcibly.
- `spec.tidb.lifecycle`: Sets the `preStop` hook for the TiDB Pod, which is the operation executed before the TiDB server stops.

```yaml
spec:
  tidb:
    terminationGracePeriodSeconds: 60
    lifecycle:
      preStop:
        exec:
          command:
          - /bin/sh
          - -c
          - "sleep 10 && kill -QUIT 1"
```

The YAML file above:

- Sets the longest tolerable duration to delete the TiDB Pod to 60 seconds. If the client does not close the connections after 60 seconds, these connections will be closed forcibly. You can adjust the value according to your needs.
- Sets the value of `preStop` hook to `sleep 10 && kill -QUIT 1`. Here `PID 1` refers to the PID of the TiDB server process in the TiDB Pod. When the TiDB server process receives the signal, it exits only after all the connections are closed by the client.

When Kubernetes deletes the TiDB Pod, it also removes the TiDB node from the service endpoints. This is to ensure that the new connection is not established to this TiDB node. However, because this process is asynchronous, you can make the system sleep for a few seconds before you send the `kill` signal, which makes sure that the TiDB node is removed from the endpoints.

### Configure graceful upgrade for TiKV cluster

During TiKV upgrade, TiDB Operator evicts all Region leaders from TiKV Pod before restarting TiKV Pod. Only after the eviction is completed (which means the number of Region leaders on TiKV Pod drops to 0) or the eviction exceeds the specified timeout (1500 minutes by default), TiKV Pod is restarted. If TiKV has fewer than 2 replicas, TiDB Operator forces an upgrade without waiting for the timeout.

If the eviction of Region leaders exceeds the specified timeout, restarting TiKV Pod causes issues such as failures of some requests or more latency. To avoid the issues, you can configure the timeout `spec.tikv.evictLeaderTimeout` (1500 minutes by default) to a larger value. For example:

```
spec:
  tikv:
    evictLeaderTimeout: 10000m
```

> **Warning:**
>
> If the TiKV version is earlier than 4.0.14 or 5.0.3, due to [a bug of TiKV](https://github.com/tikv/tikv/pull/10364), you need to configure the timeout `spec.tikv.evictLeaderTimeout` as large as possible to ensure that all Region leaders on the TiKV Pod can be evicted within the timeout. If you are not sure about the proper value, greater than '1500m' is recommended.

### Configure graceful upgrade for TiCDC cluster

> **Note:**
>
> - If the TiCDC version is earlier than v6.3.0, TiDB Operator forces an upgrade on TiCDC, which might cause replication latency increase.
> - The feature is available since TiDB Operator v1.3.8.

During TiCDC upgrade, TiDB Operator drains all replication workloads from TiCDC Pod before restarting TiCDC Pod. Only after the draining is completed or the draining exceeds the specified timeout (10 minutes by default), TiCDC Pod is restarted. If TiCDC has fewer than 2 instances, TiDB Operator forces an upgrade without waiting for the timeout.

If the draining exceeds the specified timeout, restarting TiCDC Pod causes issues such as more replication latency. To avoid the issues, you can configure the timeout `spec.ticdc.gracefulShutdownTimeout` (10 minutes by default) to a larger value. For example:

```
spec:
  ticdc:
    gracefulShutdownTimeout: 100m
```

### Configure PV for TiDB slow logs

By default, TiDB Operator creates a `slowlog` volume (which is an `EmptyDir`) to store the slow logs, mounts the `slowlog` volume to `/var/log/tidb`, and prints slow logs in the `stdout` through a sidecar container.

> **Warning:**
>
> By default, after a Pod is deleted (for example, rolling update), the slow query logs stored using the `EmptyDir` volume are lost. Make sure that a log collection solution has been deployed in the Kubernetes cluster to collect logs of all containers. If you do not deploy such a log collection solution, you **must** make the following configuration to use a persistent volume to store the slow query logs.

If you want to use a separate PV to store the slow logs, you can specify the name of the PV in `spec.tidb.slowLogVolumeName`, and then configure the PV in `spec.tidb.storageVolumes` or `spec.tidb.additionalVolumes`.

This section shows how to configure PV using `spec.tidb.storageVolumes` or `spec.tidb.additionalVolumes`.

#### Configure using `spec.tidb.storageVolumes`

Configure the `TidbCluster` CR as the following example. In the example, TiDB Operator uses the `${volumeName}` PV to store slow logs. The log file path is `${mountPath}/${volumeName}`.

For how to configure the `spec.tidb.storageVolumes` field, refer to [Multiple disks mounting](#multiple-disks-mounting).

> **Warning:**
>
> You need to configure `storageVolumes` before creating the cluster. After the cluster is created, adding or removing `storageVolumes` is no longer supported. For the `storageVolumes` already configured, except for increasing `storageVolume.storageSize`, other modifications are not supported. To increase `storageVolume.storageSize`, you need to make sure that the corresponding StorageClass supports [dynamic expansion](https://kubernetes.io/blog/2018/07/12/resizing-persistent-volumes-using-kubernetes/).


```yaml
  tidb:
    ...
    separateSlowLog: true  # can be ignored
    slowLogVolumeName: ${volumeName}
    storageVolumes:
      # name must be consistent with slowLogVolumeName
      - name: ${volumeName}
        storageClassName: ${storageClass}
        storageSize: "1Gi"
        mountPath: ${mountPath}
```

#### Configure using `spec.tidb.additionalVolumes`

In the following example, NFS is used as the storage, and TiDB Operator uses the `${volumeName}` PV to store slow logs. The log file path is `${mountPath}/${volumeName}`.

For the supported PV types, refer to [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#types-of-persistent-volumes).


```yaml
  tidb:
    ...
    separateSlowLog: true  # can be ignored
    slowLogVolumeName: ${volumeName}
    additionalVolumes:
    # name must be consistent with slowLogVolumeName
    - name: ${volumeName}
      nfs:
        server: 192.168.0.2
        path: /nfs
    additionalVolumeMounts:
    # name must be consistent with slowLogVolumeName
    - name: ${volumeName}
      mountPath: ${mountPath}
```

### Configure TiDB service

You need to configure `spec.tidb.service` so that TiDB Operator creates a service for TiDB. You can configure Service with different types according to the scenarios, such as `ClusterIP`, `NodePort`, `LoadBalancer`, and so on.

#### General configurations

Different types of services share some general configurations as follows:

* `spec.tidb.service.annotations`: the annotation added to the Service resource.
* `spec.tidb.service.labels`: the labels added to the Service resource.

#### ClusterIP

`ClusterIP` exposes services through the internal IP of the cluster. When selecting this type of service, you can only access it within the cluster using ClusterIP or the Service domain name (`${cluster_name}-tidb.${namespace}`).

```yaml
spec:
  ...
  tidb:
    service:
      type: ClusterIP
```

#### NodePort

If there is no LoadBalancer, you can choose to expose the service through NodePort. NodePort exposes services through the node's IP and static port. You can access a NodePort service from outside of the cluster by requesting `NodeIP + NodePort`.

```yaml
spec:
  ...
  tidb:
    service:
      type: NodePort
      # externalTrafficPolicy: Local
```

NodePort has two modes:

- `externalTrafficPolicy=Cluster`: All machines in the cluster allocate a NodePort port to TiDB, which is the default value.

    When using the `Cluster` mode, you can access the TiDB service through the IP and NodePort of any machine. If there is no TiDB Pod on the machine, the corresponding request will be forwarded to the machine with TiDB Pod.

    > **Note:**
    >
    > In this mode, the request source IP obtained by the TiDB service is the host IP, not the real client source IP, so access control based on the client source IP is not available in this mode.

- `externalTrafficPolicy=Local`: Only the machine that TiDB is running on allocates a NodePort port to access the local TiDB instance.

#### LoadBalancer

If the TiDB cluster runs in an environment with LoadBalancer, such as on Google Cloud or AWS, it is recommended to use the LoadBalancer feature of these cloud platforms by setting `tidb.service.type=LoadBalancer`.

```yaml
spec:
  ...
  tidb:
    service:
      annotations:
        cloud.google.com/load-balancer-type: "Internal"
      externalTrafficPolicy: Local
      type: LoadBalancer
```

See [Kubernetes Service Documentation](https://kubernetes.io/docs/concepts/services-networking/service/) to know more about the features of Service and what LoadBalancer in the cloud platform supports.

If TiProxy is specified, `tiproxy-api` and `tiproxy-sql` services are also automatically created for use.

### IPv6 Support

Starting v6.5.1, TiDB supports using IPv6 addresses for all network connections. If you deploy TiDB using TiDB Operator v1.4.3 or later versions, you can enable the TiDB cluster to listen on IPv6 addresses by configuring `spec.preferIPv6` to `true`.

```yaml
spec:
  preferIPv6: true
  # ...
```

> **Warning:**
>
> This configuration can only be applied when deploying the TiDB cluster and cannot be enabled on deployed clusters, as it may cause the cluster to become unavailable.

## Configure high availability

> **Note:**
>
> TiDB Operator provides a custom scheduler that guarantees TiDB service can tolerate host-level failures through the specified scheduling algorithm. Currently, the TiDB cluster uses this scheduler as the default scheduler, which is configured through the item `spec.schedulerName`. This section focuses on configuring a TiDB cluster to tolerate failures at other levels such as rack, zone, or region. This section is optional.

TiDB is a distributed database and its high availability must ensure that when any physical topology node fails, not only the service is unaffected, but also the data is complete and available. The two configurations of high availability are described separately as follows.

### High availability of TiDB service

#### Use nodeSelector to schedule Pods

By configuring the `nodeSelector` field of each component, you can specify the specific nodes that the component Pods are scheduled onto. For details on `nodeSelector`, refer to [nodeSelector](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector).

```yaml
apiVersion: pingcap.com/v1alpha1
kind: TidbCluster
# ...
spec:
  pd:
    nodeSelector:
      node-role.kubernetes.io/pd: true
    # ...
  tikv:
    nodeSelector:
      node-role.kubernetes.io/tikv: true
    # ...
  tidb:
    nodeSelector:
      node-role.kubernetes.io/tidb: true
    # ...
```

#### Use tolerations to schedule Pods

By configuring the `tolerations` field of each component, you can allow the component Pods to schedule onto nodes with matching [taints](https://kubernetes.io/docs/reference/glossary/?all=true#term-taint). For details on taints and tolerations, refer to [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/).

```yaml
apiVersion: pingcap.com/v1alpha1
kind: TidbCluster
# ...
spec:
  pd:
    tolerations:
      - effect: NoSchedule
        key: dedicated
        operator: Equal
        value: pd
    # ...
  tikv:
    tolerations:
      - effect: NoSchedule
        key: dedicated
        operator: Equal
        value: tikv
    # ...
  tidb:
    tolerations:
      - effect: NoSchedule
        key: dedicated
        operator: Equal
        value: tidb
    # ...
```

#### Use affinity to schedule Pods

By configuring `PodAntiAffinity`, you can avoid the situation in which different instances of the same component are deployed on the same physical topology node. In this way, disaster recovery (high availability) is achieved. For the user guide of Affinity, see [Affinity & AntiAffinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity).

The following is an example of a typical service high availability setup:


```yaml
affinity:
 podAntiAffinity:
   preferredDuringSchedulingIgnoredDuringExecution:
   # this term works when the nodes have the label named region
   - weight: 10
     podAffinityTerm:
       labelSelector:
         matchLabels:
           app.kubernetes.io/instance: ${cluster_name}
           app.kubernetes.io/component: "pd"
       topologyKey: "region"
       namespaces:
       - ${namespace}
   # this term works when the nodes have the label named zone
   - weight: 20
     podAffinityTerm:
       labelSelector:
         matchLabels:
           app.kubernetes.io/instance: ${cluster_name}
           app.kubernetes.io/component: "pd"
       topologyKey: "zone"
       namespaces:
       - ${namespace}
   # this term works when the nodes have the label named rack
   - weight: 40
     podAffinityTerm:
       labelSelector:
         matchLabels:
           app.kubernetes.io/instance: ${cluster_name}
           app.kubernetes.io/component: "pd"
       topologyKey: "rack"
       namespaces:
       - ${namespace}
   # this term works when the nodes have the label named kubernetes.io/hostname
   - weight: 80
     podAffinityTerm:
       labelSelector:
         matchLabels:
           app.kubernetes.io/instance: ${cluster_name}
           app.kubernetes.io/component: "pd"
       topologyKey: "kubernetes.io/hostname"
       namespaces:
       - ${namespace}
```

#### Use topologySpreadConstraints to make pods evenly spread

By configuring `topologySpreadConstraints`, you can make pods evenly spread in different topologies. For instructions about configuring `topologySpreadConstraints`, see [Pod Topology Spread Constraints](https://kubernetes.io/docs/concepts/workloads/pods/pod-topology-spread-constraints/).

You can either configure `topologySpreadConstraints` at a cluster level (`spec.topologySpreadConstraints`) for all components or at a component level (such as `spec.tidb.topologySpreadConstraints`) for specific components.

The following is an example configuration:


```yaml
topologySpreadConstraints:
- topologyKey: kubernetes.io/hostname
- topologyKey: topology.kubernetes.io/zone
```

The example configuration can make pods of the same component evenly spread on different zones and nodes.

Currently, `topologySpreadConstraints` only supports the configuration of the `topologyKey` field. In the pod spec, the above example configuration will be automatically expanded as follows:

```yaml
topologySpreadConstraints:
- topologyKey: kubernetes.io/hostname
  maxSkew: 1
  whenUnsatisfiable: DoNotSchedule
  labelSelector: <object>
- topologyKey: topology.kubernetes.io/zone
  maxSkew: 1
  whenUnsatisfiable: DoNotSchedule
  labelSelector: <object>
```

### High availability of data

Before configuring the high availability of data, read [Information Configuration of the Cluster Typology](https://docs.pingcap.com/tidb/stable/schedule-replicas-by-topology-labels) which describes how high availability of TiDB cluster is implemented.

To add the data high availability feature on Kubernetes:

* Set the label collection of topological location for PD.

    Replace the `location-labels` information in the `pd.config` with the label collection that describes the topological location on the nodes in the Kubernetes cluster.

    > **Note:**
    >
    > * For PD versions < v3.0.9, the `/` in the label name is not supported.
    > * If you configure `host` in the `location-labels`, TiDB Operator will get the value from the `kubernetes.io/hostname` in the node label.

* Set the topological information of the Node where the TiKV node is located.

    TiDB Operator automatically obtains the topological information of the Node for TiKV and calls the PD interface to set this information as the information of TiKV's store labels. Based on this topological information, the TiDB cluster schedules the replicas of the data.

    If the Node of the current Kubernetes cluster does not have a label indicating the topological location, or if the existing label name of topology contains `/`, you can manually add a label to the Node by running the following command:

    
    ```shell
    kubectl label node ${node_name} region=${region_name} zone=${zone_name} rack=${rack_name} kubernetes.io/hostname=${host_name}
    ```

    In the command above, `region`, `zone`, `rack`, and `kubernetes.io/hostname` are just examples. The name and number of the label to be added can be arbitrarily defined, as long as it conforms to the specification and is consistent with the labels set by `location-labels` in `pd.config`.

* Set the topological information of the Node where the TiDB node is located.

    Since TiDB Operator v1.4.0, if the deployed TiDB version >= v6.3.0, TiDB Operator automatically obtains the topological information of the Node for TiDB and calls the corresponding interface of the TiDB server to set this information as TiDB's labels. Based on these labels, TiDB sends the [Follower Read](https://docs.pingcap.com/tidb/stable/follower-read) requests to the correct replicas.

    Currently, TiDB Operator automatically sets the labels for the TiDB server corresponding to the `location-labels` in `pd.config`. TiDB depends on the `zone` label to support some features of Follower Read. TiDB Operator obtains the value of `zone`, `failure-domain.beta.kubernetes.io/zone`, and `topology.kubernetes.io/zone` labels as `zone`. TiDB Operator only sets labels of the node where the TiDB server is located and ignores other labels.

* Set the topological information of the Node where the TiProxy node is located.

    Starting from TiDB Operator v1.6.0, if the deployed TiProxy version >= v1.1.0, TiDB Operator automatically obtains the topological information of the Node for TiProxy and calls the corresponding interface of the TiProxy to set this information as TiProxy's labels. Based on these labels, TiProxy prioritizes forwarding requests to a local TiDB server.

    Currently, TiDB Operator automatically sets the labels for the TiProxy node corresponding to the `location-labels` in `pd.config`. TiProxy depends on the `zone` label to forward requests to a local TiDB server. TiDB Operator obtains the value of `zone`, `failure-domain.beta.kubernetes.io/zone`, and `topology.kubernetes.io/zone` labels as `zone`. TiDB Operator only sets labels of the node where the TiProxy is located and ignores other labels.

Starting from v1.4.0, when setting labels for TiKV and TiDB nodes, TiDB Operator supports setting shortened aliases for some labels provided by Kubernetes by default. In some scenarios, using aliases can help optimize the scheduling performance of PD. When you use TiDB Operator to set aliases for the `location-labels` of PD, if there are no corresponding labels for a Kubernetes node, then TiDB Operator uses the original labels automatically.

Currently, TiDB Operator supports the following label aliases:

- `region`: corresponds to `topology.kubernetes.io/region` and `failure-domain.beta.kubernetes.io/region`.
- `zone`: corresponds to `topology.kubernetes.io/zone` and `failure-domain.beta.kubernetes.io/zone`.
- `host`: corresponds to `kubernetes.io/hostname`.

For example, if labels such as `region`, `zone`, and `host` are not set on each node of Kubernetes, setting the `location-labels` of PD as `["topology.kubernetes.io/region", "topology.kubernetes.io/zone", "kubernetes.io/hostname"]` is the same as `["region", "zone", "host"]`.
