# ci-workflows

Vanzo-Media 共用的 GitHub Actions 可复用工作流。

目前提供一项:**AI 代码审查(PR Agent)**。

## 接入方法

### 1. 添加 secret

在目标仓库 `Settings → Secrets and variables → Actions` 添加:

| Name | Value |
| --- | --- |
| `DEEPSEEK_KEY` | DeepSeek 开放平台的 API key |

组织级 secret 在当前计划下对私有仓库无效,因此每个仓库都要各自添加一份。

### 2. 添加调用文件

在目标仓库新建 `.github/workflows/pr-agent.yml`:

```yaml
name: PR Agent

on:
  pull_request:
    types: [opened, reopened, ready_for_review, synchronize]
  issue_comment:
    types: [created]

concurrency:
  group: pr-agent-${{ github.event.pull_request.number || github.event.issue.number }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

jobs:
  review:
    uses: Vanzo-Media/ci-workflows/.github/workflows/pr-agent.yml@main
    permissions:
      contents: read
      issues: write
      pull-requests: write
      checks: write
    secrets:
      DEEPSEEK_KEY: ${{ secrets.DEEPSEEK_KEY }}
```

`permissions` 必须写在调用方:被调用的工作流在调用方授予的范围内运行,无法自行提权。本仓库的可复用工作流不声明 `permissions`,完全由调用方决定。

`cancel-in-progress` 只对 `pull_request` 事件为真:concurrency 在工作流级生效,早于 job 的 `if` 求值,若对全部事件为真,PR 上一条普通评论就会新建 run 并掐掉正在进行的评审,而该 run 自身随后又被跳过。

## 行为

| 时机 | 动作 |
| --- | --- |
| PR 打开 / 重开 / 从草稿转为 ready | 执行 `/review` 和 `/improve` |
| 向 PR 分支 push 新 commit | 重新执行 `/review` 和 `/improve` |
| 在 PR 中评论 `/review` `/improve` `/describe` `/ask <问题>` | 执行对应命令 |

审查结果以两种形式出现在 PR 上:

- 一条总览评论(问题清单、安全检查、评审工作量估计)
- 若干条锚定在代码行上的建议,带 GitHub 原生的 **Commit suggestion** 按钮,可一键提交

## 自定义

### 调整模型

```yaml
    with:
      model: "deepseek/deepseek-v4-pro"
      fallback_model: "deepseek/deepseek-v4-flash"
```

换其他厂商时模型名需带 LiteLLM 的 provider 前缀,并相应替换 secret。

### 关闭 push 触发

每次 push 都会消耗一次模型调用。只想在 PR 打开时审一次:

```yaml
    with:
      handle_push_trigger: "false"
```

同时把 `on.pull_request.types` 里的 `synchronize` 去掉。**两处必须成对修改**:只留 `synchronize` 会产生空跑,并因 `cancel-in-progress` 掐断正在进行的评审;只关 `handle_push_trigger` 则 push 事件根本不会进入工作流。

### 调整审查标准

在目标仓库根目录放 `.pr_agent.toml`:

```toml
[pr_reviewer]
extra_instructions = """
重点检查:...
以下不要报告,CI 已覆盖:代码格式、ESLint 能发现的问题、类型错误。
"""
```

**该文件由 PR-Agent 从仓库默认分支读取,不读 PR 分支。** 对它的修改要合并进 `main` 之后才会对后续 PR 生效——这是刻意的设计,避免有人通过 PR 改掉审查自己的规则。

## 注意事项

- **来自 fork 的 PR 不会被审查。** GitHub 不向 fork PR 下发 secrets,这是安全设计。内部协作(同仓库分支)不受影响。
- **审查结果不阻止合并。** job 成功与否与审查是否发现问题无关,它是建议性的。
- **采纳 suggestion 前先看行范围。** GitHub 的 Commit suggestion 对锚定行范围做字面替换,若建议内容含省略号等占位符,一次点击就会删掉整段配置。已在审查规则中约束模型不得如此输出,但跨十几行的建议仍建议展开核对。
- **第三方 action 锁定在 commit SHA。** 它在调用方仓库上下文中执行并持有 write token 与 API key,跟随 `@main` 或 tag 意味着上游任何提交都会未经审查直接运行,而 tag 可被 force-push 移动。升级由本仓库的 Dependabot 提 PR。
