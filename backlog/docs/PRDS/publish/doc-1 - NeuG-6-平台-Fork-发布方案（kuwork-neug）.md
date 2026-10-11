---
id: doc-1
title: NeuG 6 平台 Fork 发布方案（@kuwork/neug）
type: specification
created_date: '2026-10-11 05:31'
updated_date: '2026-10-11 05:38'
---
# NeuG 6 平台 Fork 发布方案（@kuwork/neug）

> 目标：fork `alibaba/neug`，以 `@kuwork/*` 自有 scope 重新发布 Node.js 绑定，预编译覆盖全部 6 个原生三元组
> `linux-x64 / linux-arm64 / darwin-x64 / darwin-arm64 / win32-x64 / win32-arm64`。
> 本文档位于 fork 仓库 `F:\git\neug` 的 `backlog/docs/PRDS/neug/` 下。

## 结论：可以，且工作量可控

- **法律层面**：NeuG 为 Apache-2.0，明确允许 fork、修改、再发布。以 `@kuwork/*` 自有 scope 发布衍生包完全合规。
- **技术层面**：加载器已原生支持 6 平台映射，linux-arm64 实际每晚都在构建（只是没发布）。真正要补的是「构建 + 发布」3 个缺失平台 + 改 scope。

---

## 一、法律义务（Apache-2.0 再发布）

1. 保留 `LICENSE` / `NOTICE` 文件，不删除原作者版权声明。
2. 在 README / CHANGELOG 明确标注为 **fork** 及所做改动。
3. 不得用 "GraphScope" / "NeuG" 商标暗示官方背书——用 `@kuwork/neug` scope 即满足；包内代码沿用 NeuG 名称但需声明 fork 来源。
4. 每个平台包（`@graphscope-neug/linux-x86_64` 等）同样 Apache-2.0，可重建并以 `@kuwork/neug-linux-x86_64` 等再发布。

---

## 二、关键发现（来自官方 CI 源码核实）

### 1. 加载器已支持全部 6 平台，无需改逻辑
`tools/nodejs_bind/assets/index.js` 用 `OS_MAP` / `ARCH_MAP` 把 `process.platform` / `process.arch` 映射到
`@graphscope-neug/<os>-<arch>`，缺失就抛错。映射关系：

| process.platform | process.arch | 解析到的包名 |
|---|---|---|
| linux | x64 | `@kuwork/neug-linux-x86_64` ✅ |
| linux | arm64 | `@kuwork/neug-linux-arm64` ❌→补 |
| darwin | arm64 | `@kuwork/neug-osx-arm64` ✅ |
| darwin | x64 | `@kuwork/neug-osx-x86_64` ❌→补 |
| win32 | x64 | `@kuwork/neug-windows-x86_64` ✅ |
| win32 | arm64 | `@kuwork/neug-windows-arm64` ❌→补 |

**只要包名遵循 `<os>-<arch>` 约定，补 3 个平台只需「构建 + 发布对应包」，加载代码一行都不用动。**
（改好 scope 的 `index.js` 见本文档末尾附录。）

### 2. linux-arm64 其实已在每晚构建，只是没发布
`npm-build-test.yml` 里有一个 `build_npm_linux_arm64` job，使用
`neug-registry.../neug/neug-manylinux:v0.1.3-arm64-node` 容器 + `ubuntu-22.04-arm` runner，产出
`graphscope-neug-linux-arm64-*.tgz`。但该 job 的 `if` 仅限 `schedule` / `workflow_dispatch`，且
`release-npm.yml` 的 publish 步骤只校验并发布 3 个包（`linux_x86_64` / `osx_arm64` / `windows_x86_64`，缺失即 `exit 1`）。
→ 说明 **C++ 内核在 arm64 Linux 编译没问题**，补这个平台几乎是「接线」而非「攻关」。

