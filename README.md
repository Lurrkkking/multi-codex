# Multi Codex

保存跨 provider 原地恢复的 Codex 源码补丁和版本升级记录。本仓库不保存个人会话、认证信息或编译产物。

## 补丁适用版本

每个补丁只对应一个上游 tag，不能直接把旧版二进制补丁覆盖到新版源码。

- `0.159.2`：[`codex-0.159.2-cross-provider-resume.patch`](patches/codex-0.159.2-cross-provider-resume.patch)，上游基线 `rust-v0.159.2`，提交 `ff6aec96948b70d94983af2641a6b67c94faeff5`。当前安装版本。
- `0.157.1`：[`codex-0.157.1-cross-provider-resume.patch`](patches/codex-0.157.1-cross-provider-resume.patch)，上游基线 `rust-v0.157.1`，提交 `36650394c5b38c2990ccf2a3457165ca3e9d9726`。

补丁让 `/resume`、`/resume <名称>` 和 `codex resume --last` 查找其他 provider 创建的会话，并用当前启动器的 provider 和模型沿用原 session ID 继续。请求副本会清理来源 provider 的加密状态、不应作为 Responses 输入发送的 `reasoning.content`，以及 OpenAI 不接受的历史 item ID；本地会话记录不会被改写。

## 新版本升级流程

官方更新会替换 standalone CLI；managed app-server daemon 也有独立二进制。升级后如果跨 provider `/resume` 失效，先比较 CLI 和 daemon 版本：

```sh
codex --version
codexds --version
codex app-server daemon version
```

三者应显示同一版本。CLI 已升版但 daemon 仍显示旧版，或新版本没有对应补丁时，按下面流程移植一次。上游 release tag 通常是 `rust-v` 加 CLI 版本号，例如 `0.159.2` 对应 `rust-v0.159.2`。

### 1. 建立干净的新版本源码工作区

从已有的 `openai/codex` checkout 创建独立 worktree，保留旧版源码和补丁，后续比较更直接：

```sh
VERSION=0.160.0
TAG="rust-v${VERSION}"
SOURCE=/root/codex-resume-patch
WORKTREE="/root/codex-resume-patch-${VERSION}"
BASE_FILE="/root/autodl-tmp/codex-${VERSION}-base-commit"

git -C "$SOURCE" fetch --depth=1 origin "refs/tags/${TAG}:refs/tags/${TAG}"
git -C "$SOURCE" worktree add --detach "$WORKTREE" "$TAG"
cd "$WORKTREE"
BASE_COMMIT=$(git rev-parse HEAD)
printf '%s\n' "$BASE_COMMIT" > "$BASE_FILE"
```

下面的命令假设仍在同一个 Bash shell 中；如果重开 shell，重新设置 `VERSION`、`WORKTREE`、`BASE_FILE`，并从文件读取 `BASE_COMMIT`。

确认 `cat "$BASE_FILE"` 输出的是新 tag 提交，再从最新补丁开始移植：

```sh
git apply --3way /root/multi-codex/patches/codex-0.159.2-cross-provider-resume.patch
git status --short
```

没有冲突时，检查九个相关源码文件都已修改。若有冲突，保留新版本上游新增的字段和逻辑，把旧补丁的跨 provider 行为合并进去；解决后对冲突文件运行 `git add`。不要直接把旧版本的整个文件复制到新版本。

### 2. 格式检查并构建

复用一个固定的 Cargo target 目录，版本间可重用未变化的依赖缓存；不要每次都 `cargo clean`。同一 target 目录不要同时跑多个版本的构建：

```sh
cd "$WORKTREE/codex-rs"
cargo fmt --all -- --check
git -C "$WORKTREE" diff --check
export CARGO_TARGET_DIR=/root/autodl-tmp/codex-resume-build-target
cargo build --release -p codex-cli --bin codex
```

如果构建只改动了 `codex-rs/Cargo.lock`，而补丁没有依赖变更，先恢复它，避免把本机 Cargo 解析结果带进补丁：

