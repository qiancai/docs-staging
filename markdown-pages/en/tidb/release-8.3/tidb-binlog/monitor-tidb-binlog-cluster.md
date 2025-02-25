---
title: TiDB Binlog Monitoring
summary: Learn how to monitor the cluster version of TiDB Binlog.
aliases: ['/tidb/dev/monitor-tidb-binlog-cluster/','/tidb/stable/monitor-tidb-binlog-cluster/','/docs/dev/tidb-binlog/monitor-tidb-binlog-cluster/','/docs/dev/reference/tidb-binlog/monitor/','/docs/dev/how-to/monitor/tidb-binlog/']
---

# TiDB Binlog Monitoring

After you have deployed TiDB Binlog successfully, you can go to the Grafana Web (default address: <http://grafana_ip:3000>, default account: admin, password: admin) to check the state of Pump and Drainer.

## Monitoring metrics

TiDB Binlog consists of two components: Pump and Drainer. This section shows the monitoring metrics of Pump and Drainer.

### Pump monitoring metrics

To understand the Pump monitoring metrics, check the following table:

| Pump monitoring metrics | Description |
| --- | --- |
| Storage Size | Records the total disk space (capacity) and the available disk space (available)|
| Metadata | Records the biggest TSO (`gc_tso`) of the binlog that each Pump node can delete, and the biggest commit TSO (`max_commit_tso`) of the saved binlog |
| Write Binlog QPS by Instance | Shows QPS of writing binlog requests received by each Pump node |
| Write Binlog Latency | Records the latency time of each Pump node writing binlog |
| Storage Write Binlog Size | Shows the size of the binlog data written by Pump |
| Storage Write Binlog Latency | Records the latency time of the Pump storage module writing binlog |
| Pump Storage Error By Type | Records the number of errors encountered by Pump, counted based on the type of error |
| Query TiKV | The number of times that Pump queries the transaction status through TiKV |

### Drainer monitoring metrics

To understand the Drainer monitoring metrics, check the following table:

| Drainer monitoring metrics | Description |
| --- | --- |
| Checkpoint TSO | Shows the biggest TSO time of the binlog that Drainer has already replicated into the downstream. You can get the lag by using the current time to subtract the binlog timestamp. But be noted that the timestamp is allocated by PD of the master cluster and is determined by the time of PD.|
| Pump Handle TSO | Records the biggest TSO time among the binlog files that Drainer obtains from each Pump node |
| Pull Binlog QPS by Pump NodeID | Shows the QPS when Drainer obtains binlog from each Pump node |
| 95% Binlog Reach Duration By Pump | Records the delay from the time when binlog is written into Pump to the time when the binlog is obtained by Drainer |
| Error By Type | Shows the number of errors encountered by Drainer, counted based on the type of error |
| SQL Query Time | Records the time it takes Drainer to execute the SQL statement in the downstream |
| Drainer Event | Shows the number of various types of events, including "ddl", "insert", "delete", "update", "flush", and "savepoint" |
| Execute Time | Records the time it takes to write binlog into the downstream syncing module |
| 95% Binlog Size | Shows the size of the binlog data that Drainer obtains from each Pump node |
| DDL Job Count | Records the number of DDL statements handled by Drainer |
| Queue Size | Records the work queue size in Drainer |

## Alert rules

This section gives the alert rules for TiDB Binlog. According to the severity level, TiDB Binlog alert rules are divided into three categories (from high to low): emergency-level, critical-level and warning-level.

### Emergency-level alerts

Emergency-level alerts are often caused by a service or node failure. Manual intervention is required immediately.

#### `binlog_pump_storage_error_count`

* Alert rule:

    `changes(binlog_pump_storage_error_count[1m]) > 0`

* Description:

    Pump fails to write the binlog data to the local storage.

* Solution:

    Check whether an error exists in the `pump_storage_error` monitoring and check the Pump log to find the causes.

### Critical-level alerts

For the critical-level alerts, a close watch on the abnormal metrics is required.

#### `binlog_drainer_checkpoint_high_delay`

* Alert rule:

    `(time() - binlog_drainer_checkpoint_tso / 1000) > 3600`

* Description:

    The delay of Drainer replication exceeds one hour.

* Solution:

    - Check whether it is too slow to obtain the data from Pump:

        You can check `handle tso` of Pump to get the time for the latest message of each Pump. Check whether a high latency exists for Pump and make sure the corresponding Pump is running normally.

    - Check whether it is too slow to replicate data in the downstream based on Drainer `event` and Drainer `execute latency`:

        - If Drainer `execute time` is too large, check the network bandwidth and latency between the machine with Drainer deployed and the machine with the target database deployed, and the state of the target database.
        - If Drainer `execute time` is not too large and Drainer `event` is too small, add `work count` and `batch` and retry.

    - If the two solutions above cannot work, [get support](/support.md) from PingCAP or the community.

### Warning-level alerts

Warning-level alerts are a reminder for an issue or error.

#### `binlog_pump_write_binlog_rpc_duration_seconds_bucket`

* Alert rule:

    `histogram_quantile(0.9, rate(binlog_pump_rpc_duration_seconds_bucket{method="WriteBinlog"}[5m])) > 1`

* Description:

    It takes too much time for Pump to handle the TiDB request of writing binlog.

* Solution:

    - Verify the disk performance pressure and check the disk performance monitoring via `node exported`.
    - If both `disk latency` and `util` are low, [get support](/support.md) from PingCAP or the community.

#### `binlog_pump_storage_write_binlog_duration_time_bucket`

* Alert rule:

    `histogram_quantile(0.9, rate(binlog_pump_storage_write_binlog_duration_time_bucket{type="batch"}[5m])) > 1`

* Description:

    The time it takes for Pump to write the local binlog to the local disk.

* Solution:

    Check the state of the local disk of Pump and fix the problem.

#### `binlog_pump_storage_available_size_less_than_20G`

* Alert rule:

    `binlog_pump_storage_storage_size_bytes{type="available"} < 20 * 1024 * 1024 * 1024`

* Description:

    The available disk space of Pump is less than 20 GB.

* Solution:

    Check whether Pump `gc_tso` is normal. If not, adjust the GC time configuration of Pump or get the corresponding Pump offline.

#### `binlog_drainer_checkpoint_tso_no_change_for_1m`

* Alert rule:

    `changes(binlog_drainer_checkpoint_tso[1m]) < 1`

* Description:

    Drainer `checkpoint` has not been updated for one minute.

* Solution:

    Check whether all the Pumps that are not offline are running normally.

#### `binlog_drainer_execute_duration_time_more_than_10s`

* Alert rule:

    `histogram_quantile(0.9, rate(binlog_drainer_execute_duration_time_bucket[1m])) > 10`

* Description:

    The transaction time it takes Drainer to replicate data to TiDB. If it is too large, the Drainer replication of data is affected.

* Solution:

    - Check the TiDB cluster state.
    - Check the Drainer log or monitor. If a DDL operation causes this problem, you can ignore it.
