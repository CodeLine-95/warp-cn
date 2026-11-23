# Warp 中文版 macOS 本地构建说明

> 目标：基于 `Heartcoolman/warp-cn`（官方 Warp 的中文 fork），合并更新的官方上游提交，在 macOS arm64 上构建出可安装、可设为默认终端的中文版 Warp。
> 本文记录了 2026-11-17 的完整实操过程与所有踩坑，供官方更新后重新构建时复用。

---

## 0. 结论速览

- 官方 Warp 无中文界面，二进制为 Rust 编译、硬编码英文，不可作为「语言包」汉化。
- 可行路径：使用 warp-cn 的源码级 i18n（Fluent `t!` 宏），**重新编译整个客户端**。
- 本机构建版本：`warp-cn` master（上游基线 2026-10-12）+ 官方上游 `63c8e7f7`（2026-11-12）。
- 产物：`Warp-cn.app`（Bundle ID `dev.warp.WarpCn`），release 约 376 MB，debug 约 830 MB。

---

## 1. 环境准备（按顺序）

```bash
# 1) Rust 工具链（需与 Cargo.lock 匹配）
rustup toolchain install 1.92.0 --profile minimal -c clippy -c rustfmt
rustc --version   # 1.92.0

# 2) cargo-bundle：最新版 0.12.x 因 sigstore 依赖在当前工具链编译失败，
#    必须用 0.10.0 且锁定依赖
cargo install cargo-bundle --version 0.10.0 --locked
cargo bundle --version   # cargo-bundle 0.10.0

# 3) git-lfs：仓库含 LFS 文件，clone 前必须装好，否则 checkout 阶段失败
brew install git-lfs

# 4) Xcode Metal 工具链：warpui/build.rs 会调用 `xcrun metal` 编译 .metal 着色器。
#    若报 "missing Metal Toolchain"，先修复 Xcode 首次启动，再下载组件：
xcodebuild -runFirstLaunch
xcodebuild -downloadComponent MetalToolchain   # 约 839 MB

# 5) 其余依赖本机已有：cmake、protoc、node、python3、pkg-config、jq
```

### 全局 GitHub 代理的坑

本机 `~/.gitconfig` 有 `url.https://githubproxy.cc/https://github.com/.insteadOf=https://github.com/`，
该代理在构建时返回 429，导致 cargo 拉取 git 依赖失败。构建期间临时移除，**结束后恢复**：

```bash
KEY='url.https://githubproxy.cc/https://github.com/.insteadOf'
OLD=$(git config --global --get "$KEY")
git config --global --unset-all "$KEY"
# ... 构建 ...
git config --global "$KEY" "$OLD"
```

---

## 2. 克隆与 LFS 占位（重要）

```bash
git clone https://github.com/Heartcoolman/warp-cn.git warp-cn-src
cd warp-cn-src
git lfs install --local
git checkout -f HEAD
git lfs pull    # 可能失败：仓库 LFS 对象在 GitHub 已丢失（batch 返回 404）
```

**LFS 对象丢失的处理**：仓库 7 个 LFS 文件无法下载，其中 macOS 构建只需要 3 个 ONNX 模型。
用等长占位文件满足编译期 `RustEmbed` 嵌入即可（只影响命令行自然语言分类，不影响界面汉化）：

```bash
for f in crates/input_classifier/models/onnx/bert_tiny_v{1,2,3}.onnx; do
  sz=$(git show "HEAD:$f" | awk '/^size /{print $2}')
  truncate -s "$sz" "$f"
done
# Windows PDB 同理，macOS 构建用不到，但占位避免 git 状态脏
```

> 注意：如果 `git status` 显示大量 `D`（索引被清空），先 `git reset --mixed HEAD` 修复索引，
> 并禁用 LFS 过滤器：`git config --local filter.lfs.required false` 等。

---

## 3. 合并官方上游（获取最新功能）

