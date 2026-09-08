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
| OpenAI 兼容网关(Portkey) | `PORTKEY_API_KEY` | Portkey 的网关密钥。各厂商自己的 key 存在 Portkey 侧,不进本仓库 |

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

### 走 OpenAI 兼容网关(Portkey)

通过网关调用时,厂商自己的 key 存在网关侧,各仓库只需要一把网关密钥:

```yaml
    with:
      model: "openai/@dashscope/qwen3.8-max"
      fallback_model: "openai/@deepseek/deepseek-chat"
      openai_api_base: "https://api.portkey.ai/v1"
      custom_model_max_tokens: "200000"
    secrets:
      PORTKEY_API_KEY: ${{ secrets.PORTKEY_API_KEY }}
```

模型串是 `openai/` 前缀 + 网关自己的模型 ID。Portkey 的格式是 `@<provider-slug>/<model>`,slug 在 Portkey 的 Model Catalog 里定义。LiteLLM 只按第一个 `/` 剥前缀,后面的 `@slug/model` 原样透传给网关。

**三条硬性要求,漏了会被前置校验直接拦下:**

- **`custom_model_max_tokens` 必填,且为正数。** PR-Agent 的 `MAX_TOKENS` 表按完整模型串匹配,而网关模型串(`openai/@slug/name`)必然不在表里,`get_max_tokens()` 届时直接抛异常。这个值纯手工维护,**写错不会报错,只会让评审静默按错的预算跑**。最终预算是 `min(max_model_tokens, 本值)`,所以填到不小于 `max_model_tokens` 即可,不必精确等于模型真实窗口。
- **`openai_api_base` 与 `PORTKEY_API_KEY` 必须成对。** 只给 key 会打到 `api.openai.com`;只给 base URL 则没有凭证。
- **`model` 与 `fallback_model` 必须都是 `openai/` 前缀。** `api_base` 一旦设置就对所有 provider 全局生效(PR-Agent 把它塞进每一次 completion 的 kwargs),主模型走网关、降级留直连的组合会让降级请求带着厂商 key 打到网关地址上,且只在降级那一刻才炸。

跨厂商降级在网关下仍然可行,写成同一网关的不同 slug 即可,上例就是 Qwen 降级到 DeepSeek。

**与直连的取舍:** 走网关会失去 `MAX_TOKENS` 自动识别。若某模型已在 PR-Agent 表中(例如 `dashscope/qwen3.8-max` 登记为 1M),直连只需一行 `model:` 加一个厂商 secret,不必维护 `custom_model_max_tokens`。网关的价值在于凭证集中管理与统一的调用看板,不在于换模型本身。

### 调整 token 预算

单次评审能读进多少 diff,由 `max_model_tokens` 决定(默认 **200000**):

```yaml
    with:
      max_model_tokens: "400000"
```

**这是硬上限,与模型能力无关。** PR-Agent 的 `get_max_tokens()` 最后一步是 `min(config.max_model_tokens, 模型自身上限)`,所以模型再大也会被这个值掐住。上游默认值是 `32000`,本工作流把它提到 200000。

预算不够时不会报错,而是**静默降级成部分审查**:PR-Agent 按文件优先级排序,塞到装不下为止,剩下的写进「未纳入审查」清单,job 照样成功。判断依据是机器人评论里的 `"complete": false, "kind": "partial"`,以及 `because of the token budget` 字样。看到这个就该往上调,或按下面的办法省预算。

几点须知:

- **换更大的模型不解决这个问题。** 瓶颈在这个配置,不在模型。参考:`deepseek/deepseek-v4-flash` 和 `gpt-5.5` 在 PR-Agent 的 `MAX_TOKENS` 表里登记为 100 万,`gemini-2.5-pro` 约 105 万,而 **Claude 全系只有 20 万**——切到 Bedrock 上的 Claude 反而会缩小可用上下文。
- **不建议直接拉到模型上限。** 上游注释指出输入过长会让模型表现下降;而且每次 push 跑 `/review` + `/improve` 两轮,输入 token 直接乘二,成本线性增长。
- **先省预算,再加预算。** 排除生成物、锁文件、测试数据往往比调高上限更划算,既省钱又提高信噪比。在调用方仓库根目录的 `.pr_agent.toml` 里配:

  ```toml
  [ignore]
  glob = ['**/_generated/**', 'pnpm-lock.yaml', 'eval/**']
  ```

  和 `extra_instructions` 一样,**该文件从默认分支读**,改完要先合进 `main` 才对后续 PR 生效。

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
- **评审可能只覆盖部分文件。** 超出 token 预算的文件会被静默跳过,job 仍然成功。大 PR 收到评审后,先看评论里是不是 `"kind": "partial"`——是的话别把"只报了一条"当成"其余干净",没被读到的文件根本没审。见 [调整 token 预算](#调整-token-预算)。
- **采纳 suggestion 前先看行范围。** GitHub 的 Commit suggestion 对锚定行范围做字面替换,若建议内容含省略号等占位符,一次点击就会删掉整段配置。已在审查规则中约束模型不得如此输出,但跨十几行的建议仍建议展开核对。
- **第三方 action 锁定在 commit SHA。** 它在调用方仓库上下文中执行并持有 write token 与 API key,跟随 `@main` 或 tag 意味着上游任何提交都会未经审查直接运行,而 tag 可被 force-push 移动。升级由本仓库的 Dependabot 提 PR。
