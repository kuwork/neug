---
id: NEUG-1
title: 修改 neug 代码适配 @kuwork scope 与 6 平台（阶段A）
status: Done
assignee: []
created_date: '2026-10-11 07:38'
updated_date: '2026-10-11 08:48'
labels: []
dependencies: []
documentation:
  - backlog/docs/PRDS/publish/doc-1 - NeuG-6-平台-Fork-发布方案（kuwork-neug）.md
  - backlog/docs/develop/doc-2 - 发布-kuwork-neug-到-npm：流程、触发条件与-npm-设置.md
modified_files:
  - tools/nodejs_bind/assets/index.js
  - tools/nodejs_bind/assets/package.json
  - tools/nodejs_bind/Makefile
  - .github/workflows/release-npm.yml
  - .github/workflows/npm-build-test.yml
ordinal: 1000
actual_start: '2026-10-11 08:05'
actual_end: '2026-10-11 08:44'
---

## Description

<!-- SECTION:DESCRIPTION:BEGIN -->
基于 doc-1（方案）与 doc-2（发布指南）第 4 节改造 5 个文件，使 alibaba/neug 的 fork 能以 @kuwork scope 发布到 npm。本任务覆盖阶段A（首次用一次性 Automation Token 把包发布一次，让 @kuwork/neug 在 npm 上存在）所需的代码侧改造；npm 组织创建、token 生成、发布 release 触发等外部步骤由用户执行；OIDC/Trusted Publisher（阶段B）不在本任务范围。
<!-- SECTION:DESCRIPTION:END -->

## Acceptance Criteria
<!-- AC:BEGIN -->
- [x] #1 tools/nodejs_bind/assets/index.js：加载器 scope 由 @graphscope-neug/${os}-${arch} 改为 @kuwork/neug-${os}-${arch}
- [x] #2 tools/nodejs_bind/assets/package.json：name 改 @kuwork/neug；optionalDependencies 列出 6 个 @kuwork 平台子包；repository.url 改 git+https://github.com/kuwork/neug.git（保持 version 0.2.0 不修改）
- [x] #3 tools/nodejs_bind/Makefile：pack 生成包名改 @kuwork/neug-<os>-<arch>，tarball 名 kuwork-neug-*
- [x] #4 release-npm.yml：保持上游原有 on: release: types: [published] 触发（不做 push-tag 改造）；版本由 release 标签派生（兼容 release-vX.Y.Z）；保留 node 20 + NODE_AUTH_TOKEN(NPM_TOKEN) 的 token 发布路径；发布/校验扩到 6 平台
- [x] #5 npm-build-test.yml：上传 tarball 名 graphscope-neug-* 改 kuwork-neug-*；linux-arm64 开放 workflow_call；新增 macos-x86_64 与 windows-arm64 构建 job，并统一 6 平台 job 在 release 事件下均参与构建
<!-- AC:END -->

## Implementation Plan

<!-- SECTION:PLAN:BEGIN -->
1. 加载器 scope：tools/nodejs_bind/assets/index.js 的 @graphscope-neug/${os}-${arch} 改为 @kuwork/neug-${os}-${arch}。
2. 主包 package.json：name 改 @kuwork/neug；repository.url 改 kuwork/neug；optionalDependencies 列出 6 个 @kuwork 平台子包；version 保持 0.2.0 不修改（发布版本由标签派生）。
3. 打包 Makefile：pack 生成包名改 @kuwork/neug-<os>-<arch>，tarball 名 kuwork-neug-*。
4. release-npm.yml：保持上游原有 on: release: types: [published] + workflow_dispatch 触发（不做 push-tag 改造）；保留 node 20 + registry-url + NODE_AUTH_TOKEN(NPM_TOKEN) 的 token 发布路径；版本由 release 标签派生并兼容 release-vX.Y.Z 前缀；发布/校验从 3 平台扩到 6 平台。
5. npm-build-test.yml：上传 tarball 名 graphscope-neug-* 改 kuwork-neug-*；linux-arm64 开放 workflow_call；新增 macos-x86_64（macos-13）与 windows-arm64（windows-11-arm）构建 job；统一 6 平台 job 的 if 条件，使 release 事件下均参与构建。
6. 外部步骤（用户负责）：npm 建 @kuwork 组织 → 生成 Automation Token → 仓库配 NPM_TOKEN secret → 发布 release-v0.2.1 → 触发 CI 现场编译 6 平台并 npm publish。
<!-- SECTION:PLAN:END -->

## Implementation Notes

<!-- SECTION:NOTES:BEGIN -->
实现状态（本地工作区，未提交，待用户审阅）：
- 5 个文件改动：tools/nodejs_bind/assets/index.js、assets/package.json、Makefile、.github/workflows/release-npm.yml、.github/workflows/npm-build-test.yml。
- 关键修正 1：release-npm.yml 曾被整体重写为 OIDC 风格（node 24 / 升 npm 11.21.0 / Compute release version 派生步骤 / node -e 版本强写 / verify job），属阶段B 残留。已回归上游范式：on: release: types: [published] + workflow_dispatch；node 20 + registry-url + NODE_AUTH_TOKEN(NPM_TOKEN)；仅保留 scope 改名、6 平台发布/校验、release-v 标签前缀派生。
- 关键修正 2：修复会导致发布失败的 if 条件 bug —— build_npm_linux_arm64 与 build_npm_windows_arm64 原缺 github.event_name == release，release 触发时不会构建，而 release-npm.yml 要求 6 平台 tarball 齐全。已统一为 workflow_call || release || schedule || workflow_dispatch（用户确认"补齐 release，真 6 平台"）。
- 一致性已核对：6 平台 job 的 runner/产物名(kuwork-neug-*)/依赖链齐全；release-npm.yml 下载 pattern nodejs-*-<run_id> 匹配 6 个 artifact，PLATFORMS 数组与 tarball 名一一对应；package_test needs 含全部 6 job。JSON/JS 语法 OK，residual graphscope 仅剩 index.js 注释。
<!-- SECTION:NOTES:END -->
