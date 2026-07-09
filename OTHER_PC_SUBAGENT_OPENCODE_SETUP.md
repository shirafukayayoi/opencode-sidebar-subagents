# サブエージェント表示付き OpenCode を別PCへ入れる手順

この手順書は、別PCの Codex などに渡して、このPCと同じ「右サイドバーにサブエージェント状況が出る OpenCode」を使える状態にするためのものです。

## 目的

- `shirafukayayoi/opencode-sidebar-subagents` の `dev` ブランチを使う。
- OpenCode TUI の右サイドバーにサブエージェント状況を表示する。
- このPCで使っている global OpenCode plugin と quota 表示設定を移行する。
- skills の探索先は `G:\マイドライブ\I am you\Skills` だけにする。
- 起動が極端に遅くなっていないことを確認する。

## 対象リポジトリ

```text
https://github.com/shirafukayayoi/opencode-sidebar-subagents
branch: dev
```

この fork には少なくとも次の変更が入っていることを確認する。

```powershell
git log --oneline -5
```

期待するコミット:

```text
7460780b7 Show subagents in session sidebar
```

## 前提

別PCで以下が使えること。

- PowerShell
- Git
- Node.js / npm
- Bun

Bun はこの repo の `package.json` に合わせて `1.3.14` が望ましい。

```powershell
bun --version
node --version
npm --version
git --version
```

## 1. fork を clone する

作業場所は任意。以下は例。

```powershell
$work = Join-Path $env:USERPROFILE "Documents\opencode-sidebar-subagents"
git clone https://github.com/shirafukayayoi/opencode-sidebar-subagents.git $work
Set-Location -LiteralPath $work
git checkout dev
git pull --ff-only
```

## 2. 依存関係を入れる

```powershell
Set-Location -LiteralPath $work
bun install
```

`postinstall` で `node-pty` などを調整するので、失敗した場合はログを確認してから再実行する。

## 3. OpenCode をビルドする

TUI 用に早く確認する場合は `--skip-embed-web-ui` を付ける。

```powershell
bun run --cwd packages/opencode build --single --skip-embed-web-ui
```

Web UI 埋め込みも含めた完全ビルドにしたい場合は次を使う。ただし時間は長くなる。

```powershell
bun run --cwd packages/opencode build --single
```

Windows x64 なら出力先は通常ここ。

```text
packages\opencode\dist\opencode-windows-x64\bin\opencode.exe
```

確認:

```powershell
$built = Join-Path $work "packages\opencode\dist\opencode-windows-x64\bin\opencode.exe"
& $built --version
```

## 4. global opencode を差し替える

既存の global `opencode` をバックアップしてから、ビルド済み exe に差し替える。

```powershell
$built = Join-Path $work "packages\opencode\dist\opencode-windows-x64\bin\opencode.exe"
$cmd = Get-Command opencode -ErrorAction Stop
$target = $cmd.Source
$stamp = Get-Date -Format "yyyyMMdd-HHmmss"

Copy-Item -LiteralPath $target -Destination "$target.bak-$stamp" -Force
Copy-Item -LiteralPath $built -Destination $target -Force

opencode --version
```

`Get-Command opencode` が `.cmd` や shim を返す場合は、npm global package の実体を確認する。

```powershell
npm root -g
Get-Command opencode | Format-List *
```

このPCでは実体は次のような場所だった。

```text
C:\Users\<user>\AppData\Roaming\npm\node_modules\opencode-ai\bin\opencode.exe
```

## 5. global OpenCode 設定を作る

既存設定は消さずにバックアップする。

```powershell
$config = Join-Path $env:USERPROFILE ".config\opencode"
if (Test-Path -LiteralPath $config) {
  $backup = Join-Path $env:USERPROFILE (".config\opencode.backup-" + (Get-Date -Format "yyyyMMdd-HHmmss"))
  Move-Item -LiteralPath $config -Destination $backup
}
New-Item -ItemType Directory -Force -Path $config | Out-Null
```

## 6. plugin を移行する

このPCで使っている plugin は以下。

```text
opencode-goal-plugin
@frankhommers/opencode-yolo
@slkiser/opencode-quota
@franlol/opencode-md-table-formatter@latest
```

別PCでは global config に入れる。

```powershell
opencode plugin -g opencode-goal-plugin
opencode plugin -g @frankhommers/opencode-yolo
opencode plugin -g @slkiser/opencode-quota
opencode plugin -g @franlol/opencode-md-table-formatter@latest
```

失敗した plugin だけ `-f` を付けて再実行する。

```powershell
opencode plugin -g -f @slkiser/opencode-quota
```

## 7. opencode.jsonc を設定する

`%USERPROFILE%\.config\opencode\opencode.jsonc` を作る。既存の provider / MCP / agent 設定がある場合は、それを残して下記の要点をマージする。

