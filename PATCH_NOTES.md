# jcode 补丁说明（给未来的自己 / jcode 会话看）

## 这是什么补丁
文件：`crates/jcode-provider-openrouter-runtime/src/lib.rs` 约 1716-1745 行
提交：`fix/provider-model-allowlist` 分支的 `fix(providers): honor pinned model allowlist`（df6ceea 起）
标题：`v0.81.1-dev`（包名 `jcode-0.81.1-dev-linux-x86_64.tar.gz`，Release 已发布）

**一句话**：让 `[providers.opencode-go]` 这类具名配置真正生效——当你写了 `model_catalog = false` 并只列 5 个好模型（ox-alpha-free/minimax-m3/kimi-k3/glm-5.3/deepseek-v4-pro），运行时就只用这 5 个，不再把远程拉来的 31 个（含 `muse-spark-1.2-contributor`）放进选择器，避免 503 / service_overloaded。

## 为什么需要
上游 0.81.1 对具名 provider 本已支持 `model_catalog=false`，但内置 `opencode-go` 的 profile 路径会忽略覆盖，仍用 live 目录。补丁在 `new_openai_compatible_profile_runtime` 里加了 `named_override` 分支：查 `config().providers.get(profile.id)`，若是 `OpenAiCompatible` 就用 `cfg.models` 和 `cfg.model_catalog`。

## 分支模型
- `master` = 上游 `1jehuang/jcode` 的 `master` + `dev-build.yml`（只编 x86_64 的 -dev 包，无需私有 secret）
- `fix/provider-model-allowlist` = `master` + 本补丁 + `changelog/v0.81.1-dev.json`
- `v0.81.1-dev` tag 指向 `fix` 分支头，已由 GHA 编出 53MB 包并发布。`--version` 仍显 `v0.81.1`（jcode 的 `parse_semver` 只认纯数字，`-dev` 后缀进不了版本号，属上游限制，不影响包身份）

## 上游发新版时怎么跟进（例：v0.82.0）
```bash
cd /home/user/code/jcode-rel
git checkout master
git fetch upstream
git merge upstream/master   # 或 git reset --hard upstream/master
git push origin master

git checkout fix/provider-model-allowlist
git rebase master           # 若有冲突，修 lib.rs 的 named_override 段
git push -f origin fix/provider-model-allowlist

# 重打 -dev tag 并触发编译
git tag -f -a v0.82.0-dev -m "dev build on v0.82.0 with pinned allowlist" HEAD
git push origin :refs/tags/v0.81.1-dev  # 清旧 tag 可选
git push origin v0.82.0-dev
# 或手动：gh workflow run dev-build.yml --ref v0.82.0-dev
# 编好后：gh release edit v0.82.0-dev --draft=false
```

## 给未来 jcode 会话的入口
- 想看补丁：`git log --oneline fix/provider-model-allowlist` 或 `git show df6ceea`
- 想看构建：`gh run list --workflow dev-build.yml --limit 3` / `gh release view v0.81.1-dev --json assets`
- 本机当前 daemon 就是该 -dev 包（sha `130ea505`，`jcode-0.81.1-dev-linux-x86_64`），回滚备份在 `~/.jcode/builds/versions/0.81.1/*.bak*`
