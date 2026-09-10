# Docker 治理规范（蒸馏版）

> 版本：v1.0  
> 日期：2026-09-10  
> 适用场景：单机 Ubuntu / Ubuntu VM 上，以 Docker Engine + Docker Compose 运行 Node-RED、EMQX、数据库、看板及其他中小型 IIoT 服务。  
> 证据层级：Docker 官方文档是规范依据；Docker Community Forums 只用于识别真实故障模式，不能替代官方定义。

## 1. 目标

本规范只解决六件事：

1. 新机器能按仓库内容重建服务；
2. 服务器重启后服务能自行恢复；
3. 单个容器失控时不拖垮整台主机；
4. 配置、数据和秘密信息不混在一起；
5. 每次变更可验证、可回滚、可追溯；
6. 换人或新建 Codex 线程后，仍能按同一套规则操作。

## 2. 规则等级

- **MUST**：强制规则。不满足不得进入生产环境。
- **SHOULD**：默认执行。偏离时必须在变更记录中说明原因。
- **MAY**：按项目复杂度选择。

## 3. 第一性原则

容器可以随时删除和重建；真正需要保护的是：

- Git 中的声明式配置；
- 容器外的持久化数据；
- 不进入 Git 的秘密信息；
- 能证明系统可恢复的备份与验证记录。

因此，容器本身不是资产，**可重复部署能力和数据恢复能力才是资产**。

## 4. 项目结构

每套独立服务栈 MUST 使用一个独立 Git 目录：

```text
project-name/
├── compose.yaml                 # 通用定义
├── compose.production.yaml      # 生产差异
├── .env.example                 # 变量名和非敏感示例值
├── .gitignore
├── README.md                    # 启停、端口、依赖、恢复入口
├── CHANGELOG.md                 # 变更历史
├── config/                      # 可公开、可审查的配置
├── secrets/                     # 本地秘密文件；禁止提交
├── scripts/                     # 检查、备份、恢复、部署脚本
├── backups/                     # 本地临时备份；禁止提交
└── docs/
    ├── ARCHITECTURE.md          # 服务、网络、数据流
    ├── OPERATIONS.md            # 巡检与故障处理
    └── RECOVERY.md              # 恢复顺序和验证方法
```

强制约束：

- MUST 通过 Compose 管理长期服务，禁止用临时 `docker run` 代替仓库中的正式定义。
- MUST 将生产差异放入 `compose.production.yaml`，不得直接复制出多套相互漂移的完整 Compose 文件。
- MUST 在 README 中写明项目名、负责人、服务、端口、卷、外部依赖、启动命令、停止命令和恢复入口。
- MUST 将 `.env`、`secrets/`、`backups/` 加入 `.gitignore`。
- SHOULD 给 Compose 项目设置稳定、唯一的项目名，避免不同目录生成重名网络和卷。

