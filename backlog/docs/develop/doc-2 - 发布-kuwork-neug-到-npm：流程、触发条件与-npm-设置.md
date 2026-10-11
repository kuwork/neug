---
id: doc-2
title: 发布 @kuwork/neug 到 npm：流程、触发条件与 npm 设置
type: guide
created_date: '2026-10-11 06:22'
updated_date: '2026-10-11 06:27'
---
# 发布 @kuwork/neug 到 npm：流程、触发条件与 npm 设置（OIDC / Trusted Publisher 版）

本文档说明如何把 `alibaba/neug` 的 fork 以自有 scope `@kuwork` 发布到 npm，使 `npm i -g @kuwork/neug` 可用。采用 **npm Trusted Publisher（OIDC）** 方式：**无需在仓库配置任何长期 token**，由 npmjs.com 在 CI 触发时自动签发短期凭证完成发布（符合 OpenSSF 可信发布标准，且自动生成 provenance 溯源证明）。

覆盖范围：npm 账号/scope 配置、Trusted Publisher 设置、仓库侧改造、GitHub Actions 的发布触发条件，以及完整发布步骤与故障排查。

关联文档：
- `doc-1` — 6 平台 Fork 发布方案（设计依据、4 处改造清单、加载器/主包模板）
- `NEUG-1` — 首个发布任务（v0.2.1，6 平台，无 -cn dist-tag）

---

## 0. 本次发布基线

| 项 | 值 |
|---|---|
| 发布版本 | **v0.2.1**（npm 版本号 `0.2.1`） |
| 上游基线 | 分支 `release-v0.2.1`，tip `79c510c9496a62a937dd352a7efdf01b85d9e16c` |
| 该 commit 的 `NEUG_VERSION` | `0.2.1` ✅ |
| 禁止用作基线 | 本地 `main` / HEAD `68299c8`（含 #973 struct property，比 v0.2.1 新） |
| scope | `@kuwork`（主包 `@kuwork/neug`，平台子包 `@kuwork/neug-<os>-<arch>`） |
| dist-tag | `latest`（不使用任何 `-cn` 后缀） |
| 鉴权方式 | **OIDC Trusted Publisher（免 token）** |

> 为什么必须用 `release-v0.2.1` 而非本地 `main`：本地 `main`（`68299c8`）是比 v0.2.1 更新的提交，若基于它发布却叫 `0.2.1`，代码内容与版本号会错位。上游 v0.2.1 是在 `release-v0.2.1` 分支上发布、tip 即 `79c510c`。

---

## 1. npm 侧的设置（一次性，发布前完成）

### 1.1 注册并登录 npm
- 打开 https://www.npmjs.com ，注册账号并登录（若已有账号直接登录）。
- 确认邮箱已验证（未验证账号无法发布）。

### 1.2 创建 scope（组织 `@kuwork`）
scoped 包（`@kuwork/xxx`）要求你拥有该 scope 的所有权。

1. 点击右上角头像 → **Add Organization**（或 "Create an Organization"）。
2. 组织名填写 **`kuwork`**（对应 scope `@kuwork`）。
3. 选择计划：**免费计划**即可发布公开包（付费计划才支持私有包与高级协作）。
4. 完成后你是该 scope 的 owner，具备发布权限（用于首次 token 打底与包创建）。

> ⚠️ scope 名全局唯一。若 `kuwork` 已被他人占用，需换名（例如 `@kuwork-neug`），并同步修改第 4 节列出的所有 `@kuwork`/`@graphscope-neug` 字样。

### 1.3 两阶段鉴权策略（关键）
本方案采用 **Trusted Publisher（OIDC）** 作为正式发布方式，**仓库内不存放任何长期 token**。但有一个前提：

> **Trusted Publisher 是在「已存在的 package」设置页配置的**，因此目标包必须先在 npm 上创建出来。做法分两步：
> - **阶段 A（仅首次）**：用一次性 **Automation Token** 把 `@kuwork/neug`（及平台子包）发布一次，让包在 npm 上"存在"。
> - **阶段 B（之后所有发布）**：在 npmjs.com 上给该包配置 **Trusted Publisher**，此后 CI 通过 OIDC 自动发布，**不再需要 token**。

### 1.4 阶段 A：生成一次性 Automation Token（仅首次打底用）
- 头像 → **Access Tokens** → **Generate New Token** → **Classic Token → Automation**。
- 复制生成的 token（仅显示一次）。它只用于阶段 A 的首次 `npm publish`，用后即可从 GitHub secret 中删除。
- 把该 token 临时配到 `kuwork/neug` 仓库 `Settings → Secrets and variables → Actions → New repository secret`，Name 填 `NPM_TOKEN`（阶段 A 用完可删）。

