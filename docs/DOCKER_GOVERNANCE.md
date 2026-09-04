# Docker / WSL2 空间治理规范

## 1. 目的与适用范围

本规范用于 Windows Docker Desktop + WSL2 项目，尤其是包含 PostgreSQL、EMQX、Node-RED 和应用服务的 IoT 运行环境。

目标不是把磁盘清空，而是让每一块空间都有明确归属，并在达到风险阈值前自动清理可重建内容，在达到硬门禁后停止继续扩大风险。

本规范不替代数据库备份、业务数据保留策略、Docker Desktop 官方维护流程或项目自己的发布审批。

候选测试环境的版本、缓存、Compose、Secret、迁移、UI 和回滚顺序以 [`docs/iot/候选环境交付契约.md`](iot/候选环境交付契约.md) 为唯一入口；本文仅保留 Windows Docker/WSL2 的通用空间治理。

## 2. 两层存储模型

Docker 使用的空间必须区分为两层：

```text
Windows / D 盘
├─ Docker Desktop WSL2 VHDX
│  ├─ 镜像层
│  ├─ 容器可写层
│  ├─ BuildKit 构建缓存
│  ├─ 未绑定的 Docker Volume
│  └─ WSL2 文件系统空闲区
└─ 项目绑定目录
   ├─ PostgreSQL 数据
   ├─ EMQX 数据和日志
   ├─ Node-RED /data、flow 和模块
   ├─ 备份、证据和运行配置
   └─ 其他业务持久化数据
```

清理 VHDX 内部对象不会保证 Windows 看到的 VHDX 文件立即变小；VHDX 压缩必须在 Docker Desktop 和所有 WSL 发行版完全退出后，在维护窗口执行。

项目必须在 `environment.manifest` 中登记：

- VHDX 绝对路径；
- 绑定目录、Volume 和 owner；
- Git、Compose、镜像 digest 和环境编码；
- 数据、备份、证据和恢复入口。

## 3. 资源分类

| 分类 | 默认处理 | 示例 |
| --- | --- | --- |
| Preserve | 保留 | 当前镜像、运行容器、数据库、EMQX、Node-RED、业务数据、Volume、备份 |
| Rebuildable | 无使用者时可定向清理 | Build Cache、编译缓存、确认无引用的临时层 |
| Deletable | 逐项核验后才允许删除 | 当前任务创建且已停止、无挂载、无回滚价值的临时资源 |
| Unknown | 不删除，登记并上报 | 无标签镜像、无 owner Volume、用途不明的 VHDX 内容 |

任何清理前必须核对容器引用、挂载、进程使用、业务 owner 和恢复路径。禁止按文件年龄、`RECLAIMABLE` 字段或名称猜测可删除性。

## 4. 推荐阈值

阈值以 GiB 为单位，必须写入环境 manifest，不写死在多个脚本中。

| 状态 | VHDX 文件 | 动作 |
| --- | ---: | --- |
| 正常 | `< 12 GiB` | 正常运行，按计划执行安全维护 |
| 警告 | `>= 12 GiB` | 停止非必要构建/拉取，执行 dry-run 和定向缓存清理 |
| 临界 | `>= 18 GiB` | 禁止新增镜像和构建，安排维护窗口，检查绑定数据增长 |
| 硬门禁 | `>= 20 GiB` | 固定启动入口 fail-closed；不得继续启动或扩容，等待人工维护 |

D 盘总空间另设项目阈值：剩余小于 80 GiB 警告，小于 40 GiB 停止高风险操作。VHDX 阈值不等于 D 盘总空间阈值。

Docker Desktop 的 WSL2 页面可能没有可设置的磁盘上限控件。此时 20 GiB 是项目门禁，不应伪称为 Docker 引擎的硬配额；固定入口和维护脚本必须阻止本项目继续启动，但不能拦截用户手工执行的所有 `docker pull` 或 `docker build`。

## 5. 日常自动维护

每个环境只允许一个带 owner 的定时任务，建议每天一次，时间写入 manifest。任务必须支持 `--dry-run` 和 `--apply`，默认不得破坏数据。

允许的默认动作：

```text
docker buildx prune --filter until=24h --force
docker image prune --filter dangling=true --force
```

