# goose

本地运行的开源 AI Agent。它可以安装依赖、执行命令、修改文件并验证结果，并通过命令行、桌面应用和 [Agent Client Protocol](https://agentclientprotocol.com/)（ACP）接口提供同一套能力。

本仓库从上游 [aaif-goose/goose](https://github.com/aaif-goose/goose) 迁移而来，准备发布到 AtomGit，供 openEuler 社区获取源码、在本机构建和使用。当前版本 **1.54.0**，许可证为 Apache-2.0。

## 能做什么

- 在本机以交互会话或单次任务的方式完成开发、检索、写作和自动化。
- 接入多种模型服务，包括 OpenAI、Anthropic、Gemini 等在线接口，以及 Ollama、LM Studio 等本地推理服务。兼容 OpenAI、Anthropic 或 Ollama 接口的服务可以通过自定义 Provider 接入，不必等待发版。
- 通过 [Model Context Protocol](https://modelcontextprotocol.io/)（MCP）连接外部工具。
- 用 Recipe 描述可复用任务，用会话保留上下文，用定时任务重复执行。
- 以 ACP 服务的形式嵌入编辑器或其他客户端。

模型密钥由使用者在自己的环境中配置。openEuler 上从本仓库源码构建。

## 组件

| 路径 | 说明 |
| --- | --- |
| `crates/goose-cli` | `goose` 命令行 |
| `crates/goose` | Agent 主循环与会话 |
| `crates/goose-providers` | 模型 Provider |
| `crates/goose-mcp` | 随二进制提供的 MCP 服务 |
| `ui/desktop` | Electron 桌面应用 |
| `documentation/docs` | 使用与配置文档 |
| `examples` | MCP 与插件示例 |

## 在 openEuler 上构建命令行

以下步骤面向 openEuler 的 `dnf` 环境，在 x86_64 与 aarch64 上构建 `goose` 可执行文件。工具链版本以仓库根目录的 `rust-toolchain.toml` 为准，当前为 **Rust 1.99.0**。发行版自带的 Rust 版本通常不够，请使用 rustup。

### 系统依赖

```bash
sudo dnf install -y \
  git gcc gcc-c++ make cmake pkgconf-pkg-config openssl-devel clang
```

包名在不同 openEuler 版本上可能略有差异。某个包不存在时，用 `dnf search` 查找对应的开发包后再安装。

### 安装 Rust

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source "$HOME/.cargo/env"
```

进入本仓库后，rustup 会按 `rust-toolchain.toml` 安装指定工具链。

### 编译

```bash
git clone <本仓库地址>
cd goose
cargo build --release -p goose-cli --bin goose
```

产物为 `target/release/goose`。确认可以运行：

```bash
./target/release/goose --help
```

安装到当前用户：

```bash
mkdir -p "$HOME/.local/bin"
install -m 755 target/release/goose "$HOME/.local/bin/goose"
```

把 `$HOME/.local/bin` 加入 `PATH` 后即可直接执行 `goose`。

RISC-V 构建是实验性路径，见 [RISCV_SETUP.md](RISCV_SETUP.md)。

## 配置与使用

首次运行交互配置，选择 Provider 并填写对应的 API Key 或本地服务地址：

```bash
goose configure
```

之后可以直接进入会话，或执行一次任务：

```bash
goose session
goose run -t "说明当前目录里的项目结构"
```

常用命令：

| 命令 | 作用 |
| --- | --- |
| `goose configure` | 配置 Provider、模型和扩展 |
| `goose info` | 查看当前配置；`--check` 测试 Provider 连通性 |
| `goose doctor` | 检查本机环境是否可用 |
| `goose session` | 交互会话；`-r` 恢复上次会话 |
| `goose run -t "..."` | 用一段文本执行一次任务 |
| `goose run -i 文件` | 从指令文件执行；`-i -` 表示标准输入 |
| `goose session list` | 列出会话 |
| `goose recipe` | 校验和查看 Recipe |
| `goose acp` | 以 ACP Agent 方式在 stdio 上提供服务 |

也可以不写配置文件，临时用环境变量指定模型：

```bash
export GOOSE_PROVIDER=ollama
export GOOSE_MODEL=qwen2.5-coder
export OLLAMA_HOST=http://127.0.0.1:11434
goose run -t "用中文总结 README 里的构建步骤"
```

在线服务把 `GOOSE_PROVIDER` 换成 `openai`、`anthropic` 等名称，并设置该服务要求的 API Key 环境变量。Provider 与变量的完整列表见 `documentation/docs/getting-started/providers.md`。

本地模型需要先在本机启动推理服务。以 Ollama 为例，先拉取并运行模型，再把 goose 的 Provider 设为 `ollama`。

## 桌面应用

桌面应用在命令行二进制之外还需要 Node.js 22.9 或更高版本、pnpm 10，以及界面打包用的系统库。openEuler 上可按 Fedora 的依赖安装：

```bash
sudo dnf install -y dpkg-dev fakeroot libxcb-devel vulkan-headers vulkan-loader glslc
```

构建顺序是：先得到 release 版 `goose` 二进制，再在 `ui/desktop` 中安装前端依赖并打包。完整步骤、ZIP 与 DEB 产物路径见 [BUILDING_LINUX.md](BUILDING_LINUX.md)。容器构建见 [BUILDING_DOCKER.md](BUILDING_DOCKER.md)。

日常只使用命令行时，不必构建桌面应用。

## 开发

仓库用 Hermit 管理上游开发依赖（Rust、Node、pnpm、just）。在 x86_64 Linux 上可以启用：

```bash
source bin/activate-hermit
cargo build
cargo test -p goose
```

提交前格式化，并在需要时做静态检查：

```bash
cargo fmt
cargo clippy --all-targets -- -D warnings
```

Agent 主循环正在从 `crates/goose/src/agents/agent.rs` 迁移到 `crates/goose/src/agents/state_machine/`。状态机路径由环境变量 `GOOSE_STATE_MACHINE=1` 打开。修改主循环行为时，两条路径都要覆盖。

更细的约定见 [AGENTS.md](AGENTS.md) 和 [CONTRIBUTING.md](CONTRIBUTING.md)。其中的 GitHub Issue 看板和讨论渠道属于上游项目。

## 文档

| 文档 | 内容 |
| --- | --- |
| [BUILDING_LINUX.md](BUILDING_LINUX.md) | Linux 桌面应用构建 |
| [BUILDING_DOCKER.md](BUILDING_DOCKER.md) | 容器构建 |
| [RISCV_SETUP.md](RISCV_SETUP.md) | RISC-V 实验性构建 |
| `documentation/docs/getting-started/` | 安装、Provider、扩展 |
| `documentation/docs/guides/` | 会话、环境变量、配置文件 |

上游在线文档位于 <https://goose-docs.ai/>，描述的是上游发行版本。本仓库的行为以当前源码和上述文档为准。

## 参与贡献

与 openEuler 构建、打包和本仓库文档相关的问题，请在本仓库提交 Issue 或 Pull Request。

goose 的功能设计和上游缺陷仍由原项目处理：<https://github.com/aaif-goose/goose>。向原项目贡献代码时，遵循其 [CONTRIBUTING.md](CONTRIBUTING.md) 与 [GOVERNANCE.md](GOVERNANCE.md)。

## 上游与许可

- 上游仓库：<https://github.com/aaif-goose/goose>
- 许可证：[Apache License 2.0](LICENSE)
- 本仓库是引入 openEuler 社区的源码副本。上游的 GitHub Release、社区频道和治理角色仍以上游仓库为准；仓库内的 [GOVERNANCE.md](GOVERNANCE.md) 与 [MAINTAINERS.md](MAINTAINERS.md) 记录的是上游维护者。