### 1.5 阶段 B：在 npmjs.com 配置 Trusted Publisher（正式免 token 发布）
包经阶段 A 创建后，在 npmjs.com 上配置 OIDC 信任关系：

1. 进入 **npmjs.com → Packages → `@kuwork/neug` → Settings → Trusted publishing**（或页面中的 "Trusted Publisher" 区块）。
2. 点击 **Add trusted publisher** → 选择 **GitHub Actions**。
3. 填写字段（**全部区分大小写，须与仓库完全一致**）：
   - **Organization or user（必填）**：`kuwork`
   - **Repository（必填）**：`neug`
   - **Workflow filename（必填）**：`release-npm.yml`（只填文件名含扩展名，不要写路径）
   - **Environment name（可选）**：留空
   - **Allowed actions（必填）**：**务必勾选允许 `npm publish`**（2026-09-03 后新建的信任配置默认只允许 `npm stage publish`，不勾选则 CI 里的 `npm publish` 会失败）
4. 保存。

> 配置后效果：GitHub Actions 运行 `release-npm.yml` 时，npm CLI 通过 GitHub 下发的 OIDC ID-token 自动向 npm 换取短期发布凭证，**自动生成 provenance 溯源证明**，无需 `--provenance` flag，也无需 `NPM_TOKEN`。

### 1.6 最大安全（可选但推荐）
在 `@kuwork/neug` 的 **Settings → Publishing access** 选择 **"Require two-factor authentication and disallow tokens"**。这样即使阶段 A 的 token 已删除，也彻底禁止 token 发布，只走 OIDC。注意此设置不影响 Trusted Publisher（它用 OIDC，不是 token）。

> ⚠️ **隐性前提**：Trusted Publisher 要求主包 `package.json` 的 `repository.url` 字段**精确匹配** `kuwork/neug`。改 `@kuwork` 时必须同步把 `tools/nodejs_bind/assets/package.json` 里的 `repository` 改成 `kuwork/neug`（见第 4 节），否则发布报 `ENEEDAUTH`。

### 1.7 确认仓库 Actions 已启用
- `kuwork/neug` → Settings → Actions → General → 确保 **Allow all actions** 已开启（fork 仓库默认可能未启用，需手动打开）。

---

## 2. 仓库侧准备（fork 与基线）

```bash
# 1) 已 fork alibaba/neug → kuwork/neug，并 clone 到本地
cd /f/git/neug

# 2) 添加上游 remote 并获取发布分支（若尚未添加）
git remote add upstream https://github.com/alibaba/neug.git
git fetch upstream

# 3) 确认基线分支就位
git rev-parse upstream/release-v0.2.1     # 期望: 79c510c9496a62a937dd352a7efdf01b85d9e16c
git show upstream/release-v0.2.1:NEUG_VERSION   # 期望: 0.2.1

# 4) 确保工作区干净（无未提交改动），避免切分支丢数据
git status --short
```

---

## 3. npm 发布的触发条件（重点）

`release-npm.yml` 的真实 `on:` 段（来自 `release-v0.2.1` 分支）如下：

```yaml
name: Release NPM Packages
on:
  release:
    types: [published]
  workflow_dispatch:
```

### 3.1 触发事件 = 推送 `vX.Y.Z` 标签（类似 Backlog.md 的 `release.yml`）
- 把 `release-npm.yml` 的 `on:` 从 `release: [published]` 改为：
  ```yaml
  on:
    push:
      tags: ['v*.*.*']
    workflow_dispatch:
  ```
- **推送形如 `v0.2.1` 的标签即自动触发**：`git tag v0.2.1 && git push origin v0.2.1` → CI 自动构建 6 平台原生 addon 并经 OIDC 发布到 npm。
- **无需在 GitHub 手动建 Release**（这正是与上游原 `release: [published]` 机制的关键差异，也是用户要求的"推送标签自动构建 release 并触发 npm 拉取"）。
- `workflow_dispatch`：允许在 Actions 页面手动触发（用于调试，可选）。

### 3.2 与「在哪个分支打 tag」无关
- 改为 push-tag 触发后，版本号由 **`GITHUB_REF`** 派生（push tag 时 `GITHUB_REF=refs/tags/v0.2.1`），脚本中取 `VERSION="${GITHUB_REF##refs/tags/v}"`（与 Backlog.md 的 `release.yml` 一致）。
- 所以 **tag 必须叫 `v0.2.1`（带 `v` 前缀）**；分支名（main / release-v0.2.1 / release/v0.2.1）不影响发布。只要推送 `v0.2.1` 标签，CI 就能捕捉。