### 3. 构建系统 = CMake + node-addon-api（N-API），不是 napi-rs
核心命令（来自 Windows job）：
```
cmake -B build -G Ninja -DBUILD_NODEJS=ON -DVCPKG_TARGET_TRIPLET=x64-windows-static-md \
      -DNODE_INCLUDE_DIR=... -DNODE_IMPORT_LIB=... .
cmake --build build --target neug_node_bind
```
产物：`neug_node_bind.node`（Windows 另含 `neug.dll`）。加载靠 N-API，ABI 稳定，**不依赖进程名**——这正是 Kuzu 那个 `node.exe` hack 的根源，NeuG 天然没有。
→ 加平台 = 为对应工具链写构建 job + 正确设置 `NODE_INCLUDE_DIR` / `NODE_IMPORT_LIB` / `node.lib`。

---

## 三、fork 内需改动的文件

### A. `tools/nodejs_bind/assets/index.js`
将硬编码 `@graphscope-neug` 改为 `@kuwork`。

### B. `tools/nodejs_bind/assets/package.json`
- `name` → `@kuwork/neug`
- `optionalDependencies` 扩到 6 项：`@kuwork/neug-linux-x86_64` / `osx-arm64` / `windows-x86_64` / `linux-arm64` / `osx-x86_64` / `windows-arm64`
- `repository.url` 指向你的 fork

### C. `.github/workflows/release-npm.yml`
- 去掉三选齐的硬 `exit 1` 约束，改为 6 个包的循环校验 + 发布。
- `validate_npm_version.py` 的 `--optional-dependency` 扩到 6 个（含新 scope 名）。
- 下载/解包/发布的 tarball 列表从 3 增到 6。

### D. `.github/workflows/npm-build-test.yml`
- `build_npm_linux_arm64`：把 `if` 放宽到 `release` 事件（或保留 schedule/dispatch 手动发版）。
- **新增 `build_npm_macos_x86_64`**：runner `macos-13`（x64 原生）或 `macos-15` + `-DCMAKE_OSX_ARCHITECTURES=x86_64`；复用现有 macOS arm64 job 的依赖安装（`scripts/install_deps.sh`、delocate 捆绑）。
- **新增 `build_npm_windows_arm64`**：runner `windows-11-arm`（GitHub 公开预览）或 `windows-2022` + ARM64 MSVC 交叉；vcpkg 切换到 `arm64-windows-static-md`、OpenSSL `arm64-windows-static-md`、`node.lib` 取 `nodejs.org/dist/<ver>/win-arm64/node.lib`。注意 OpenSSL 的 arm64 vcpkg triplet 与 `NODE_IMPORT_LIB` 路径需调通——这是 6 个平台里**最棘手的一个**。
- 每个新 job 最后 `npm pack` 时把包名写成 `@kuwork/neug-<os>-<arch>` 并加 `os` / `cpu` 字段（Windows job 已有现成写法可照搬；linux/macos 走 `make pack` 模板，需在模板里同步改 scope 与包名）。

---

## 四、各平台可行性 / 置信度

| 平台 | 官方现状 | fork 动作 | 置信度 |
|---|---|---|---|
| linux-x64 | ✅ 已发布 | 改 scope | 高 |
| darwin-arm64 | ✅ 已发布 | 改 scope | 高 |
| win32-x64 | ✅ 已发布(0.2.1) | 改 scope | 高（**本机 win32-x64 实测可加载**）|
| linux-arm64 | 每晚构建未发布 | 接入 release 发布 | 高（arm64 容器已存在）|
| darwin-x64 | 无 | 新增 macOS x64 job | 中高（同 macOS 工具链）|
| win32-arm64 | 无 | 新增 ARM64 Win job | 中（vcpkg/OpenSSL arm64 + node.lib 需调通）|

---

## 五、⚠️ 致命提醒：6 个预编译包 ≠ 能在 `bun build --compile` 独立二进制里用

这是整个方案最容易被忽略、也最可能让你重蹈 Kuzu 覆辙的一点：

