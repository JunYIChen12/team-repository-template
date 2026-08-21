# 新人接入指南

目标：让新成员在 15～30 分钟内理解协作入口，并完成一次低风险 PR。

## 一、确认访问权限

- 能打开仓库并读取代码和 Issue。
- 能创建分支并推送到远程仓库。
- 知道直属项目负责人和自己负责的模块。
- 已启用 GitHub 双重身份验证，凭据没有通过聊天或文件明文传递。

## 二、完成本地准备

```bash
git clone <repository-url>
cd <repository-name>
git status
```

项目仓库还应按其 README 完成依赖安装、启动和测试。无法在本地运行时，不要直接开始功能开发，应先在 Issue 中记录阻塞。

## 三、理解日常流程

```text
Issue/任务
  → 创建短期分支
  → 修改和自测
  → 推送并创建PR
  → 自动检查与人工审查
  → Squash合并
  → 删除短期分支
```

具体要求见[贡献指南](../CONTRIBUTING.md)。

## 四、完成第一个 PR

第一个任务应选择错别字、文档补充或小型测试等低风险工作：

```bash
git switch main
git pull --ff-only
git switch -c docs/<issue-number>-first-change
# 完成修改
git add -- <明确的文件路径>
git commit -m "docs: 完成首次协作修改"
git push -u origin HEAD
```

随后创建 PR，完整填写模板，并请项目成员审查。不要为了演练直接向 `main` 推送。

## 五、接入完成标准

- [ ] 能找到本项目采用的规范版本。
- [ ] 能独立创建符合命名要求的分支。
- [ ] 能提交并推送一个小型变更。
- [ ] 能创建完整填写的 PR。
- [ ] 能理解自动检查结果和审查意见。
- [ ] 知道安全问题不能通过公开 Issue 报告。

