# Multi Codex

保存多 provider Codex CLI 的可复用源码补丁。本仓库不保存个人会话、凭据或编译产物。

## 跨 provider 原地恢复

按 Codex 版本选择对应补丁：

- 当前版本 `0.159.2`：[`codex-0.159.2-cross-provider-resume.patch`](patches/codex-0.159.2-cross-provider-resume.patch)，基于 `rust-v0.159.2`，基线提交 `ff6aec96948b70d94983af2641a6b67c94faeff5`。
- 旧版本 `0.157.1`：[`codex-0.157.1-cross-provider-resume.patch`](patches/codex-0.157.1-cross-provider-resume.patch)，基于 `rust-v0.157.1`，基线提交 `36650394c5b38c2990ccf2a3457165ca3e9d9726`。

补丁让 `/resume`、`/resume <名称>` 和 `codex resume --last` 查找其他 provider 创建的会话，并用当前启动器的 provider 和模型沿用原 session ID 继续。请求副本会清理来源 provider 的加密状态、Responses 输入不接受的 `reasoning.content`，以及 OpenAI 不接受的历史 item ID；本地会话记录不会被改写。

## 从 0.159.2 基线构建

```sh
git clone --branch rust-v0.159.2 --depth 1 https://github.com/openai/codex.git codex-0.159.2
cd codex-0.159.2
git apply /path/to/multi-codex/patches/codex-0.159.2-cross-provider-resume.patch
cd codex-rs
cargo build --release -p codex-cli --bin codex
```

CLI 与托管 app-server daemon 是分开的可执行文件，更新时两处都要安装同一补丁版本，再运行 `codex app-server daemon restart`。所有启动器应共用一个 `CODEX_HOME`，并通过 profile 选择 provider。更新官方 CLI 会覆盖 standalone 二进制；新版本需要移植并重新应用对应补丁。

## 本地数据

仓库只跟踪源码补丁和文档，不包含会话历史、`CODEX_HOME` 数据库、认证信息、构建缓存或编译后的二进制。
