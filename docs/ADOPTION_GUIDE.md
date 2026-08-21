# 项目采用指南

本指南解决两个问题：新项目如何采用规范，以及中心规范更新后项目如何跟进。

## 一、中心引用和项目内文件

| 内容 | 维护方式 | 中心更新后的效果 |
| --- | --- | --- |
| 总规范、新人指南、安全原则 | 项目 README 链接到本仓库 | 打开链接即可看到最新版 |
| PR/Issue 模板、CODEOWNERS | 文件存在于项目仓库 | 不会自动更新，需要同步 PR |
| CI 工作流 | 项目内工作流或后续可复用工作流 | 依据采用方式决定 |
| Rulesets 和权限 | GitHub 仓库或组织设置 | 由管理员集中配置 |

模板仓库只在创建项目时复制一次，之后不会自动把新版本覆盖到旧项目。

## 二、项目 README 引用模板

把以下内容加入项目 README：

```markdown
## 团队协作

本项目遵循 [团队仓库协作规范 v1](https://github.com/JunYIChen12/team-repository-template/blob/main/docs/REPOSITORY_GOVERNANCE.md)。

新成员请先完成[新人接入指南](https://github.com/JunYIChen12/team-repository-template/blob/main/docs/ONBOARDING.md)，日常提交遵循本项目的 `CONTRIBUTING.md`。
```

因为本仓库当前是私有仓库，项目成员必须同时具有本仓库的读取权限才能打开链接。

## 三、新项目接入清单

- [ ] 在 README 中声明规范版本并添加中心链接。
- [ ] 复制并按项目调整 `CONTRIBUTING.md`。
- [ ] 复制 `.github/` 下的 PR、Issue 和 CODEOWNERS 模板。
- [ ] 把 CODEOWNERS 中的示例负责人改成项目真实负责人。
- [ ] 根据技术栈增加构建、测试和安全检查。
- [ ] 按 [GitHub 设置清单](GITHUB_SETTINGS.md)配置默认分支和合并规则。
- [ ] 邀请一名新成员按新人指南完成演练并记录问题。

## 四、规范升级

中心规范更新时：

1. 在 `CHANGELOG.md` 说明变化和受影响对象。
2. 纯澄清和错字修正直接生效。
3. 新增流程时通知项目负责人，由项目创建升级 PR。
4. 不兼容变化发布新的主版本，项目明确决定是否和何时升级。
5. 项目内模板通过 PR 同步，禁止静默覆盖。