- Kuzu 当年的缺陷 = 「`.node` 原生 addon 路径被写死成构建机绝对路径 + Windows 进程名不对」。
- NeuG 用 N-API，靠 `require('@kuwork/neug-<os>-<arch>')` 加载；而 **`bun build --compile` 不会把 `.node` 打进二进制**，运行期它必须作为外部文件存在，且 `require` 路径要在运行期正确解析。
- 所以 **fork 解决的是「安装期 6 平台覆盖」，但「独立二进制运行期」仍要解决原生 addon 的落盘与加载路径**——这和 Kuzu 是**同类问题**。

两条出路：

1. **embedded 路线（把 addon 嵌进 6 个独立二进制）**
   - 每个平台二进制旁边放对应 `neug_node_bind.node`，运行期基于可执行文件位置计算相对/绝对路径 `dlopen`，**绝不在构建期写死绝对路径**。
   - 需先在本机 win32-x64 验证 `bun build --compile` + 外部 `.node` 加载能否跑通——这是真正决定能否 1:1 替代 Kuzu 打包缺陷的一步。

2. **service 路线（dual-mode）**
   - 原生引擎作为独立 `neug` 服务进程跑在受支持主机（linux-x64 / darwin-arm64 / win32-x64 已 ✅），6 个平台只发**纯 JS HTTP 客户端**（`NeugServiceGraphStore`）。
   - **此路线下 fork 的 6 平台其实不需要**——客户端零原生依赖，官方 NeuG 已够用。

---

## 六、建议的下一步

- 若走 **service 模式**：不必 fork，直接在 3 个主机平台用官方 NeuG（Python/C++ 服务），客户端纯 JS。
- 若坚持 **embedded + 6 平台独立二进制**：fork 可行，工作量集中在上面 (A)~(D) 四文件/CI 改动；但**真正决定成败的是运行期 `.node` 落盘 + 加载路径设计**，建议先在本机 win32-x64 做 `bun build --compile` + 外部 `.node` 加载 spike。

---

## 附录：改好的加载器 `tools/nodejs_bind/assets/index.js`

```js
'use strict';

const OS_MAP = { linux: 'linux', darwin: 'osx', win32: 'windows' };
const ARCH_MAP = { x64: 'x86_64', arm64: 'arm64' };

const os = OS_MAP[process.platform];
const arch = ARCH_MAP[process.arch];
if (!os || !arch) {
  throw new Error(`NeuG: unsupported platform ${process.platform}-${process.arch}`);
}

// kuwork fork: scope changed from @graphscope-neug to @kuwork
const optionalPkg = `@kuwork/neug-${os}-${arch}`;

let binding;
try {
  binding = require(optionalPkg);
} catch (err) {
  throw new Error(
    `NeuG: failed to load platform package "${optionalPkg}".\n` +
    `Original error: ${err.message}`
  );
}

module.exports = binding;
```

## 附录：改好的主包模板 `tools/nodejs_bind/assets/package.json`

```json
{
  "name": "@kuwork/neug",
  "version": "0.2.1",
  "description": "NeuG Graph Database Node.js bindings (kuwork fork) - high performance embedded graph database with Cypher query support",
  "main": "index.js",
  "keywords": ["graph", "database", "embedded", "cypher", "neug", "native", "addon", "napi"],
  "author": "GraphScope Team (forked & republished by kuwork)",
  "license": "Apache-2.0",
  "repository": {
    "type": "git",
    "url": "git+https://github.com/kuwork/neug.git"
  },
  "engines": { "node": ">=20.0.0" },
  "publishConfig": { "access": "public" },
  "optionalDependencies": {
    "@kuwork/neug-linux-x86_64": "0.2.1",
    "@kuwork/neug-osx-arm64": "0.2.1",
    "@kuwork/neug-windows-x86_64": "0.2.1",
    "@kuwork/neug-linux-arm64": "0.2.1",
    "@kuwork/neug-osx-x86_64": "0.2.1",
    "@kuwork/neug-windows-arm64": "0.2.1"
  },
  "files": ["index.js"]
}
```
