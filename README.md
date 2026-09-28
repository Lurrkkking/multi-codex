# Multi Codex

保存多 provider Codex CLI 的可复用源码补丁。本仓库以 OpenAI Codex CLI `0.157.1` 为基线，不保存个人会话或凭据。

## 跨 provider 原地恢复

补丁位于 [`patches/codex-0.157.1-cross-provider-resume.patch`](patches/codex-0.157.1-cross-provider-resume.patch)，基于 `openai/codex` 的 `rust-v0.157.1` 标签，基线提交为 `36650394c5b38c2990ccf2a3457165ca3e9d9726`。

它允许 `/resume`、`/resume <名称>` 和 `codex resume --last` 查询其他 provider 创建的会话。恢复时使用当前启动器的 provider 和模型，沿用原 session ID 原地继续，不创建 fork。对非 OpenAI provider 发送请求时，会移除 OpenAI 专用的加密状态；本地会话记录本身不会被改写。

会话列表仍沿用 Codex 的工作目录筛选行为；需要查看其他工作目录的会话时使用 picker 的全局显示选项。

## 从基线构建

```sh
git clone --branch rust-v0.157.1 --depth 1 https://github.com/openai/codex.git codex-0.157.1
cd codex-0.157.1
git apply --unidiff-zero /path/to/multi-codex/patches/codex-0.157.1-cross-provider-resume.patch
cd codex-rs
cargo build --release -p codex-cli --bin codex
```

CLI 和托管 app-server daemon 是分开的可执行文件。安装自建版本时，两处都要更新，再运行 `codex app-server daemon restart`，这样 picker 与实际模型请求会使用同一版代码。多个启动器应共用同一个 `CODEX_HOME`，并通过 profile 选择各自 provider。

## 本地数据

仓库只跟踪源码补丁和文档，不包含 Codex 会话历史、`CODEX_HOME` 数据库、认证信息、构建缓存或编译后的二进制。升级 Codex 基线时，需要检查并移植这份补丁。