```bash
# 绕过代理拉取官方仓库
git -c 'url.https://github.com/warpdotdev/warp.git.insteadOf=https://github.com/warpdotdev/warp.git' \
    fetch --filter=blob:none --no-tags https://github.com/warpdotdev/warp.git master

# 选定目标提交（示例：2026-11-12）
TARGET=$(git rev-list -1 --before='2026-11-12' FETCH_HEAD)

# 安装 i18n-aware 合并驱动（自动解决纯 use 块冲突）
./script/install-merge-driver

git checkout -B build/nov11 master
git add -A && git commit -m 'chore: local LFS placeholders'
git merge --no-commit --no-ff "$TARGET"
```

本次合并只有 5 个冲突：3 个 GitHub 工作流（删除/取 theirs）+ 2 个 Rust 文件
（`app/src/pricing/addon_pack.rs`、`app/src/settings_view/billing_and_usage_page.rs`）。
Rust 冲突按「保留双方内容」解决（同一 hunk 内 ours + theirs 拼接），因为 ours 是 `t!()` 汉化函数、theirs 是上游新增常量，两者都被后续代码引用。

### 已知代码适配（上游 API 变更）

合并后需修两处，否则编译失败（本次已提交为 `e1d64d47c`）：

1. `crates/warp_server_client/src/lib.rs`：`HttpStatusError` 重复导出 → 合并为一行。
2. `app/src/server/direct_backend/computer_use_adapter.rs`：上游 `Action::MouseDown.at` 变为 `Option<Vector2I>` → 4 处用 `Some(...)` 包裹；
   注意 `Action::MouseWheel.at` 仍是 `Vector2I`，不要误改。

---

## 4. 构建

```bash
KEY='url.https://githubproxy.cc/https://github.com/.insteadOf'
OLD=$(git config --global --get "$KEY")
git config --global --unset-all "$KEY"

WARP_SKIP_COMMON_SKILLS_INSTALL=1 WARP_SKIP_GCLOUD_AUTH=1 \
  ./script/run --dont-open --release > /tmp/build.log 2>&1
RC=$?

git config --global "$KEY" "$OLD"   # 恢复代理
```

- 产物路径：`target/release/bundle/osx/Warp-cn.app`
- 首次 release 构建约 29 分钟（1000+ crate）；debug 约 4 分钟。
- `--dont-open` 很关键，否则构建完会自动启动。

---

## 5. 补齐打包（沙箱/权限 workaround）

构建脚本在 `prepare_bundled_resources` 复制内置 skills 时可能被沙箱拦截，且脚本在此中断。
手动补齐剩余步骤（**需在无沙箱环境执行**）：

```bash
APP="$PWD/target/release/bundle/osx/Warp-cn.app"

cp -R resources/bundled "$APP/Contents/Resources/bundled"
./script/copy_conditional_skills oss resources/channel-gated-skills \
   "$APP/Contents/Resources/bundled/skills"   # 本 fork 无 gated skills，会提示跳过
./script/compile_icon oss "$APP"              # 无 .icon bundle 时跳过，属正常
"$APP/Contents/MacOS/warp-oss" dump-settings-schema \
   "$APP/Contents/Resources/settings_schema.json"

codesign --force --deep --options runtime --sign - "$APP" \
   --entitlements script/Debug-Entitlements.plist
codesign --verify --deep --strict "$APP"
```

---

## 6. 安装与设为默认终端

```bash
cp -R target/release/bundle/osx/Warp-cn.app /Applications/Warp-cn.app
xattr -dr com.apple.quarantine /Applications/Warp-cn.app
```

### 6.1 默认终端（Launch Services）

Warp 内置的「设为默认终端」改的是 `public.unix-executable` 的 Shell 角色处理器。
系统 API `LSSetDefaultRoleHandlerForContentType` 在新 macOS 上**返回 0 但不生效**，
必须直接改用户偏好 `~/Library/Preferences/com.apple.LaunchServices/com.apple.launchservices.secure.plist`。

