# GitHub 设置清单

文档只能告诉成员应该怎么做，GitHub 设置负责阻止明显违规操作。

## 一、本规范仓库

首次初始化 PR 合并后再启用保护，避免空仓库无法完成引导。

- [ ] `Settings > General`：将仓库标记为 Template repository。
- [ ] 只保留 Squash merging，关闭 Merge commit 和 Rebase merge。
- [ ] 开启 Automatically delete head branches。
- [ ] 确认 Issues 已开启，Wiki 按需关闭。
- [ ] 为协作者分配最小够用权限。

## 二、main 规则集

在套餐支持私有仓库 Rulesets 时，创建针对 `main` 的规则集；否则使用 Branch protection 实现相同目标。

建议初始规则：

- [ ] 禁止删除和强制推送。
- [ ] 必须通过 Pull Request 才能合并。
- [ ] 有独立审查者时，至少一名非作者批准。
- [ ] 要求解决全部 PR 对话。
- [ ] 有代码负责人且具备独立审查者时，要求 CODEOWNERS 审查其负责范围。
- [ ] CI 建立后，将构建、测试和静态检查设为必需检查。
- [ ] 管理员绕过只用于事故处置，并保留事后记录。

单人维护且没有第二位合格审查者时，可将批准数设为 `0` 并关闭 CODEOWNERS 审查要求；仍必须保留 PR、已解决对话、禁止强推和删除、以及 Squash Merge。高风险动作不能因该例外而免除单独授权和记录。

## 三、项目仓库的补充设置

- [ ] 默认分支统一为 `main`。
- [ ] 合并方式和分支删除策略保持一致。
- [ ] 开启 Dependabot alerts 和适用的依赖更新。
- [ ] 在套餐支持时开启 Secret scanning 与 Push protection。
- [ ] 对生产环境使用 Environments、必需审查者和受控 Secrets。
- [ ] 定期复核协作者、Deploy keys、Apps 和 Actions 权限。

具体检查名称必须等项目 CI 稳定运行后再设为必需，避免引用不存在的检查导致所有 PR 无法合并。