最小設定例:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "instructions": [
    "AGENTS.md"
  ],
  "skills": {
    "paths": [
      "G:\\マイドライブ\\I am you\\Skills"
    ]
  },
  "permission": {
    "skill": {
      "*": "allow",
      "customize-opencode": "deny"
    }
  },
  "plugin": [
    "opencode-goal-plugin",
    "@frankhommers/opencode-yolo",
    "@slkiser/opencode-quota",
    "@franlol/opencode-md-table-formatter@latest"
  ],
  "agent": {
    "general": { "mode": "subagent" },
    "explore": { "mode": "subagent" },
    "gpt-5.4": {
      "model": "openai/gpt-5.4",
      "description": "OpenAI GPT-5.4",
      "mode": "subagent",
      "hidden": true
    },
    "gpt-5.4-mini": {
      "model": "openai/gpt-5.4-mini",
      "description": "OpenAI GPT-5.4 Mini",
      "mode": "subagent",
      "hidden": true
    },
    "deepseek-v4-flash": {
      "model": "deepseek/deepseek-v4-flash",
      "description": "DeepSeek V4 Flash",
      "mode": "subagent",
      "hidden": true
    },
    "deepseek-v4-pro": {
      "model": "deepseek/deepseek-v4-pro",
      "description": "DeepSeek V4 Pro",
      "mode": "subagent",
      "hidden": true
    },
    "glm-5.2": {
      "model": "zai_coding/glm-5.2",
      "description": "GLM 5.2 (Z.AI)",
      "mode": "subagent",
      "hidden": true
    }
  }
}
```

注意:

- provider の API key、OAuth、MCP 認証情報はコピーしない。別PCで再ログインする。
- `G:\マイドライブ\I am you\Skills` が存在しないPCでは、Google Drive のマウント先を同じにするか、このパスだけ移行先に合わせて変更する。

## 8. AGENTS.md を作る

`%USERPROFILE%\.config\opencode\AGENTS.md` に最低限これを入れる。

```markdown
# Development Rules

## Language

Use Japanese for all communication with the user, including explanations, progress reports, summaries, and code-related comments, unless the user explicitly requests another language.

## Sub-agents

When using sub-agents, always use GPT-5.4 or GPT-5.4-mini.
Prefer GPT-5.4 for complex, ambiguous, or high-impact tasks.
Prefer GPT-5.4-mini for simple, low-risk, or lightweight tasks.
```

## 9. quota plugin の右パネルをバー表示にする

quota 設定ファイルを作成する。

```powershell
$quotaDir = Join-Path $env:USERPROFILE ".config\opencode\opencode-quota"
New-Item -ItemType Directory -Force -Path $quotaDir | Out-Null
@'
{
  "enableToast": true,
  "showSessionTokens": false,
  "enabledProviders": ["copilot", "openai", "zai", "deepseek"],
  "formatStyle": "allWindows",
  "percentDisplayMode": "remaining",
  "tuiSidebarPanel": {
    "enabled": true,
    "formatStyle": "allWindows"
  },
  "tuiCompactStatus": {
    "enabled": true,
    "homeBottom": true,
    "sessionPrompt": true,
    "suppressWhenNativeProviderQuota": false
  },
  "maintainerAnnouncements": {
    "enabled": false,
    "home": false
  }
}
'@ | Set-Content -LiteralPath (Join-Path $quotaDir "quota-toast.json") -Encoding UTF8
```

もし右パネルが `Copilot 100%` のような短い1行表示のままなら、quota plugin の TUI ファイルを確認する。

```powershell
$quotaTuiFiles = @(
  "$env:USERPROFILE\.cache\opencode\node_modules\@slkiser\opencode-quota\dist\tui.tsx",
  "$env:USERPROFILE\.config\opencode\node_modules\@slkiser\opencode-quota\dist\tui.tsx"
) | Where-Object { Test-Path -LiteralPath $_ }

$quotaTuiFiles
```

対象ファイルの `SidebarContentView` で、詳細行がある場合は `getSidebarPanelLinesExpanded(panel())` を表示するようにする。このPCでは cache と config の両方を同じ内容にした。

重要な変更点:

```ts
const [collapsed, setCollapsed] = createSignal(false);

const displayLines = () => {
  if (!hasDetailLines()) return lines();
  return getSidebarPanelLinesExpanded(panel());
};

const toggleIcon = () => "▼";
```

## 10. 起動速度を確認する

```powershell
$sw=[Diagnostics.Stopwatch]::StartNew()
opencode debug startup
$sw.Stop()
"measured_ms=$([math]::Round($sw.Elapsed.TotalMilliseconds,0))"
```

目安:

- 1から2秒程度なら問題なし。
- 5秒以上かかる場合は plugin を1つずつ外して原因を確認する。

## 11. 表示確認

OpenCode を完全に終了して起動し直す。

```powershell
opencode
```

確認すること:

- 右サイドバーに `Quota` が出る。
- quota は `Copilot 100%` のような短い1行だけではなく、バー付きの詳細表示になる。
- サブエージェントを使うタスクを投げると、右サイドバーに `Subagents` が出る。
- `MCP` はサイドバー下側に寄る。
- `LSP` は右サイドバーに出ない。
- MCP サーバーが多すぎる場合、MCP 枠は非表示になって quota / subagents / todo を圧迫しない。

サブエージェント表示の確認用プロンプト例:

```text
3つの軽い調査タスクに分けて、サブエージェントで並行実行して。結果は短くまとめて。
```

## 12. 戻し方

ビルド済み exe に差し替えて問題が出た場合は、手順4で作ったバックアップに戻す。

```powershell
$cmd = Get-Command opencode -ErrorAction Stop
$target = $cmd.Source
Get-ChildItem -LiteralPath (Split-Path $target) -Filter "opencode.exe.bak-*" | Sort-Object LastWriteTime -Descending | Select-Object -First 5
Copy-Item -LiteralPath "<backup-path>" -Destination $target -Force
opencode --version
```

設定だけ戻す場合は、手順5で作った `opencode.backup-*` を戻す。

## 13. 作業完了条件

別PCの Codex は、完了前に次をユーザーへ報告する。

- clone した repo と branch
- `opencode --version`
- `opencode debug startup` の実測時間
- plugin 一覧
- skills path が `G:\マイドライブ\I am you\Skills` だけになっていること
- 右サイドバーで quota バー表示と subagents 表示を確認したか