### 3.3 不会误触发
- `release-npm.yml` 听 `push: tags: ['v*.*.*']` 与 `workflow_dispatch`，**只有推送 `vX.Y.Z` 标签才触发**，普通分支 push（含发布分支的日常提交）**不会**误触发发版。
- 其 `build` job 复用 `npm-build-test.yml`（通过 `workflow_call`）。`npm-build-test.yml` 的 `on.push` 仅限 `branches: [main]`，与 tag 触发互不影响。

---

## 4. 需要改的文件（scope → @kuwork，平台 → 6 个，鉴权 → OIDC）

> 改完提交到发布分支，再走第 5 节的发布步骤。详细改法见 `doc-1`。

| 文件 | 改动 |
|---|---|
| `tools/nodejs_bind/assets/index.js` | 加载器 scope `@graphscope-neug` → `@kuwork` |
| `tools/nodejs_bind/assets/package.json` | name 改 `@kuwork/neug`；`optionalDependencies` 列出 6 个平台子包；**`repository.url` 必须改为 `kuwork/neug`**（OIDC 前置，否则 ENEEDAUTH） |
| `.github/workflows/release-npm.yml` | ① `on:` 由 `release: [published]` 改为 **`push: tags: ['v*.*.*']`**（类似 Backlog.md，推送标签即自动触发，无需建 Release）；② 确认 `publish` job 已含 `permissions: id-token: write` 与 `contents: read`（当前 release-npm.yml 已具备，勿删，OIDC 必需）；③ publish job 的 `setup-node` `node-version: '20'` → **`'24'`** 并 `npm install -g npm@11.21.0`（OIDC 要求 npm≥11.5.1 / Node≥22.14.0）；④ 去掉 `env: NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}`（改走 OIDC）；⑤ 版本号改用 `GITHUB_REF` 派生（`${GITHUB_REF##refs/tags/v}`）；⑥ publish 步骤的 `graphscope-neug-*-*.tgz` 改 `@kuwork` 对应包名并**扩到 6 平台**；⑦ 去掉"3 选齐即 exit 1"强校验；⑧ validate 脚本的包名同步改 |
| `.github/workflows/npm-build-test.yml` | `linux-arm64` 接入 release 流程；新增 `darwin-x64`、`win32-arm64` 两个 build job（win32-arm64 需调通 vcpkg/OpenSSL arm64 triplet 与 `node.lib`，最棘手） |

> `release-npm.yml` 的 `publish` job **已含 `id-token: write` 权限**（OIDC 必需），无需新增；按上表 ①⑥⑦⑧ 改触发/tarball/校验，③④⑤ 为运行时与版本派生调整。

6 个平台子包名（`optionalDependencies` 与目标三元组映射）：
- `@kuwork/neug-linux-x86_64` → linux-x64
- `@kuwork/neug-linux-arm64` → linux-arm64
- `@kuwork/neug-osx-arm64` → darwin-arm64
- `@kuwork/neug-osx-x86_64` → darwin-x64
- `@kuwork/neug-windows-x86_64` → win32-x64
- `@kuwork/neug-windows-arm64` → win32-arm64

---

## 5. 完整发布步骤

### 5.1 阶段 A — 首次用 token 打底（让包在 npm 上"存在"）
仅在**第一次**发布时需要；之后所有发布走阶段 B 的 OIDC。

```bash
# (1) 切到正确基线，建发布分支
git checkout -b release/v0.2.1 upstream/release-v0.2.1

# (2) 改第 4 节的 4 个文件（scope + 6 平台 + repository.url + OIDC 工作流），然后提交
git add -A && git commit -m "chore: publish @kuwork/neug v0.2.1 (6 platforms, OIDC ready)"

# (3) 推送发布分支
git push -u origin release/v0.2.1

# (4) 打标签（必须带 v 前缀）
git tag v0.2.1
git push origin v0.2.1
```

推送标签即自动触发（类似 Backlog.md 的 push-tag 机制）：
```bash
git push origin v0.2.1
```
→ 触发 `release-npm.yml`。阶段 A 时因 Trusted Publisher 尚未配置，CI 用仓库里的 `NPM_TOKEN`（阶段 A 的 Automation Token）完成首次 `npm publish`。

> 验证首次发布成功：`npm view @kuwork/neug version` 应返回 `0.2.1`。