两个坑：
1. bundle ID 必须写**全小写** `dev.warp.warpcn`（系统规范化结果）。
2. `LSHandlerModificationDate` 必须是合理非零值（用已生效记录的值 `816601073` 即可）。

```python
import plistlib, os
p = os.path.expanduser('~/Library/Preferences/com.apple.LaunchServices/com.apple.launchservices.secure.plist')
d = plistlib.load(open(p,'rb'))
for item in d['LSHandlers']:
    if item.get('LSHandlerContentType') in ('public.unix-executable', 'com.apple.terminal.shell-script'):
        item['LSHandlerRoleShell'] = 'dev.warp.warpcn'
        item['LSHandlerModificationDate'] = 816601073
        item['LSHandlerPreferredVersions'] = {'LSHandlerRoleShell': '-'}
plistlib.dump(d, open(p,'wb'))
```

改完刷新并验证：

```bash
/System/Library/Frameworks/CoreServices.framework/Frameworks/LaunchServices.framework/Support/lsregister -seed -r
# 验证：两项应均为 dev.warp.WarpCn
swift -e 'import CoreServices; for t in ["public.unix-executable","com.apple.terminal.shell-script"] { print(t, LSCopyDefaultRoleHandlerForContentType(t as CFString, LSRolesMask.shell)?.takeRetainedValue() ?? "none") }'
# 功能验证：创建 .command 文件，NSWorkspace 应返回 /Applications/Warp-cn.app
```

> `com.apple.terminal.shell-script` 决定双击 `.command` 文件用哪个终端；官方 Warp 的菜单只改前者，两者需分别设置。
> 修改前建议备份 plist；恢复时拷回即可。

---

## 7. 验证清单

```bash
# 签名与隔离
codesign --verify --deep --strict /Applications/Warp-cn.app
xattr -l /Applications/Warp-cn.app      # 应为空
codesign --verify --deep --strict /Applications/Warp.app   # 官方版不受影响

# 中文确实编进二进制
grep -aoc '设置\|命令面板\|对话已恢复' \
  /Applications/Warp-cn.app/Contents/MacOS/warp-oss

# 版本信息
/usr/libexec/PlistBuddy -c 'Print :CFBundleIdentifier' /Applications/Warp-cn.app/Contents/Info.plist
/usr/libexec/PlistBuddy -c 'Print :CFBundleShortVersionString' /Applications/Warp-cn.app/Contents/Info.plist
```

---

## 8. 已知限制

- 3 个 ONNX 模型是占位文件：命令行自然语言输入自动检测不可用；界面汉化、终端、AI agent 均正常。
- ad-hoc 签名（无 Apple Developer ID）：首次启动需 `xattr -dr` 或「仍要打开」，应用内自更新走 GitHub Releases + minisign。
- 凭据存于 `~/Library/Application Support/dev.warp.WarpCn/`，AES-256-GCM 加密，不用钥匙串。
- fork 版本显示 `0.1.0`，与官方版本号不同，属正常。
- 屏幕录制/辅助功能未授权时无法自动截图验证界面，需人工目视确认。

---

## 9. 官方更新后如何重建

1. 重新 `git fetch` 官方 master，选新提交作 TARGET。
2. `git checkout build/nov11` 后 `git merge --no-commit --no-ff <new TARGET>`。
3. 处理冲突（重点看 `app/src/**` 里 `t!()` 函数与上游新增常量的冲突，保留双方）。
4. 检查第 3 节两处 API 适配是否仍需要（上游可能再次改动）。
5. 跑 `cargo xtask check-i18n --mode hard` 和 `--check-parity` 验证翻译对齐。
6. 按第 4、5 节重新构建、补齐、签名，覆盖 `/Applications/Warp-cn.app`（覆盖前先退出正在运行的应用）。

---

*记录于 2026-11-17，对应 fork commit `e1d64d47c`，上游 `63c8e7f7`。*
