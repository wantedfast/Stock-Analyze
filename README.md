# Stock Analyze

用 Codex App Server 做本地 A 股产业链分析。网页或脚本发送股票分析 prompt，服务端把请求转给 `codex app-server`，可以显式注入本机 Codex skill，例如 `stock-reverse-engineering`，并把 Codex 的完整回答返回给网页或写入文件。

这个项目不使用 `codex exec`，也不模拟点击桌面 App。

## 能做什么

- 从网页输入股票分析请求。
- 通过 Codex App Server 创建 thread/turn。
- 指定 Codex skill：`skillName + skillPath`。
- 默认关闭审批弹窗：`approvalPolicy = never`。
- 默认只读沙箱，适合自动化研究任务。
- 支持 PowerShell 一键启动。
- 无 npm 第三方依赖。

## 前置条件

另一台电脑需要：

1. Windows + PowerShell。
2. 已安装并登录 Codex App。
3. Node.js 20+，或 Codex 本机 runtime 里带的 Node。
4. 如果要用股票逆向工程 skill，需要本机存在：

```text
C:\Users\<你>\.codex\skills\stock-reverse-engineering\SKILL.md
```

`start.ps1` 会自动优先查找：

```text
C:\Users\<你>\AppData\Local\OpenAI\Codex\bin\...\codex.exe
```

这是可被脚本调用的用户本地 Codex binary。不要直接用 `C:\Program Files\WindowsApps\...` 里的 Store 应用资源，Windows 可能会返回 `Access is denied`。

## 快速启动

```powershell
git clone https://github.com/wantedfast/Stock-Analyze.git
cd Stock-Analyze
.\start.ps1
```

打开：

```text
http://127.0.0.1:3000
```

如果你想先不调用 Codex，只测试网页和 API：

```powershell
.\start.ps1 -Mock
```

## 指定股票分析 Skill

网页上可以填写：

```text
skillName: stock-reverse-engineering
skillPath: C:\Users\<你>\.codex\skills\stock-reverse-engineering\SKILL.md
```

也可以启动时一次性指定：

```powershell
.\start.ps1 `
  -SkillName stock-reverse-engineering `
  -SkillPath "$env:USERPROFILE\.codex\skills\stock-reverse-engineering\SKILL.md"
```

之后网页里只需要写 prompt，例如：

```text
请使用 $stock-reverse-engineering 技能，分析华海清科（688120.SH）的产业链地位。
```

## 命令行发送 Prompt

先启动服务：

```powershell
.\start.ps1 `
  -SkillName stock-reverse-engineering `
  -SkillPath "$env:USERPROFILE\.codex\skills\stock-reverse-engineering\SKILL.md"
```

另开一个 PowerShell：

```powershell
$env:CODEX_SKILL_NAME = "stock-reverse-engineering"
$env:CODEX_SKILL_PATH = "$env:USERPROFILE\.codex\skills\stock-reverse-engineering\SKILL.md"
node .\tools\post-prompt-file.mjs .\examples\huahai-qingke.txt .\huahai-result.json
```

如果 `node` 不在 PATH，但 Codex runtime 有 Node，可以这样跑：

```powershell
& "$env:USERPROFILE\.cache\codex-runtimes\codex-primary-runtime\dependencies\node\bin\node.exe" `
  .\tools\post-prompt-file.mjs .\examples\huahai-qingke.txt .\huahai-result.json
```

## 无弹窗自动化

服务端默认给每个 turn 发送：

```json
{
  "approvalPolicy": "never",
  "sandboxPolicy": {
    "type": "readOnly",
    "networkAccess": true
  }
}
```

这意味着自动化任务不会卡在审批弹窗上。需要写文件时，用：

```powershell
.\start.ps1 -WorkspaceWrite
```

这会把 `CODEX_CWD` 作为可写根目录。

## 环境变量

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `PORT` | `3000` | HTTP 端口 |
| `HOST` | `127.0.0.1` | HTTP host |
| `CODEX_BIN` | 自动探测 | 可调用的 `codex.exe` |
| `CODEX_CWD` | 项目目录 | Codex turn 工作目录 |
| `CODEX_MODEL` | 空 | 可选模型覆盖 |
| `CODEX_SKILL_NAME` | 空 | 默认注入 skill 名称 |
| `CODEX_SKILL_PATH` | 空 | 默认注入 skill 路径 |
| `CODEX_APPROVAL_POLICY` | `never` | 审批策略 |
| `CODEX_SANDBOX` | `readOnly` | `readOnly` / `workspaceWrite` / `dangerFullAccess` |
| `CODEX_TIMEOUT_MS` | `300000` | 单次请求超时 |
| `WEB_TO_CODEX_TOKEN` | 空 | 可选 Bearer token |

长产业链报告建议：

```powershell
$env:CODEX_TIMEOUT_MS = "600000"
```

## API

```http
POST /api/codex
Content-Type: application/json
```

```json
{
  "prompt": "请分析华海清科",
  "skillName": "stock-reverse-engineering",
  "skillPath": "C:\\Users\\<你>\\.codex\\skills\\stock-reverse-engineering\\SKILL.md"
}
```

返回：

```json
{
  "answer": "..."
}
```

## 目录

```text
.
├─ server.mjs                 # HTTP + Codex app-server bridge
├─ start.ps1                  # Windows 一键启动和 Codex binary 自动探测
├─ public/index.html          # 简单网页 UI
├─ tools/post-prompt-file.mjs # 从文件发送 prompt
├─ tools/smoke-test.mjs       # API smoke test
├─ examples/                  # 示例股票 prompt
├─ docs/runbook.md            # 运维说明
├─ spec/mvp.md                # MVP 验收标准
└─ AGENTS.md                  # Codex/agent 项目说明
```

## 已知限制

- 当前 HTTP 接口是完整回答后一次性返回，不是流式。长报告需要等 `turn/completed`。
- Codex App Server 协议可能随 Codex 版本变化，本项目按本地 `codex-cli 0.142.5` 验证过。
- 如果另一台电脑首次运行找不到 `codex.exe`，先打开一次 Codex App，或手动设置 `CODEX_BIN`。
- 这是研究工具，不构成投资建议。