```sh
git -C "$WORKTREE" status --short
git -C "$WORKTREE" restore -- codex-rs/Cargo.lock
```

确认构建产物版本：

```sh
"$CARGO_TARGET_DIR/release/codex" --version
```

### 3. 安装同一版本的 CLI 和 daemon

先备份上游安装的二进制。以下路径针对本机 `x86_64-unknown-linux-musl`；其他平台按实际 release 目录调整：

```sh
VERSION=0.160.0
ARCH=x86_64-unknown-linux-musl
export CODEX_HOME=/root/.codex
STANDALONE="$CODEX_HOME/packages/standalone/releases/${VERSION}-${ARCH}"
DAEMON="$CODEX_HOME/packages/app-server-daemon/releases/${VERSION}-${ARCH}"
BACKUPS=/root/autodl-tmp/codex-resume-backups

mkdir -p "$BACKUPS" "$DAEMON/bin"
cp -p "$STANDALONE/bin/codex" "$BACKUPS/codex-standalone-stock-${VERSION}"
cp -p "$STANDALONE/bin/codex-code-mode-host" "$BACKUPS/codex-code-mode-host-stock-${VERSION}"
readlink -f "$CODEX_HOME/packages/app-server-daemon/current" > "$BACKUPS/app-server-daemon-current-before-${VERSION}.txt"

# 文件可能正被运行中的 Codex 占用；先复制到临时文件，再原子替换。
cp "$CARGO_TARGET_DIR/release/codex" "$STANDALONE/bin/codex.new"
mv "$STANDALONE/bin/codex.new" "$STANDALONE/bin/codex"
cp "$CARGO_TARGET_DIR/release/codex" "$DAEMON/bin/codex"
cp "$STANDALONE/bin/codex-code-mode-host" "$DAEMON/bin/codex-code-mode-host"
ln -sfn "$DAEMON" "$CODEX_HOME/packages/app-server-daemon/current"

codex app-server daemon restart
codex app-server daemon version
codex --version
codexds --version
```

daemon 版本输出中的 `cliVersion`、`appServerVersion` 和 `managedCodexVersion` 应一致。四个启动器还必须共用同一个 `CODEX_HOME`，再通过 profile 选择 provider，否则它们看不到同一份会话历史。

### 4. 做一次跨 provider 冒烟检查并保存新补丁

用另一个 provider 创建的低风险会话，在新 launcher 中执行 `/resume`，确认 picker 能找到它，并能原地继续。随后把本次移植保存成对应版本的补丁：

```sh
cd "$WORKTREE"
BASE_COMMIT=$(cat "$BASE_FILE")
git diff --check "$BASE_COMMIT"
git diff --binary "$BASE_COMMIT" > "/root/multi-codex/patches/codex-${VERSION}-cross-provider-resume.patch"
```

在 `README.md` 的补丁版本列表中登记新 tag 和 `BASE_COMMIT`，再提交并推送 README 与补丁文件：

```sh
cd /root/multi-codex
git add README.md "patches/codex-${VERSION}-cross-provider-resume.patch"
git commit -m "Add cross-provider resume patch for ${VERSION}"
git push origin main
```

这样下次从最新版本补丁移植，不必重新从 0.157.1 开始。

## 回退

若新构建启动失败，恢复 stock CLI，并把 daemon `current` 指回升级前记录的 release：

```sh
PREVIOUS_DAEMON=$(cat "$BACKUPS/app-server-daemon-current-before-${VERSION}.txt")
cp "$BACKUPS/codex-standalone-stock-${VERSION}" "$STANDALONE/bin/codex.restore"
mv "$STANDALONE/bin/codex.restore" "$STANDALONE/bin/codex"
ln -sfn "$PREVIOUS_DAEMON" "$CODEX_HOME/packages/app-server-daemon/current"
codex app-server daemon restart
```

旧 release 目录和会话数据都保留，不要清理 `.codex/sessions` 或数据库。

## 本地数据

本仓库只跟踪源码补丁和文档。不要提交会话历史、`CODEX_HOME` 数据库、认证信息、构建缓存或编译后的二进制。