Docker 官方建议用额外的生产 Compose 文件覆盖开发定义，并在生产环境移除源码绑定、调整端口和日志、配置重启策略。[来源](https://docs.docker.com/compose/how-tos/production/)

## 5. 镜像治理

### 5.1 来源

- MUST 优先使用 Docker Official Image、Verified Publisher 或软件厂商官方镜像。
- MUST 记录镜像仓库、版本、用途和升级负责人。
- MUST 禁止来源不明的个人镜像进入生产。

### 5.2 版本

- 生产环境 MUST 禁止使用裸镜像名或 `latest`，例如 `emqx/emqx:latest`。
- MUST 至少固定到明确版本标签。
- 核心数据库、消息代理和无法快速回滚的组件 SHOULD 同时固定 digest。
- 升级 MUST 通过 Git 变更完成，不允许只在服务器上手动拉取新镜像。

标签可能被发布者重新指向其他镜像；固定 digest 才能保证同一声明得到同一镜像。[来源](https://docs.docker.com/build/building/best-practices/#pin-base-image-versions)

### 5.3 自建镜像

- MUST 使用 `.dockerignore` 排除密钥、备份、日志、Git 元数据和无关文件。
- SHOULD 使用多阶段构建，并让最终镜像只包含运行所需内容。
- SHOULD 使用非 root 用户运行主进程。
- SHOULD 删除无用包、构建工具和缓存。
- MUST 在 CI 或发布前实际构建并启动测试镜像。

Docker 官方建议选择可信且较小的基础镜像、使用多阶段构建、减少无关包，并在 CI 中构建测试。[来源](https://docs.docker.com/build/building/best-practices/)

## 6. 配置与秘密信息

### 6.1 配置

- 可公开配置 MUST 进入 Git。
- 环境差异 SHOULD 通过变量或生产覆盖文件表达。
- `.env.example` MUST 只列变量名、含义和安全示例，不得包含真实密码、Token、证书私钥或设备凭据。

### 6.2 秘密信息

- 密码、Token、证书私钥和 API Key MUST 禁止写入 Dockerfile、Compose 文件、镜像层和 Git。
- Linux Compose 环境 SHOULD 使用按服务授权的 `secrets` 文件挂载。
- 秘密文件 MUST 限制宿主机读取权限，并建立轮换方法。
- 日志、错误页面和诊断命令 MUST 避免打印秘密值。

Docker 官方指出，环境变量可能被其他进程读取或意外打印到日志；Compose secrets 会按服务授权，并挂载到 `/run/secrets/<name>`。[来源](https://docs.docker.com/compose/how-tos/use-secrets/)

## 7. 网络治理

- 每个项目 MUST 使用自己的用户定义网络，禁止依赖默认 `bridge` 网络。
- 容器之间 MUST 使用 Compose 服务名和容器端口通信，例如 `emqx:1883`；不得使用 `localhost` 访问另一个容器。
- 只有需要被宿主机或外部设备访问的端口 MAY 使用 `ports` 发布。
- 管理端口 SHOULD 默认绑定 `127.0.0.1`，再通过 SSH、VPN或受控反向代理访问。
- 数据库、Redis 等内部服务 MUST 默认不发布宿主机端口。
- SHOULD 按访问关系拆分前端网络和后端网络，避免所有容器互通。
- MUST 在文档中区分三类地址：容器内地址、Docker 宿主机地址、局域网地址。

用户定义 bridge 网络提供容器名 DNS 和更好的项目隔离；同一网络内的容器无需发布端口即可互通。[来源](https://docs.docker.com/engine/network/drivers/bridge/)

## 8. 数据与挂载治理

### 8.1 数据分类

| 数据类型 | 默认载体 | 是否备份 | 说明 |
|---|---|---:|---|
| 数据库、Node-RED 用户数据、EMQX 数据 | 命名卷 | 是 | 与容器生命周期分离 |
| 可审查配置 | Git + 只读 bind mount | 是 | 人工可读、可版本化 |
| 导入导出目录 | bind mount | 视业务而定 | 需要宿主机直接访问 |
| 缓存、临时文件 | tmpfs 或容器临时层 | 否 | 可重新生成 |
| 密钥 | secrets 文件 | 是，受控保存 | 不进入 Git |

### 8.2 强制规则

- 持久化业务数据 MUST 放在命名卷或明确批准的外部存储中，禁止只写容器可写层。
- MUST 禁止生产使用匿名卷。
- bind mount MUST 使用明确路径；能只读时必须设置 `read_only` / `ro`。
- 新增 bind mount 前 MUST 检查目标目录是否已有镜像内文件，防止挂载后被遮蔽。
- 使用非 root 容器访问 bind mount 前 MUST 验证宿主机 UID/GID 和权限映射。
- 远程 NFS/SMB 卷 MUST 单独验证断网、重启、存储延迟和恢复行为，不得只验证手动启动。
- `docker compose down -v`、`docker volume prune` 属于数据破坏命令，生产环境执行前 MUST 完成目标确认、备份确认和人工批准。

Docker 官方将 volumes 作为容器持久化数据的首选机制；bind mount 默认可写宿主机文件、依赖宿主机目录结构，并可能遮蔽镜像内原有内容。[Volumes](https://docs.docker.com/engine/storage/volumes/)｜[Bind mounts](https://docs.docker.com/engine/storage/bind-mounts/)

社区案例显示，bind mount 的写入权限取决于宿主机与容器内 UID/GID 的实际映射，简单地把容器改成 root 只是掩盖问题。[论坛样本](https://forums.docker.com/t/help-needed-how-to-mount-a-directory-as-a-non-root-user-in-a-container/141661)

## 9. 可用性与启动治理

- 长期服务 MUST 配置明确的重启策略；单机生产默认使用 `unless-stopped`。
- 一次性任务 SHOULD 使用 `on-failure:<次数>`，不得无限重启掩盖确定性错误。
- 有业务依赖的服务 MUST 提供 `healthcheck`，不能只检查进程存在。
- `depends_on` 需要等待依赖可用时，MUST 使用 `condition: service_healthy`。
- 应用自身 MUST 能处理依赖短暂不可用：重试、退避、断线重连，不得只依赖 Compose 启动顺序。
- 每次正式投产 MUST 做一次完整的宿主机重启或断电恢复测试。
- 重启验收 MUST 检查业务链路，而不只是检查容器状态为 `running`。

Compose 只会等待容器进入运行状态，不会自动等待服务真正可用；要等待就绪状态，需要 healthcheck 与 `service_healthy`。[来源](https://docs.docker.com/compose/how-tos/startup-order/)

重要边界：Docker Engine 在主机重启时执行容器级 restart policy，不会重新执行一次完整的 `docker compose up` 流程。因此，生产系统不能把 `depends_on` 当成永久编排保证。Docker 社区长期存在“重启后依赖顺序失效”的案例；更稳妥的根治方式是让应用能重试依赖。[论坛样本](https://forums.docker.com/t/how-to-handle-server-reboot-when-using-docker-compose/26374)

若必须由 systemd 管理整套 Compose 启动，MUST 作为例外方案评审，并避免与容器 restart policy 重复管理；Docker 官方明确警告不要混用两套重启管理机制。[来源](https://docs.docker.com/engine/containers/start-containers-automatically/)

## 10. 资源治理

- 所有长期运行服务 MUST 设置内存上限。
- 高 CPU 服务 SHOULD 设置 CPU 上限或权重。
- 上限 MUST 基于测试和运行数据设定，不得机械复制模板值。
- 宿主机 MUST 保留 Docker daemon、SSH、监控和突发负载所需余量。
- MUST 监控容器重启次数、OOMKilled、CPU、内存、磁盘和卷容量。
- 因 OOM 退出时 MUST 先查负载和泄漏，不得只靠不断提高内存上限解决。

Docker 容器默认没有 CPU 和内存限制，单个容器可耗尽宿主机资源；官方建议在投产前测试需求并设置内存限制。[来源](https://docs.docker.com/engine/containers/resource_constraints/)

## 11. 日志治理

- 所有容器 MUST 配置日志轮转或使用自带轮转的 `local` 日志驱动。
- 使用 `json-file` 时 MUST 设置 `max-size` 和 `max-file`。
- 应用日志 SHOULD 输出到 stdout/stderr，由 Docker 统一收集。
- 业务审计日志与普通运行日志 MUST 分开定义保留周期。
- MUST 禁止在日志中输出密码、Token、完整连接串和个人敏感数据。
- 磁盘巡检 MUST 同时覆盖镜像、容器层、构建缓存、日志和卷。

Docker 默认 `json-file` 驱动不自动轮转，可能耗尽磁盘；官方在非 Kubernetes 场景推荐使用自动轮转的 `local` 驱动。[来源](https://docs.docker.com/engine/logging/configure/)

## 12. 安全治理

- Docker daemon 控制权等同于宿主机高权限，只有受信任人员 MAY 获得访问权。
- MUST 禁止无 TLS 的远程 Docker TCP API。
- 远程管理 SHOULD 使用 SSH Docker context；必须走 TCP 时使用双向 TLS。
- 未经专项批准，MUST 禁止：
  - `privileged: true`；
  - 挂载 `/var/run/docker.sock`；
  - 挂载宿主机根目录或敏感系统目录；
  - `network_mode: host`；
  - 关闭默认 seccomp；
  - 添加不必要的 Linux capabilities。
- 容器 SHOULD 使用非 root 用户。
- 经兼容性验证后，服务 SHOULD 使用只读根文件系统、`cap_drop: [ALL]` 和 `no-new-privileges`，再按实际需要最小化放开权限。
- MUST 定期更新宿主机安全补丁、Docker Engine、Compose 插件和镜像；更新必须经过本规范的发布流程。

Docker 官方说明 Docker daemon 通常具有 root 权限，只有可信用户才能控制；远程访问应使用 SSH 或 TLS。[Daemon security](https://docs.docker.com/engine/security/)｜[Protect daemon socket](https://docs.docker.com/engine/security/protect-access/)

默认 seccomp 配置是兼顾保护与兼容性的允许列表，官方不建议关闭或随意替换。[来源](https://docs.docker.com/engine/security/seccomp/)

## 13. 变更与发布流程

### 13.1 变更前

1. 在 Git 创建变更记录；
2. 写明原因、影响服务、数据风险和回滚方法；
3. 固定待发布镜像版本；
4. 对有状态服务完成一致性备份；
5. 执行配置校验；
6. 确认可用磁盘与资源余量。

最低校验命令：

```bash
docker compose -f compose.yaml -f compose.production.yaml config -q
docker compose -f compose.yaml -f compose.production.yaml config > rendered-compose.yaml
docker compose -f compose.yaml -f compose.production.yaml pull
```

`docker compose config` 会合并 Compose 文件、解析变量并生成 Docker Engine 实际接收的模型；`-q` 可只做配置验证。[来源](https://docs.docker.com/reference/cli/docker/compose/config/)

### 13.2 发布

```bash
docker compose -f compose.yaml -f compose.production.yaml up -d
docker compose -f compose.yaml -f compose.production.yaml ps
docker compose -f compose.yaml -f compose.production.yaml logs --tail=200
```

发布后 MUST 验证：

1. 所有长期服务状态符合预期；
2. healthcheck 通过；
3. 容器间依赖可访问；
4. 外部端口只暴露预期服务；
5. Node-RED → MQTT → 数据库 → 看板等真实业务链路至少完成一次冒烟测试；
6. 新增卷和目录中确实产生了预期数据；
7. 观察期内无重启循环、OOM 或磁盘异常增长。

### 13.3 回滚

- 每次变更 MUST 有上一版本 Git commit 或 tag。
- 镜像 MUST 能恢复到上一版本标签或 digest。
- 有状态服务的数据库结构变更 MUST 有向后兼容方案或明确的数据恢复方案。
- 回滚后 MUST 重复执行发布后的业务验证。

## 14. 备份与恢复

- MUST 为每个持久化卷登记：服务、用途、重要级别、备份方法、频率、保留期和恢复顺序。
- 数据库 SHOULD 优先采用数据库原生一致性备份；文件型数据再使用卷级备份。
- 备份文件 MUST 离开 Docker 宿主机保存；只在同一块磁盘复制不算灾备。
- MUST 定期实际恢复到隔离环境。没有恢复验证的备份只能视为“可能可用”。
- 重大升级前 MUST 做即时备份并记录校验结果。
- 恢复顺序 SHOULD 为：基础存储 → 数据库/消息代理 → 应用 → 看板/入口 → 业务链路验证。

Docker 官方说明 volume 独立于容器生命周期，并提供卷的备份与恢复方法。[来源](https://docs.docker.com/engine/storage/volumes/#back-up-restore-or-migrate-data-volumes)

社区案例显示，远程 NFS 卷可能在主机重启时因网络尚未完全可用而挂载失败，手动启动正常不能证明自动恢复可靠。[论坛样本](https://forums.docker.com/t/issue-with-service-using-nfs-volume-not-starting-during-boot/143759)

## 15. 禁止直接执行的生产命令

以下命令不是绝对不能用，而是执行前 MUST 明确目标、影响范围、备份状态和恢复办法：

```text
docker compose down -v
docker volume prune
docker system prune
docker system prune -a
docker rm -f
docker rmi -f
docker volume rm
```

不得对未解析的变量、通配符或不明确的 Compose 项目执行清理操作。

## 16. Compose 最小基线示例

此示例只表达治理结构，资源值、用户、健康检查和权限必须按具体镜像验证后调整：

```yaml
name: example-stack

x-service-defaults: &service-defaults
  restart: unless-stopped
  init: true
  logging:
    driver: local
    options:
      max-size: "10m"
      max-file: "3"

services:
  app:
    <<: *service-defaults
    image: registry.example.com/app:1.4.2@sha256:REPLACE_WITH_REAL_DIGEST
    mem_limit: 512m
    cpus: 1.0
    read_only: true
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    tmpfs:
      - /tmp
    volumes:
      - app_data:/data
      - ./config/app.yaml:/etc/app/app.yaml:ro
    secrets:
      - app_password
    networks:
      - backend
    depends_on:
      broker:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "/app/healthcheck"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 20s

  broker:
    <<: *service-defaults
    image: vendor/broker:5.0.0@sha256:REPLACE_WITH_REAL_DIGEST
    mem_limit: 1g
    cpus: 1.5
    ports:
      - "1883:1883"
    volumes:
      - broker_data:/var/lib/broker
    networks:
      - backend
    healthcheck:
      test: ["CMD", "/usr/local/bin/broker-healthcheck"]
      interval: 30s
      timeout: 5s
      retries: 5
      start_period: 30s

networks:
  backend:
    internal: false

volumes:
  app_data:
    name: example-stack-app-data
  broker_data:
    name: example-stack-broker-data

secrets:
  app_password:
    file: ./secrets/app_password.txt
```

## 17. 上线门禁清单

### 仓库

- [ ] Compose、生产覆盖文件、README、变更记录已进入 Git
- [ ] `.env.example` 不含真实秘密
- [ ] `.gitignore` 已排除 `.env`、`secrets/`、`backups/`
- [ ] 镜像没有使用 `latest`
- [ ] `docker compose config -q` 通过

### 网络与权限

- [ ] 容器间使用服务名，不使用 `localhost`
- [ ] 仅发布必要端口
- [ ] 管理端口未直接暴露到不可信网络
- [ ] 未使用 `privileged`、Docker socket 或不必要 capability
- [ ] bind mount 权限和只读属性已验证

### 稳定性

- [ ] 长期服务配置 restart policy
- [ ] 关键服务有真实 healthcheck
- [ ] 应用能重试暂时不可用的依赖
- [ ] CPU、内存和日志上限已设置
- [ ] 宿主机重启后完整业务链路验证通过

### 数据

- [ ] 每个持久化数据目录都有明确卷
- [ ] 已登记备份频率、保留期和负责人
- [ ] 本次发布前备份已完成
- [ ] 最近一次恢复演练通过
- [ ] 回滚版本和步骤明确

## 18. 例外机制

任何 MUST 规则如需偏离，必须在 Git 中记录：

```text
例外编号：
偏离规则：
业务原因：
风险：
临时控制措施：
负责人：
到期日期：
关闭条件：
```

例外不能无限期存在；到期后必须关闭、续期或转为正式规则变更。

## 19. 规范维护方式

- 本文件 MUST 进入治理仓库，由 `AGENTS.md` 指向它。
- Docker 相关线程开始任务前 MUST 读取本文件及目标项目 README。
- 新规则必须来自以下至少一种证据：官方定义、已复现故障、真实项目事故、稳定自动化检查。
- 论坛帖子只能提出候选规则；必须经官方文档或本地复现确认后，才能升级为 MUST。
- 每次修订 MUST 更新版本、日期和 CHANGELOG。
- SHOULD 每季度检查官方文档变化，并用最近真实故障更新论坛样本。

推荐在根目录 `AGENTS.md` 中加入：

```markdown
## Docker 任务

凡涉及 Dockerfile、Compose、镜像、容器、网络、卷、部署、升级、备份或恢复的任务，开始前必须读取 `docs/docker/DOCKER_GOVERNANCE.md`。生产变更必须通过该文件的上线门禁清单；任何偏离必须登记例外。
```

---

## 蒸馏结论

Docker 治理的核心不是“会启动容器”，而是把每套服务变成一个可重复生产的产品：

**配置能重建、数据能恢复、故障能自愈、权限有边界、变更能回滚、规则能被下一条线程继续执行。**
