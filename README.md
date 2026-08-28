# ci-workflows

Vanzo-Media 共用的 GitHub Actions 可复用工作流。

目前提供一项:**AI 代码审查(PR Agent)**。

## 接入方法

### 1. 添加 secret

在目标仓库 `Settings → Secrets and variables → Actions` 添加**所用 provider 的**凭证。默认使用 DeepSeek,只需第一行:

| Provider | Name | Value |
| --- | --- | --- |
| DeepSeek(默认) | `DEEPSEEK_KEY` | DeepSeek 开放平台的 API key |
| AWS Bedrock | `AWS_ACCESS_KEY_ID` | 具备 `bedrock:InvokeModel` 权限的 IAM 凭证 |
| AWS Bedrock | `AWS_SECRET_ACCESS_KEY` | 同上 |

只需配置实际用到的那一套,其余留空即可——见 [切换模型 / 更换厂商](#切换模型--更换厂商)。区域不是敏感信息,通过 `aws_region` 参数传,不用建 secret。

组织级 secret 在当前计划下对私有仓库无效,因此每个仓库都要各自添加一份。

### 2. 添加调用文件

在目标仓库新建 `.github/workflows/pr-agent.yml`:

```yaml
name: PR Agent

on:
  pull_request:
    types: [opened, reopened, ready_for_review, synchronize]

concurrency:
  group: pr-agent-${{ github.event.pull_request.number }}
  cancel-in-progress: true

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

工作流只在**代码变动**时触发,不监听评论。评论命令通道(`/review` 等)曾先后引入两次 concurrency 竞态且无实际使用,已移除;评审失败需要重试时,close 再 reopen 该 PR 即可。

## 行为

| 时机 | 动作 |
| --- | --- |
| PR 打开 / 重开 / 从草稿转为 ready | 评审并给出改进建议 |
| 草稿(draft)PR | **不评审**,转为 ready 时才评审 |
| 向 PR 分支 push 新 commit | 重新评审(连续 push 时取消过时的一轮,只跑最新)|

审查结果以两种形式出现在 PR 上:

- 一条总览评论(问题清单、安全检查、评审工作量估计)
- 若干条锚定在代码行上的建议,带 GitHub 原生的 **Commit suggestion** 按钮,可一键提交

## 自定义

### 切换模型 / 更换厂商

模型名是 LiteLLM 标识,**必须带 provider 前缀**——切换厂商就是换这个前缀,工作流本身不用改:

```yaml
    with:
      model: "deepseek/deepseek-v4-pro"
      fallback_model: "deepseek/deepseek-v4-flash"
```

换成 AWS Bedrock 上的 Claude,同时把凭证一并传进来:

```yaml
    with:
      model: "bedrock/<inference-profile-id>"
      fallback_model: "deepseek/deepseek-v4-pro"
      aws_region: "us-east-1"
    secrets:
      DEEPSEEK_KEY: ${{ secrets.DEEPSEEK_KEY }}
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

`<inference-profile-id>` 从 Bedrock 控制台 → Inference profiles 取,形如 `us.anthropic.claude-...`,随区域和已开通的模型而变。跨区域推理配置(inference profile)必须用带区域前缀的 ID,直接写基础模型 ID 会被 Bedrock 拒绝。

几点须知:

- **未使用的 provider 传空 secret 是安全的**,不必删掉 `secrets:` 中的条目。PR-Agent 的各 provider 分支都是真值判断,空字符串会被整段跳过,不会互相干扰。
- **主模型与降级模型可以跨厂商**,上例即"Bedrock 限流或故障时落回 DeepSeek"。但这要求**两套凭证都在场**——工作流会对 `model` 和 `fallback_model` 分别校验,漏配直接快速失败,不会拖到降级那一刻才炸。
- **Bedrock 的 AK 与 SK 必须成对配置。** 只配其中一个会被前置校验拦下;若绕过校验,PR-Agent 会抛 `AWS credentials are incomplete`。
- **成本会变。** Bedrock 上的 Claude 单价高于 DeepSeek,而每次 push 都会跑一轮 `/review` + `/improve`。换厂商前先看 [关闭 push 触发](#关闭-push-触发)。

### 关闭 push 触发

每次 push 都会消耗一次模型调用。只想在 PR 打开时审一次:

```yaml
    with:
      handle_push_trigger: "false"
```

同时把 `on.pull_request.types` 里的 `synchronize` 去掉。关闭后评审只在 PR 打开时跑一次,后续 push 的代码不会被审查。**两处必须成对修改**:只留 `synchronize` 会产生空跑,并因 `cancel-in-progress` 掐断正在进行的评审;只关 `handle_push_trigger` 则 push 事件根本不会进入工作流。

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