必须先记录 VHDX、D 盘、`docker system df`、容器状态和 Build Cache，再执行清理；执行后回读释放量和服务状态。

默认禁止：

- `docker system prune`；
- `docker system prune --volumes`；
- `docker volume prune`、`docker container prune`；
- 删除当前 main 镜像、Volume、网络或数据库；
- 运行态 VHDX 压缩；
- 关闭外键、`TRUNCATE`、`CASCADE` 或猜测 SQL 清理业务数据；
- 以清理 Docker 空间为名删除 Node-RED flow、credentials、settings 或 modules。

## 6. 日志和业务数据治理

Compose 服务应使用有限轮转日志，例如 `json-file` 的 `max-size=10m`、`max-file=3`。日志轮转限制 Docker 内部日志，不代表绑定目录中的 EMQX、Ingester 或数据库日志会自动受限。

历史业务数据必须由独立的 retention 方案管理：

1. 先 dry-run 统计 cutoff、候选表、外键闭包和可释放空间；
2. 确认无开放窗口、未完成 job 和 pending outbox；
3. 备份并验证可恢复；
4. 使用应用层状态转换和精确外键顺序，分批短事务删除；
5. 回读删除、跳过、近 24 小时保留、表大小、WAL、`pgsql_tmp` 和业务结果。

业务数据清理不会自动缩小 VHDX，也不能替代 Docker 缓存清理。保留策略是否启用必须由业务负责人单独批准。

## 7. 启动硬门禁

固定启动入口必须在启动前读取 manifest，并执行以下门禁：

- Docker/Linux engine 可达；
- VHDX 存在且低于硬门禁；
- D 盘高于 stop 阈值；
- 当前 Git/Compose/镜像/数据路径绑定一致；
- Secret 文件只检查存在性和权限，不输出值；
- Node-RED flow、credentials、settings、modules 来源一致；
- 端口未被其他环境占用。

VHDX 达到硬门禁时，启动入口必须返回明确错误并写入脱敏回执，不能通过“重试启动”规避。已经运行的服务不应由空间维护脚本自动强杀；应由维护窗口按依赖顺序停机。

## 8. 维护窗口和 VHDX 回收

只有在用户批准的维护窗口执行：

1. 记录版本、容器、挂载、Volume、端口、VHDX、C/D 空间和数据库恢复路径；
2. 停止 Node-RED、应用写入者、EMQX、PostgreSQL；
3. 完全退出 Docker Desktop；
4. 执行 `wsl --shutdown`，确认无发行版和进程占用 VHDX；
5. 仅对 manifest 登记的绝对路径执行官方 compact；
6. 失败时保留原始错误，不删除或重建 VHDX；
7. 启动 Docker Desktop，使用固定入口恢复服务；
8. 回读健康、迁移账本、端口、OOM、重启次数、VHDX 和磁盘差值。

Docker Desktop 的 Clean/Purge 属于更高风险的重建动作，只能在确认数据库/EMQX/Node-RED 绑定目录、镜像材料和恢复入口后执行；不得把它作为日常清理。

## 9. 回执和验收

每次维护回执至少记录：

- 环境、Git、Compose、镜像 digest；
- VHDX 前后大小和阈值状态；
- `docker system df` 前后摘要；
- 清理对象、跳过对象和原因；
- 容器、网络、Volume、端口和健康状态；
- C/D 空间、OOM、重启次数、数据库迁移状态；
- 备份、回滚路径和未确认项。

“Docker 服务 healthy”不能单独表示业务数据闭环通过。业务数据、UI、PLC、MQTT 和计算验收必须按各自责任边界另行确认。

## 10. 当前 IoT 实施映射

项目运行时应提供以下能力，具体路径和阈值由 manifest 绑定：

- `Docker_空间维护.ps1`：每日安全清理、VHDX 观察和硬门禁回执；
- 固定启动入口：启动前检查 VHDX 硬门禁，达到 20 GiB 时 fail-closed；
- Compose：服务日志轮转，当前 main 镜像和数据目录不自动删除；
- 证据目录：保存治理回执和 SHA-256；
- 备份目录：保存脚本、manifest 和维护前恢复副本。

该映射不包含任何密码、Token、证书、数据库内容或 Node-RED credential 明文。