### 5.2 阶段 B — 配置 Trusted Publisher 并验证（此后免 token）
1. 按第 1.5 节在 npmjs.com 给 `@kuwork/neug` 配置 Trusted Publisher（字段 `kuwork` / `neug` / `release-npm.yml`，勾选允许 `npm publish`）。
2. 删除 GitHub 仓库里的 `NPM_TOKEN` secret（阶段 A 已不需要；阶段 B 完全走 OIDC）。
3. **验证免 token 发布**：再次推送一个标签（如升版 `v0.2.2`，或另打测试标签）确认 `release-npm.yml` 在**没有 `NPM_TOKEN`** 的情况下仍能成功 `npm publish`，且包页面出现 **provenance** 标记。

### 5.3 后续发版（v0.2.2 等）
重复 5.1 的 (1)(2)(3)(4)（切基线、改文件、推分支、推标签）即可，**全程无需 token、无需建 Release**：基于新基线打新 tag（如 `v0.2.2`）→ `git push origin v0.2.2` → CI 经 OIDC 自动发布。

---

## 6. 故障排查

| 现象 | 原因 / 处理 |
|---|---|
| CI 完全没跑 | 标签格式不对（必须形如 `v0.2.1`，`v*.*.*` 才匹配）；或 fork 仓库 Actions 未启用（Settings→Actions 开启）；或推送的不是 tag 而是分支。 |
| 报 `ENEEDAUTH`（OIDC） | 最常见三类：① Trusted Publisher 里 **Workflow filename 拼错**（必须 `release-npm.yml` 含扩展名、区分大小写）；② 主包 `package.json` 的 `repository.url` 不是 `kuwork/neug`（fork 后未改）；③ 用的是 self-hosted runner（不支持，必须用 GitHub-hosted，当前 `ubuntu-22.04` 符合）。 |
| 报 "npm publish not allowed" | Trusted Publisher 配置时**未勾选允许 `npm publish`**（默认仅 `npm stage publish`）。去 npmjs.com 删掉该信任配置重建并勾选。 |
| 报 npm CLI 版本过低 | OIDC 需 npm≥11.5.1 / Node≥22.14.0。确认 `release-npm.yml` 的 `setup-node` 已升到 `node-version: '24'`。 |
| 版本号不是 0.2.1 | tag 没带 `v` 前缀（如打成 `0.0.1` 之类非 `vX.Y.Z`），`${GITHUB_REF##refs/tags/v}` 派生异常。改用 `v0.2.1` 形式。 |
| 只发布出 3 个平台 | `release-npm.yml` 的 publish 步骤仍硬编码 3 平台 tarball 名，未按第 4 节扩到 6 平台。 |
| win32-arm64 构建失败 | 原生依赖（vcpkg/OpenSSL arm64 triplet、node.lib）未调通，属预计 hardest 项，需单独攻关。 |
| 本地 `main` 被误改 | 发布分支应基于 `upstream/release-v0.2.1`，不要基于本地 `main`/`68299c8`。 |

---

## 7. 平台覆盖与已知风险

- 上游 0.2.1 官方预编译仅 3/6：`linux-x86_64`、`osx-arm64`、`windows-x86_64`。fork 目标补齐 `linux-arm64`、`osx-x86_64`、`windows-arm64`。
- **致命提醒**：6 个预编译包 ≠ 能直接塞进 `bun build --compile` 的独立二进制。Bun 不会把 `.node` 打进二进制，运行期仍需外部 `.node` 文件与正确 require 路径——这与原 Kuzu 打包缺陷同类。若走 service 模式（HTTP 客户端 + 独立 neug 服务进程），则无需把原生 addon 嵌入各平台可执行文件。
- 迁移期其他坑（来自 win32-x64 实测）：整数以 JS BigInt 返回需 `Number()` 转换；glog 向 stderr 打大量日志（可抑制）；内存模式临时目录关闭时清理失败（无害残留）。

---

## 8. 一句话流程

> fork → 加 upstream → 基于 `release-v0.2.1`(79c510c) 建 `release/v0.2.1` → 改 4 文件（@kuwork + 6 平台 + `repository.url`=`kuwork/neug` + 工作流升 Node 24/去 NPM_TOKEN）→ push 分支 → `git tag v0.2.1` → GitHub 把 tag **Publish 成 Release**（首次用 `NPM_TOKEN` 打底）→ 在 npmjs.com 给 `@kuwork/neug` 配 **Trusted Publisher**（kuwork/neug/release-npm.yml，勾选 npm publish）→ 删 `NPM_TOKEN` secret → 之后所有发布经 **OIDC 自动发布、免 token、带 provenance** → `npm i -g @kuwork/neug` 验证。
