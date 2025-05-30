---
title: TiDB 日志备份与 PITR 使用指南
summary: 了解 TiDB 的日志备份与 PITR 功能使用。
aliases: ['/zh/tidb/dev/pitr-usage/']
---

# TiDB 日志备份与 PITR 使用指南

全量备份包含集群某个时间点的全量数据，但是不包含其他时间点的更新数据，而 TiDB 日志备份能够将业务写入 TiDB 的数据记录及时备份到指定存储中。如果你需要灵活的选择恢复的时间点，即实现 Point-in-time recovery (PITR)，可以开启[日志备份](#开启日志备份)，并[定期执行快照（全量）备份](#定期执行全量备份)。

使用 br 命令行工具备份与恢复数据前，请先[安装 br 命令行工具](/br/br-use-overview.md#部署和使用-br)。

## 对集群进行备份

### 开启日志备份

> **注意：**
>
> - 以下场景采用 Amazon S3 Access key 和 Secret key 授权方式来进行模拟。如果使用 IAM Role 授权，需要设置 `--send-credentials-to-tikv` 为 `false`。
> - 如果使用不同存储或者其他授权方式，请参考[备份存储](/br/backup-and-restore-storages.md)来进行参数调整。

执行 `tiup br log start` 命令启动日志备份任务，一个集群只能启动一个日志备份任务。

```shell
tiup br log start --task-name=pitr --pd "${PD_IP}:2379" \
--storage 's3://backup-101/logbackup?access-key=${access-key}&secret-access-key=${secret-access-key}'
```

### 查询日志备份状态

日志备份任务启动后，会在 TiDB 集群后台持续地运行，直到你手动将其暂停。在这过程中，TiDB 变更数据将以小批量的形式定期备份到指定存储中。如果你需要查询日志备份任务当前状态，执行如下命令：

```shell
tiup br log status --task-name=pitr --pd "${PD_IP}:2379"
```

日志备份任务状态输出如下：

```
● Total 1 Tasks.
> #1 <
           name: pitr
         status: ● NORMAL
          start: 2022-05-13 11:09:40.7 +0800
            end: 2035-01-01 00:00:00 +0800
        storage: s3://backup-101/log-backup
    speed(est.): 0.00 ops/s
checkpoint[global]: 2022-05-13 11:31:47.2 +0800; gap=4m53s
```

其中，各个字段含义如下：

- `name`：Log Backup 任务的名字。
- `status`：Log Backup 的状态，包括 `NORMAL`、`PAUSED`、`ERROR`。
- `start`：Log Backup 的起始时间戳。
- `end`：Log Backup 的结束时间戳，目前这个字段并不会生效。
- `storage`：Log Backup 的外部存储的 URI。
- `speed(est.)`：Log Backup 目前的流量。这个值是取最近数秒内的流量采样估算而成。如需更加精确的流量统计，你可以通过 Grafana 的 **[TiKV-Details](/grafana-tikv-dashboard.md#tikv-details-面板)** 监控面板中的 `Log Backup` 行查看。
- `checkpoint[global]`：Log Backup 目前的进度。你可以使用 PiTR 恢复到这个时间戳前的时间点。

如果 Log Backup 任务暂停，`log status` 会输出额外的字段来展示暂停的细节，例如：

```
● Total 1 Tasks.
> #1 <
              name: pitr
            status: ○ ERROR
               <......>
        pause-time: 2025-03-14T14:35:06+08:00
    pause-operator: atelier.local
pause-operator-pid: 64618
     pause-payload: The checkpoint is at 2025-03-14T14:34:54+08:00, now it is 2025-03-14T14:35:06+08:00, the lag is too huge (11.956113s) hence pause the task to avoid impaction to the cluster
```

其中，这些额外的字段含义如下：

- `pause-time`：执行暂停操作的时间。
- `pause-operator`：执行暂停操作的机器的 hostname。
- `pause-operator-pid`：执行暂停操作的进程的 PID。
- `pause-payload`：暂停时所附带的额外信息。

如果 Log Backup 任务的暂停是由于 TiKV 发生错误导致的，你可能还会额外看到 TiKV 上报的错误：

- `error[store=*]`：TiKV 处的错误代码。
- `error-happen-at[store=*]`：在 TiKV 处发生错误的时间。
- `error-message[store=*]`：在 TiKV 处的错误消息。

### 定期执行全量备份

快照备份功能可作为全量备份的方法，运行 `tiup br backup full` 命令可以按照固定的周期（比如 2 天）进行全量备份。

```shell
tiup br backup full --pd "${PD_IP}:2379" \
--storage 's3://backup-101/snapshot-${date}?access-key=${access-key}&secret-access-key=${secret-access-key}'
```

## 进行 PITR

如果你想恢复到备份保留期内的任意时间点，可以使用 `tiup br restore point` 命令。执行该命令时，你需要指定**要恢复的时间点**、**恢复时间点之前最近的快照备份**以及**日志备份数据**。br 命令行工具会自动判断和读取恢复需要的数据，然后将这些数据依次恢复到指定的集群。

```shell
tiup br restore point --pd "${PD_IP}:2379" \
--storage='s3://backup-101/logbackup?access-key=${access-key}&secret-access-key=${secret-access-key}' \
--full-backup-storage='s3://backup-101/snapshot-${date}?access-key=${access-key}&secret-access-key=${secret-access-key}' \
--restored-ts '2022-05-15 18:00:00+0800'
```

恢复期间，可通过终端中的进度条查看进度，如下。恢复分为两个阶段：全量恢复 (Full Restore) 和日志恢复（Restore Meta Files 和 Restore KV Files）。每个阶段完成恢复后，br 命令行工具都会输出恢复耗时和恢复数据大小等信息。

```shell
Split&Scatter Region <--------------------------------------------------------------------------------------------------------------------------------------------------------> 100.00%
Download&Ingest SST <--------------------------------------------------------------------------------------------------------------------------------------------------------> 100.00%
Restore Pipeline <--------------------------------------------------------------------------------------------------------------------------------------------------------> 100.00%
*** ["Full Restore success summary"] ****** [total-take=xxx.xxxs] [restore-data-size(after-compressed)=xxx.xxx] [Size=xxxx] [BackupTS={TS}] [total-kv=xxx] [total-kv-size=xxx] [average-speed=xxx]
Restore Meta Files <--------------------------------------------------------------------------------------------------------------------------------------------------> 100.00%
Restore KV Files <----------------------------------------------------------------------------------------------------------------------------------------------------> 100.00%
*** ["restore log success summary"] [total-take=xxx.xx] [restore-from={TS}] [restore-to={TS}] [total-kv-count=xxx] [total-size=xxx]
```

## 清理过期的日志备份数据

如[使用指南概览](/br/br-use-overview.md)所述：

* 进行 PITR 不仅需要恢复时间点之前的全量备份，还需要全量备份和恢复时间点之间的日志备份，因此，对于超过备份保留期的日志备份，应执行 `tiup br log truncate` 命令删除指定时间点之前的备份。**建议只清理全量快照之前的日志备份**。

你可以按照以下步骤清理超过**备份保留期**的备份数据：

1. 查找备份保留期之外的**最近一次全量备份**。
2. 使用 `validate` 指令获取该备份对应的时间点。假如需要清理 2022/09/01 之前的备份数据，则应查找该日期之前的最近一次全量备份，且保证它不会被清理。

    ```shell
    FULL_BACKUP_TS=`tiup br validate decode --field="end-version" --storage "s3://backup-101/snapshot-${date}?access-key=${access-key}&secret-access-key=${secret-access-key}"| tail -n1`
    ```

3. 清理该快照备份 `FULL_BACKUP_TS` 之前的日志备份数据。

    ```shell
    tiup br log truncate --until=${FULL_BACKUP_TS} --storage='s3://backup-101/logbackup?access-key=${access-key}&secret-access-key=${secret-access-key}'
    ```

4. 清理该快照备份 `FULL_BACKUP_TS` 之前的快照备份数据。

    ```shell
    aws s3 rm --recursive s3://backup-101/snapshot-${date}
    ```

## PITR 的性能指标

- PITR 恢复速度，平均到单台 TiKV 节点：全量恢复 (Full Restore) 为 2 TiB/h，日志恢复（Restore Meta Files 和 Restore KV Files）为 30 GiB/h
- 使用 `tiup br log truncate` 清理过期的日志备份数据速度为 600 GB/h

> **注意：**
>
> 以上功能指标是根据下述两个场景测试得出的结论，如有出入，建议以实际测试结果为准：
>
> - 全量恢复速度 = 集群中所有 TiKV 节点恢复数据总量 /（时间 * TiKV 数量）
> - 日志恢复速度 = 集群中所有 TiKV 节点日志恢复总量 /（时间 * TiKV 数量）
>
> 外部存储中只存放单个副本中的 KV 数据，因此外部存储上的数据量并不代表集群实际恢复后的数据量。BR 恢复数据时会根据集群设置的副本数来恢复全部副本，当副本数越多时，实际恢复的数据量也就越多。
> 所有测试集群默认设置 3 副本。
> 如果想提升整体恢复的性能，可以通过根据实际情况调整 TiKV 配置文件中的 [`import.num-threads`](/tikv-configuration-file.md#import) 配置项以及 BR 命令的 [`pitr-concurrency`](/br/use-br-command-line-tool.md#常用选项) 参数。

测试场景 1（[TiDB Cloud](https://tidbcloud.com) 上部署）如下：

- TiKV 节点（8 core，16 GB 内存）数量：21
- TiKV 配置项 `import.num-threads`：8
- BR 命令参数 `pitr-concurrency`：128
- Region 数量：183,000
- 集群新增日志数据：10 GB/h
- 写入 (INSERT/UPDATE/DELETE) QPS：10,000

测试场景 2（本地部署）如下：

- TiKV 节点（8 core，64 GB 内存）数量：6
- TiKV 配置项 `import.num-threads`：8
- BR 命令参数 `pitr-concurrency`：128
- Region 数量：50,000
- 集群新增日志数据：10 GB/h
- 写入 (INSERT/UPDATE/DELETE) QPS：10,000

## 监控与告警

在日志备份任务下发后，各 TiKV 节点会持续将数据写入外部存储。此过程的监控数据可以通过 **TiKV-Details > Backup Log** 面板查看。

如需在指标异常时收到通知，可以参考[日志备份告警](/br/br-monitoring-and-alert.md#日志备份告警)配置告警规则。

## 探索更多

* [TiDB 集群备份与恢复实践示例](/br/backup-and-restore-use-cases.md)
* [`br` 命令行手册](/br/use-br-command-line-tool.md)
* [日志备份与 PITR 架构设计](/br/br-log-architecture.md)
* [备份恢复监控告警](/br/br-monitoring-and-alert.md)
