# AGENTS.md

本文件是仓库内所有 Codex 线程和自动化代理的共同入口。任何代理开始修改前，必须先读取本文件以及与任务相关的规范；不得只依赖聊天上下文或旧线程记忆。

## 共同规则

1. 本仓库是团队协作规范的唯一权威来源；先阅读 [README.md](README.md)。
2. 所有变更遵守 [仓库管理与协作总规范](docs/REPOSITORY_GOVERNANCE.md) 和 [贡献指南](CONTRIBUTING.md)。
3. 禁止直接修改 `main`；每项任务使用独立短期分支并通过 Pull Request 进入 `main`。
4. 修改前必须读取现有文件，保留无关内容；不得用整份重写覆盖已有规则。
5. PR 必须说明变更目的、内容、验证证据、风险和回滚方式。
6. 未经明确授权，不得合并 PR、删除分支、删除数据、暴露秘密信息或执行生产变更。

## 任务路由

- 开发、运行、测试或验收任务：读取 [目标驱动的开发、运行与验收规范](docs/DEVELOPMENT_READINESS.md)。
- IoT 项目任务：继续读取 [IoT 项目专用规范](docs/iot/README.md) 及其中指向的场景规范。
- Dockerfile、Compose、镜像、容器、网络、卷、部署、升级、备份或恢复任务：必须读取 [Docker 治理规范](docs/docker/DOCKER_GOVERNANCE.md)。
- Windows Docker Desktop、WSL2、VHDX、磁盘阈值或空间清理任务：在通用 Docker 规范之外，还必须读取并遵守现有的 [Docker / WSL2 空间治理规范](docs/DOCKER_GOVERNANCE.md)。
- 两份 Docker 规范同时适用时，取更严格的限制；若规则冲突，停止执行并在 PR 中提出，不得自行覆盖旧规则。

## Docker 生产门禁

涉及生产 Docker 环境时，代理必须：

1. 先确认目标环境、Compose 项目、镜像版本、端口、卷和秘密信息边界；
2. 执行适用的配置校验、备份确认、健康检查和真实业务链路验证；
3. 为变更提供明确回滚路径；
4. 对危险命令和任何 MUST 规则例外取得人工批准；
5. 将证据写入 PR，不得仅以容器显示 `running` 作为验收结论。
