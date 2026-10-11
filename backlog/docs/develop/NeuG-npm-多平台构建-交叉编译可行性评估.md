# NeuG npm 多平台构建：交叉编译可行性评估

- **背景**：@kuwork/neug 需发布 6 平台 npm 包（linux-x86_64 / linux-arm64 / osx-arm64 / osx-x86_64 / windows-x86_64 / windows-arm64）。
- **触发问题**：`build_npm_macos_x86_64` 原本用 `runs-on: macos-13`，该镜像已于 **2025-12-04 EOL**，job 永久排队（实测卡住 1h50m+）。
- **参考对照**：`Backlog.md/.github/workflows/release.yml` 只用 `macos-latest` 一种 macOS runner 就能产出 `bun-darwin-x64`，为何 NeuG 不行？
- **结论**：NeuG 不适合改交叉编译，采用**方案 A（原生编译 + `macos-15-intel`）**。

## 一、为什么 Backlog.md 能出 darwin-x64，NeuG 不能

| 维度 | Backlog.md | NeuG |
|---|---|---|
| 产物类型 | Bun 编译的单文件可执行程序 | C++20 N-API 原生扩展（`.node`） |
| 构建方式 | **交叉编译**：`bun build --compile --target=<target>` | **原生编译**：CMake + 平台原生工具链 |
| macOS runner | 只用 `macos-latest`（arm64）即可编 darwin-x64/arm64 | 必须 arm64 与 x86_64 runner **各一台**分别原生编译 |
| 覆盖 6 平台所需 runner 数 | 3 种（ubuntu/macos/windows-latest） | 6 种（每个目标平台一台原生 runner） |
| 受 macos-13 EOL 影响 | 否（从不使用 macos-13） | 是（原用 macos-13 编 darwin-x64） |

**关键差异**：Bun 官方内置并维护了各平台交叉编译工具链与运行时，`--target` 可跨平台产出。NeuG 是自研 CMake + 一长串 C++ 第三方依赖，无现成交叉编译工具链。

## 二、NeuG 交叉编译的分档可行性

| 目标 | 可行性 | 说明 |
|---|---|---|
| linux-x64 → linux-arm64 | ⚠️ 勉强 | 需 aarch64 交叉工具链，且**每个**第三方库都要能交叉编译（`cmake/Build*ThirdParty.cmake` 十多个：abseil / arrow / protobuf / brpc / leveldb / lz4 / re2 / mimalloc / parquet …） |
| linux → windows | ❌ 极难 | 需 mingw-w64/clang-cl 交叉链，且要交叉编全部重型依赖，基本不现实 |
| 任意 → darwin-x64 / darwin-arm64 | ❌ 不可行 | macOS 编译须用 Apple Xcode/clang + Apple SDK；异平台交叉编 darwin 需 osxcross 并自备 Xcode 工具链（涉版权），CI 中不可行 |

## 三、核心障碍（NeuG 特有）

1. **全量第三方库需交叉编译**：十几个重型 C++ 依赖，每个都要提供交叉编译支持——最大工作量。
2. **N-API 平台绑定**：Windows 需 per-arch 的 `node.lib`；macOS 需对应 SDK。
3. **平台原生工具链不可替代**：MSVC `cl`、`xcodebuild` 无法在异平台运行。
4. **`NeugNativeArch.cmake` 与平台 RPATH**：均假设本机原生构建。

## 四、方案对比

| 方案 | 做法 | 评价 |
|---|---|---|
| **A（已采用）** | 保持原生编译，darwin-x64 用 `macos-15-intel` | ✅ 零改造、可靠；官方支持至 2027-08。**已实施** |
| B | 放弃 darwin-x64，只发 5 平台 | 若 macos-15-intel 亦不可用时最省事的降级 |
| C | 仅对 linux 内部做交叉（x64 编 arm64） | 可省一台 runner，但对 darwin/windows 无效，收益有限；需单独评估，不纳入本任务 |
| D | 架构级重写（核心做成独立可执行 + Node 薄壳） | 远超本任务范围 |

## 五、决定

采用**方案 A**：`build_npm_macos_x86_64` 的 runner 由退役的 `macos-13` 改为官方保留的 x86_64 镜像 **`macos-15-intel`**（注释保留退役说明）。其余 runner 标签经官方清单核验均有效：`ubuntu-22.04` / `ubuntu-22.04-arm` / `windows-2022` / `macos-15` / `windows-11-arm`。

## 附：官方 macOS runner 参考（2026-10）

- `macos-13`：**EOL 2025-12-04**
- `macos-15-intel` / `macos-26-intel`：**x86_64（Intel）** 保留镜像，`macos-15-intel` 官方称可用至 2027-08（最后一个 x86_64 镜像）
- `macos-15` / `macos-26` / `macos-latest`：arm64（M1）
