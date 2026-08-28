# jcode 补丁说明（v0.81.2-dev 部署记录，给未来的自己 / jcode 会话）

## 这次补丁是什么

分支：`fix/swarm-stop-terminates-detached-tasks`（基于上游 master 的修复分支）
发布：`v0.81.2-dev`（Release 已公开，资产 `jcode-0.81.2-dev-linux-x86_64.tar.gz`）
部署：`~/.jcode/builds/versions/0.81.2/`（daemon 已 reload，PID 1012 re-exec 到新二进制）

**一句话**：让 `[providers.opencode-go]` 的 `model_catalog = false` + 5 模型 allowlist 在
**活动与活动外两条 picker 路径**都生效，32 个 gateway 模型（含坏掉的
`muse-spark-1.2-contributor`）不再泄漏进 `/model`；muse 仍可通过 openai-api provider
（Responses 端点）直接使用。

## 提交清单（按顺序）

| 提交 | 内容 |
|---|---|
| `43daecb` | `new_openai_compatible_profile_runtime`（活动 provider 路径，生产经由 `OpenRouterRuntimeSpec::CompatibleProfile` → startup 工厂）加 named_override 分支：读 `[providers.<id>]`，OpenAiCompatible 时用 `cfg.models` + `cfg.model_catalog`。等价于旧分支 `fix/provider-model-allowlist` 的 `57f4186`（该分支基于旧 master，行号不同） |
| `4e4fa50` | `direct_openai_compatible_profile_routes`（非活动 profile 路径）同样遵守 `model_catalog=false` + 非空 allowlist。**旧 fork 补丁没有这块**：切换活动 provider 后 opencode-go 会经此路径重新泄漏 32 个模型 |
| `558beb5` | 集成测试 `picker_allowlist_gated_by_named_override_for_builtin_profile`：临时 JCODE_HOME + 生产构造器，断言 picker 只出 allowlist(+当前默认模型) |
| `6d75993` | `dev-build.yml`：fork-local CI（无 private secrets），tag `v*-dev` 触发，仅 Linux x86_64，`JCODE_BUILD_SEMVER` 硬编码 `v0.81.2-dev` |

## 为什么不能用上游 release.yml + 重要陷阱

- fork `iasds/jcode` **没有任何 GitHub secrets**（DEPLOY_KEY 为空），上游 release.yml 每次
  都在 "Configure SSH for cargo git dependencies" 步骤失败。
- **`gh` CLI 默认解析到上游 `1jehuang/jcode`**（`gh repo view` 会骗你），操作 fork 一律
  显式 `-R iasds/jcode` 或 `gh api repos/iasds/jcode/...`。
- **`provider-doctor` 的 "Picker shows live models" 是假象**：doctor 用
  `OpenRouterProvider::new()`（env 自动探测路径）构造 runtime，且 picker 检查是合成路由，
  永远显示 32 个 live catalog 模型。**验证 picker 修复只能用集成测试
  `picker_allowlist_gated_by_named_override_for_builtin_profile`**（或真 TUI 里看 /model）。
- 本地构建：`scripts/dev_cargo.sh` 之前要用 `~/.cargo/bin` 的 rustc 1.97.1
  （`export PATH="$HOME/.cargo/bin:$PATH"`），系统 rustc 1.85 太老编不过。

## 部署产物布局（与 install_release.sh 约定一致）

```
~/.jcode/builds/versions/0.81.2/{jcode(launcher), jcode-linux-x86_64.bin}
~/.jcode/builds/{stable,current,shared-server}/jcode -> versions/0.81.2/jcode
~/.jcode/builds/{stable-version,current-version} = "0.81.2"
~/.local/bin/jcode -> builds/current/jcode
```

launcher `jcode`（505B sh）exec 同目录 `jcode-linux-x86_64.bin`；dev-build 的 tar 里只有
裸二进制，部署时要重命名并配 launcher。

## 给未来 jcode 会话的入口

- 修复与测试：`git log --oneline 43daecb..558beb5`（本分支）
- 构建：tag `v0.81.2-dev` 推送触发 dev-build.yml；或 `gh workflow run dev-build.yml -R iasds/jcode --ref v0.81.2-dev`
- 下载：`gh run download <run> -R iasds/jcode -n jcode-v0.81.2-dev-linux-x86_64`
- 发布：`gh release edit v0.81.2-dev -R iasds/jcode --draft=false`
- 部署：替换 `versions/<ver>` 目录 + 重指 3 条软链 + `jcode server reload`（会瞬时打断当前会话的重连）
- muse 可用性验证：`jcode run --no-update -m 'openai-api:muse-spark-1.2-contributor' 'Reply with exactly: MUSE_OK'`

## 上游跟进

上游发新版（如 v0.82.0）时：rebase 本分支到新上游，把 `43daecb` 与 `4e4fa50` 的改动
重新落到对应行号（两处逻辑都以 `config().providers.get(profile.id)` 为准绳），
`dev-build.yml` 的 `JCODE_BUILD_SEMVER` 改成新版本号，重打 `v0.82.0-dev`。
## 附带修复：config 里的 ox-alpha-free 陷阱（2026-08-28 追加）

诊断时发现 `~/.jcode/config.toml` 有 **两处** `ox-alpha-free`：
1. `[providers.opencode-go.models]` allowlist 第一条
2. `[provider] default_model`（全局默认！每个新会话首请求必挂 "Model ox-alpha-free is not supported"）

**`ox-alpha-free` 不在网关目录**（https://opencode.ai/zen/go/v1/models 实测 32 个模型无它），
疑似 OpenCode 其他产品线的旧模型名。两处均已替换为 `qwen3.7-max`（网关实测可用），
备份：`config.toml.bak-allowlist` / `config.toml.bak-defaultmodel`。
**勿把 ox-alpha-free 加回 allowlist 或 default_model**；用户自定义 allowlist 只应包含网关
真实存在的模型（验证命令见上）。

## v0.81.3-dev：memory sidecar 尊重 OpenAI base override（2026-08-28）

**症状**：每个会话持续报 `Memory consensus judge permanently misconfigured ... 401 Unauthorized
Incorrect API key provided: sk-hlONY...`。

**根因**：`crates/jcode-base/src/sidecar.rs` 的 `complete_openai` 硬编码
`https://api.openai.com/v1`（API-key 模式），而 daemon 的 openai-api provider 走
`JCODE_OPENAI_API_BASE=https://opencode.ai/zen/go/v1`，用的同一个 key（openai.env 的
OPENAI_API_KEY = opencode-go 网关 key）→ 拿网关 key 打 api.openai.com → 必 401。

**修复**：API-key 模式改走 `crate::provider::openai::resolve_api_base()`（依次尊重
JCODE_OPENAI_API_BASE / OPENAI_BASE_URL / OPENAI_API_BASE / ~/.codex/config.toml responses
base，最后回落 api.openai.com），与 openai-api provider 保持一致。顺手删掉不再使用的
`OPENAI_API_BASE` 常量。提交 `69d6619`，tag `v0.81.3-dev`，已部署 `versions/0.81.3`。

**验证**：日志出现 `Memory consensus rerank: 2/2 judges`（修复前必 401），
`openai-api:muse-spark-1.2-contributor` 仍 MUSE_V3 ✅。

**遗留噪音（非本次任务）**：内置 openai 槽的后台 catalog sweep 仍用网关 key 打
api.openai.com/v1/models → 周期 401 INFO（仅日志噪音，用户无 OpenAI 平台凭据；
/ model 的 OpenAI 分类为空属预期）。要消除可把 `model_picker_providers` 里的 `"openai"`
移除。
