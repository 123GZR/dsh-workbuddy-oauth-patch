# dsh-workbuddy-oauth 本地补丁快照

保存于 2026-11-06。用于 DSH 升级、插件重装后把国内版 WorkBuddy 的 400「请求被拦截」(code 11128) 修复重新贴回去。

## 目标文件

```
C:\Users\Administrator\.dsh\profiles\desktop\node_modules\dsh-workbuddy-oauth\lib\index.js
```

- 插件：`dsh-workbuddy-oauth` 0.0.4，来源 `github:molly-ovo/dsh-workbuddy-oauth`
- 锁定提交：`3fcb35d6a0d147b5e97bfc076c8d2a9a2ddcce29`（见 profile 的 pnpm-lock.yaml）
- profile：`desktop`（DSH Home = `C:\Users\Administrator\.dsh`）

## 本目录文件

| 文件 | 大小 | SHA256 |
| --- | --- | --- |
| `dsh-workbuddy-oauth-0.0.4-patched.js` | 225026 | `DAFF2B3685D6F43110A7160931B2046ACBE3030371D20C1CC9785B07FC82EABC` |
| `dsh-workbuddy-oauth-0.0.4-original.js` | 221913 | `A8EE89BEB7D1C09C5EBCD8A8B234A999D321C7E018D25B6F9790295A1AFE45FA` |
| `dsh-workbuddy-oauth-0.0.4.patch` | 5463 | 原版 → 补丁版的 unified diff（5 个 hunk） |

`-patched.js` 是当前在用的文件逐字节快照；`-original.js` 是插件 0.0.4 的原样文件；`.patch` 只用于在原版基础上重新生成补丁版。

## 补丁改了什么

1. `chatHeaders()`：补 `X-Agent-Intent`、`X-Agent-Purpose`、`X-Conversation-ID`、`X-Conversation-Request-ID`、`X-Conversation-Message-ID`、`X-Request-ID`（`X-Agent-Type` 仅国际区）。
2. `WorkBuddyUpstreamClient` 的 chat 请求：把请求体/请求头先算出来，并写调试日志到 `C:\Users\Administrator\AppData\Local\Temp\wb_debug.log`（Authorization 打码，body 截前 3000 字符）。
3. `prepareCnChatBody()` 末尾调用 `demoteCnSystemMessages()`：把 system / developer 消息原地降级为 `user` 角色（内容逐字节保留，顺序不变），并把 system 消息内容里的 `DeepSeek Harness` 一类品牌串替换为 `WorkBuddy`。

真正的关键修复是第 3 条：上游只检查 system 角色消息的内容，降级角色后请求即通过（2026-11-06 实测 HTTP 200）。

## 升级 DSH 后的情况

- DSH 自身升级只覆盖 `D:\deepseekharness`，不重装 profile 依赖，本补丁不受影响（依据：桌面端 `applyRelease()` 注释为 "prepare the external plugin profile without installing packages"，`initProfile` 对已存在的文件一律不写）。
- 会让补丁失效的：插件管理器里重装/更新 `dsh-workbuddy-oauth`、`pnpm install --force`、删除 profile、启动崩溃后的"禁用全部插件"恢复。

## 恢复（补丁被覆盖后）

用快照整文件覆盖即可（最稳，字节一致）：

```powershell
Copy-Item -LiteralPath "C:\Users\Administrator\.dsh\patches\dsh-workbuddy-oauth-0.0.4-patched.js" `
  -Destination "C:\Users\Administrator\.dsh\profiles\desktop\node_modules\dsh-workbuddy-oauth\lib\index.js" -Force
node --check "C:\Users\Administrator\.dsh\profiles\desktop\node_modules\dsh-workbuddy-oauth\lib\index.js"
```

然后完全退出 DSH 再启动（模块在启动时载入，热改不生效）。

## 用 patch 重新打补丁（插件被重装、磁盘上是新的原版文件时）

```powershell
# 1. 把重装后的原版覆盖到 patches 目录里的 original 文件
Copy-Item -LiteralPath "C:\Users\Administrator\.dsh\profiles\desktop\node_modules\dsh-workbuddy-oauth\lib\index.js" `
  -Destination "C:\Users\Administrator\.dsh\patches\dsh-workbuddy-oauth-0.0.4-original.js" -Force

# 2. 生成补丁版（必须关掉 autocrlf，否则换行会被改成 CRLF、哈希对不上）
Set-Location "C:\Users\Administrator\.dsh\patches"
Remove-Item -LiteralPath ".\dsh-workbuddy-oauth-0.0.4-patched.js" -Force -ErrorAction SilentlyContinue
git -c core.autocrlf=false apply ".\dsh-workbuddy-oauth-0.0.4.patch"

# 3. 贴回并校验
Copy-Item -LiteralPath ".\dsh-workbuddy-oauth-0.0.4-patched.js" `
  -Destination "C:\Users\Administrator\.dsh\profiles\desktop\node_modules\dsh-workbuddy-oauth\lib\index.js" -Force
node --check "C:\Users\Administrator\.dsh\profiles\desktop\node_modules\dsh-workbuddy-oauth\lib\index.js"
```

第 2 步若报 `patch does not apply`，说明插件已不是 0.0.4 的代码，别硬套：以 `-patched.js` 为参照，在新版本上手工重做同样的三处改动（新版本行号会变）。

## 校验补丁是否还在

```powershell
Get-FileHash "C:\Users\Administrator\.dsh\profiles\desktop\node_modules\dsh-workbuddy-oauth\lib\index.js" -Algorithm SHA256
# 期望：DAFF2B3685D6F43110A7160931B2046ACBE3030371D20C1CC9785B07FC82EABC
```

若应用了 patch 而非覆盖快照，哈希会不同（换行差异），此时看功能：发一条消息后 `wb_debug.log` 最后一段出现 `-> HTTP 200 (stream)` 即为成功。

## 收尾事项

- 调试日志做了硬编码路径（`C:/Users/Administrator/AppData/Local/Temp/wb_debug.log`）。确认稳定后可删掉 `debugAppend` 的三处调用和 `debugLogPath` 定义。
- 上游 `molly-ovo/dsh-workbuddy-oauth` 截至 2026-11-06：main 仍是 `3fcb35d6`（v0.0.4），PR #1/#2/#3 未合并，issue #4 仍 OPEN。若官方修复发布，可还原 `-original.js` 去掉补丁。

## 变更记录

- 2026-11-06：补丁写入 profile，快照保存到本目录。
- 2026-11-07 23:04：DSH 更新到 `0.2.2-rc.1`；插件管理器把 `dsh-session-manager` 升到 0.7.5 并重新解析 profile 依赖，`lib/index.js` 被换回 0.0.4 原版（`A8EE89…`），补丁丢失，WorkBuddy 再次 400。
- 2026-11-08：用本目录快照整文件覆盖恢复（哈希回到 `DAFF2B36…`，`node --check` 通过），重启 DSH；对 `copilot.tencent.com` 用真实账号实测，`model: hy4free` 下"system 降级为 user"的请求返回 HTTP 200。0.2.2-rc.1 仍在插件 peer 范围 `^0.2.0-rc.2` 内，无版本冲突。
- 另有一份同哈希副本：`C:\Users\Administrator\.dsh\workbuddy-oauth\index.js.patched`，也可作为恢复来源。
- 教训：插件管理器里任何一次安装/升级/删除都会重解 profile 依赖并覆盖手改的 `index.js`。动过插件后先跑一次哈希校验。
