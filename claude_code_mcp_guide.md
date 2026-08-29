# Claude Code：MCPサーバー設定の構成（中身）についての解説

> 対象: Claude Code の MCP（Model Context Protocol）  
> 更新基準: 2026-08-28 時点のAnthropic公式ドキュメント  
> 目的: MCPサーバーをどのスコープにどう定義し、権限・認証・セキュリティをどう扱うかを、設計意図も含めて理解する

---

<!-- toc -->
# 目次

- [1. MCPとは何か](#1-mcpとは何か)
- [2. 追加の基本形](#2-追加の基本形)
- [3. インストールスコープ](#3-インストールスコープ)
- [4. Local scopeの保存形](#4-local-scopeの保存形)
- [5. .mcp.json（Project scope）](#5-mcpjsonproject-scope)
- [6. Transportの選び方](#6-transportの選び方)
- [7. 環境変数の展開](#7-環境変数の展開)
- [8. 認証：OAuth 2.0](#8-認証oauth-20)
- [9. 認証：Bearer token](#9-認証bearer-token)
- [10. 認証：headersHelper（動的ヘッダ）](#10-認証headershelper動的ヘッダ)
- [11. 管理コマンド](#11-管理コマンド)
- [12. サーバーの状態表示](#12-サーバーの状態表示)
- [13. Toolの命名規則](#13-toolの命名規則)
- [14. permissionsでMCP toolを制御する](#14-permissionsでmcp-toolを制御する)
- [15. HookでMCP toolを捕まえる](#15-hookでmcp-toolを捕まえる)
- [16. Subagentへスコープする](#16-subagentへスコープする)
- [17. Plugin提供のMCPサーバー](#17-plugin提供のmcpサーバー)
- [18. claude.aiのコネクタ](#18-claudeaiのコネクタ)
- [19. タイムアウトと出力制限](#19-タイムアウトと出力制限)
- [20. Tool searchと自動バックグラウンド](#20-tool-searchと自動バックグラウンド)
- [21. セキュリティ：prompt injection](#21-セキュリティprompt-injection)
- [22. Toolごとにユーザー確認を必須にする](#22-toolごとにユーザー確認を必須にする)
- [23. サーバーの無効化](#23-サーバーの無効化)
- [24. Project scopeの承認とworkspace trust](#24-project-scopeの承認とworkspace-trust)
- [25. Claude Code自体をMCPサーバーにする](#25-claude-code自体をmcpサーバーにする)
- [26. 他形式からの移行](#26-他形式からの移行)
- [27. 何をMCPにするか](#27-何をmcpにするか)
- [28. トラブルシュート](#28-トラブルシュート)
- [29. 最終チェックリスト](#29-最終チェックリスト)
- [30. まとめ](#30-まとめ)
- [参考資料](#参考資料)

<!-- /toc -->

---

# 1. MCPとは何か

MCP（Model Context Protocol）は、**Claudeに外部のツールとデータソースを提供するための共通プロトコル**です。

```text
Claude Code
     │
     ├── 組み込みTool（Read / Bash / Edit ...）
     │
     └── MCPサーバー
            │
            ├── GitHub
            ├── PostgreSQL
            ├── Notion
            └── 社内API
```

MCPサーバーを追加すると、そのサーバーが提供するtoolがClaudeから呼べるようになります。

```text
組み込みTool
    ↓
Claude Codeが持っている機能

MCP tool
    ↓
外部プロセス・外部サービスが提供する機能
```

---

# 2. 追加の基本形

CLIから追加します。

```bash
# HTTPサーバー（リモートは基本これ）
claude mcp add --transport http <name> <url>

# stdioサーバー（ローカルプロセス）
claude mcp add [options] <name> -- <command> [args...]

# SSEサーバー（非推奨。HTTPを使う）
claude mcp add --transport sse <name> <url>

# WebSocketサーバー（JSON指定のみ）
claude mcp add-json <name> '{"type":"ws","url":"wss://..."}'
```

実例:

```bash
# GitHub
claude mcp add --transport http github https://api.githubcopilot.com/mcp/ \
  --header "Authorization: Bearer YOUR_GITHUB_PAT"

# PostgreSQL（読み取り専用ユーザーで）
claude mcp add --transport stdio db -- npx -y @bytebase/dbhub \
  --dsn "postgresql://readonly:pass@prod.db.com:5432/analytics"

# Notion
claude mcp add --transport http notion https://mcp.notion.com/mcp
```

---

# 3. インストールスコープ

3つあります。

| スコープ | 読み込まれる範囲 | 共有 | 保存先 |
|---|---|---|---|
| Local（既定） | そのプロジェクトのみ | × | `~/.claude.json` |
| Project | そのプロジェクトのみ | ○（`.mcp.json`） | プロジェクトルートの `.mcp.json` |
| User | 自分の全プロジェクト | × | `~/.claude.json` |

指定方法:

```bash
claude mcp add --transport http stripe --scope local   https://mcp.stripe.com
claude mcp add --transport http shared --scope project https://example.com/mcp
claude mcp add --transport http hubspot --scope user   https://mcp.hubspot.com/anthropic
```

使い分け:

```text
自分だけ・このプロジェクトだけ試す
    ↓
local

チーム全員に使ってほしい
    ↓
project（.mcp.json をコミット）

自分がどこでも使う
    ↓
user
```

---

# 4. Local scopeの保存形

`~/.claude.json` にプロジェクト単位で入ります。

```json
{
  "projects": {
    "/path/to/your/project": {
      "mcpServers": {
        "stripe": {
          "type": "http",
          "url": "https://mcp.stripe.com"
        }
      }
    }
  }
}
```

自分専用なので、認証情報を含んでもリポジトリには入りません。

---

# 5. `.mcp.json`（Project scope）

プロジェクトルートに置き、コミットして共有します。

```json
{
  "mcpServers": {
    "server-name": {
      "type": "http",
      "url": "https://...",
      "command": "...",
      "args": ["--flag"],
      "env": { "KEY": "value" },
      "headers": {
        "Authorization": "Bearer token"
      },
      "timeout": 600000,
      "alwaysLoad": false
    }
  }
}
```

主なフィールド:

```text
type       … http / sse / ws / stdio
url        … http / sse / ws のとき
command    … stdio のとき
args       … stdio の引数
env        … 環境変数
headers    … HTTPヘッダ
timeout    … tool呼び出しごとのタイムアウト（ミリ秒）
alwaysLoad … 使わなくてもロードするか
```

重要な制約として、

```text
project scopeのサーバーは
対話セッションで承認してから使える
```

という点があります。

---

# 6. Transportの選び方

## HTTP（推奨）

```bash
claude mcp add --transport http secure-api https://api.example.com/mcp \
  --header "Authorization: Bearer your-token"
```

JSONでは `streamable-http` が `http` の別名です（MCP仕様側の名前）。

## SSE（非推奨）

```bash
claude mcp add --transport sse asana https://mcp.asana.com/sse \
  --header "X-API-Key: your-key-here"
```

HTTPが使えるならHTTPを使います。

## stdio（ローカルプロセス）

```bash
claude mcp add --env AIRTABLE_API_KEY=YOUR_KEY --transport stdio airtable \
  -- npx -y airtable-mcp-server
```

`--` が**必須**です。

```text
claude mcp add [options] <name> -- <command> [args...]
                                 ↑
                        ここから先はそのままサーバーへ渡る
```

`--` を忘れると、サーバー側の引数がClaude Codeのオプションとして解釈されます。

stdioサーバーの環境には `CLAUDE_PROJECT_DIR` がプロジェクトルートとしてセットされます。

## WebSocket

```bash
claude mcp add-json events-server \
  '{"type":"ws","url":"wss://mcp.example.com/socket","headers":{"Authorization":"Bearer YOUR_TOKEN"}}'
```

双方向の持続接続が必要な場合だけ使います。リクエスト・レスポンス型ならHTTPで十分です。

---

# 7. 環境変数の展開

`.mcp.json` の中で環境変数を参照できます。

```json
{
  "mcpServers": {
    "api-server": {
      "type": "http",
      "url": "${API_BASE_URL:-https://api.example.com}/mcp",
      "headers": {
        "Authorization": "Bearer ${API_KEY}"
      },
      "env": {
        "DEBUG": "${DEBUG:-false}"
      }
    }
  }
}
```

構文:

```text
${VAR}            … 環境変数へ展開
${VAR:-default}   … 未設定ならデフォルト値
```

展開されるフィールド:

```text
command / args / env / url / headers / headersHelper
```

これにより、

```text
.mcp.json はコミットする
    ↓
トークンは環境変数から渡す
```

という運用ができます。**トークンを `.mcp.json` に直書きしない**のが原則です。

---

# 8. 認証：OAuth 2.0

多くのリモートサーバーはOAuthに対応しています。

セッション内から:

```text
/mcp
    ↓
サーバーを選ぶ
    ↓
ブラウザでログイン
```

CLIから:

```bash
claude mcp login <name>
claude mcp logout <name>
```

コールバックポートを固定したい場合:

```bash
claude mcp add --transport http \
  --callback-port 8080 \
  my-server https://mcp.example.com/mcp
```

クライアント資格情報を事前設定する場合:

```bash
# 対話でシークレットを入力
claude mcp add --transport http \
  --client-id your-client-id --client-secret --callback-port 8080 \
  my-server https://mcp.example.com/mcp

# CI等では環境変数で
MCP_CLIENT_SECRET=your-secret claude mcp add --transport http \
  --client-id your-client-id --client-secret --callback-port 8080 \
  my-server https://mcp.example.com/mcp
```

---

# 9. 認証：Bearer token

固定トークンで済む場合はヘッダで渡します。

```bash
claude mcp add --transport http stripe https://mcp.stripe.com \
  --header "Authorization: Bearer YOUR_TOKEN"
```

`.mcp.json` に書くなら環境変数経由にします。

```json
{
  "mcpServers": {
    "api": {
      "type": "http",
      "url": "https://api.example.com/mcp",
      "headers": {
        "Authorization": "Bearer ${API_KEY}"
      }
    }
  }
}
```

---

# 10. 認証：`headersHelper`（動的ヘッダ）

短命トークンや社内独自の認証方式に対応するための仕組みです。

```json
{
  "mcpServers": {
    "internal-api": {
      "type": "http",
      "url": "https://mcp.internal.example.com",
      "headersHelper": "/opt/bin/get-mcp-auth-headers.sh"
    }
  }
}
```

ヘルパースクリプトはJSONを標準出力へ返します。

```bash
#!/bin/bash
echo '{"Authorization": "Bearer '$(get-token)'"}'
```

ヘルパーが使える環境変数:

```text
CLAUDE_CODE_MCP_SERVER_NAME
CLAUDE_CODE_MCP_SERVER_URL
CLAUDE_PLUGIN_ROOT（Plugin由来のとき）
```

```text
トークンが数分で期限切れになる
    ↓
静的ヘッダでは運用できない
    ↓
headersHelper で毎回取得する
```

---

# 11. 管理コマンド

```bash
# 一覧
claude mcp list

# 詳細
claude mcp get <name>

# 削除
claude mcp remove <name> [--scope local|project|user]

# Claude Desktopから取り込む
claude mcp add-from-claude-desktop

# JSONで追加
claude mcp add-json <name> '<json>'

# project scopeサーバーの承認状態をリセット
claude mcp reset-project-choices

# Claude Code自体をMCPサーバーとして公開
claude mcp serve
```

セッション内では:

```text
/mcp
    ↓
サーバーの確認・認証・再接続・無効化

/reload-plugins
    ↓
Plugin由来のサーバーを再接続
```

---

# 12. サーバーの状態表示

`claude mcp list` の表示です。

```text
✔ Connected                  … 正常
! Needs authentication       … OAuthサインインが必要
✘ Failed to connect          … 接続失敗（詳細付き）
⏸ Pending approval           … .mcp.json のサーバーが承認待ち
⊘ Disabled for this project  … /mcp で無効化されている
cached                       … tool一覧をキャッシュから読み込み
```

「toolが見えない」ときは、まずここを確認します。

---

# 13. Toolの命名規則

MCP toolの名前は決まった形式になります。

```text
mcp__<server>__<tool>
```

例:

```text
mcp__memory__create_entities
mcp__filesystem__read_file
mcp__github__search_repositories
```

Plugin同梱のサーバーはスコープ付きになります。

```text
mcp__plugin_<plugin-name>_<server-name>__<tool-name>
```

例:

```text
mcp__plugin_my-plugin_database-tools__query
```

設定内でこのサーバーを参照するときの名前は:

```text
plugin:my-plugin:database-tools
```

claude.aiのコネクタ由来のものは:

```text
mcp__claude_ai_<server>__<tool>
```

この命名規則は、permissionルールとHookのmatcherの両方で使うので重要です。

---

# 14. permissionsでMCP toolを制御する

```json
{
  "permissions": {
    "allow": [
      "mcp__github__get_*",
      "mcp__puppeteer__puppeteer_navigate"
    ],
    "deny": [
      "mcp__github__delete_*"
    ]
  }
}
```

書き方:

```text
mcp__puppeteer
    ↓
puppeteerサーバーの全tool

mcp__puppeteer__*
    ↓
同上

mcp__puppeteer__puppeteer_navigate
    ↓
特定のtoolだけ
```

allowルールのグロブには制約があります。

```text
mcp__<server>__ のリテラル接頭辞が必要
サーバー部分にグロブは使えない

○ mcp__puppeteer__*
○ mcp__github__get_*
× "*"
× "mcp__*"
```

アンカーの無いallowグロブは警告付きでスキップされ、何も自動承認しません。

---

# 15. HookでMCP toolを捕まえる

matcherは正規表現として扱われます。

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "mcp__.*__(delete|drop|remove).*",
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/confirm-destructive.sh",
            "args": []
          }
        ]
      }
    ]
  }
}
```

```text
サーバー単位
    ↓
mcp__memory__.*

操作の種類単位
    ↓
mcp__.*__write.*
```

MCPサーバーは外部システムを触るため、**破壊的なtoolをHookで押さえる**のは有効な防御です。

---

# 16. Subagentへスコープする

重いMCPサーバーは、メイン会話ではなくSubagentに持たせられます。

```yaml
---
name: browser-tester
description: Tests features in a real browser using Playwright
mcpServers:
  - playwright:
      type: stdio
      command: npx
      args: ["-y", "@playwright/mcp@latest"]
  - github
---

Use the Playwright tools to navigate, screenshot, and interact with pages.
```

```text
メイン会話に常時接続
    ↓
tool listingがcontextを圧迫

必要なSubagentだけに持たせる
    ↓
メイン側は軽いまま
```

toolが多いサーバーほど効果があります。

---

# 17. Plugin提供のMCPサーバー

Pluginは `.mcp.json` を同梱できます。

```json
{
  "mcpServers": {
    "database-tools": {
      "command": "${CLAUDE_PLUGIN_ROOT}/servers/db-server",
      "args": ["--config", "${CLAUDE_PLUGIN_ROOT}/config.json"],
      "env": {
        "DB_URL": "${DB_URL}"
      }
    }
  }
}
```

使える変数:

```text
${CLAUDE_PLUGIN_ROOT}   … Pluginのインストールディレクトリ
${CLAUDE_PLUGIN_DATA}   … 更新をまたぐ永続データディレクトリ
${CLAUDE_PROJECT_DIR}   … プロジェクトルート
```

置き場所は**Pluginルート直下の `.mcp.json`**です。`.claude-plugin/` の中ではありません。

---

# 18. claude.aiのコネクタ

claude.aiアカウントでログインしていると、claude.ai側で設定したコネクタが自動的に現れます。

全部止める:

```bash
ENABLE_CLAUDEAI_MCP_SERVERS=false claude
```

設定で止める:

```json
{
  "disableClaudeAiConnectors": true
}
```

特定のものだけ止める:

```json
{
  "deniedMcpServers": ["claude.ai Slack"]
}
```

「意図しないサーバーが繋がっている」ときは、まずこれを疑います。

---

# 19. タイムアウトと出力制限

```bash
# 起動時のグローバルタイムアウト
MCP_TIMEOUT=10000 claude

# tool出力のトークン上限（既定 25,000）
MAX_MCP_OUTPUT_TOKENS=50000 claude

# アイドルタイムアウト（既定 HTTP/SSE 5分、stdio 30分）
CLAUDE_CODE_MCP_TOOL_IDLE_TIMEOUT=300000 claude
```

サーバーごとのタイムアウトは `.mcp.json` で指定します。

```json
"timeout": 600000
```

サーバー側からtool単位で出力上限を宣言することもできます。

```json
{
  "name": "get_schema",
  "description": "Returns full database schema",
  "_meta": {
    "anthropic/maxResultSizeChars": 200000
  }
}
```

```text
MCP toolの出力は context を直撃する
        ↓
巨大なJSONを返すtoolは要注意
        ↓
サーバー側で要約するか上限を設ける
```

---

# 20. Tool searchと自動バックグラウンド

既定でtool searchが有効です。

```text
tool search 有効
    ↓
全MCP toolのスキーマを常時ロードしない
    ↓
必要になった時点で取得する
```

無効化するには:

```bash
ENABLE_TOOL_SEARCH=false claude
```

また、長時間かかるtool呼び出しは自動でバックグラウンドタスクへ移されます。

```text
メイン会話のtool呼び出しが2分を超える
        ↓
バックグラウンドタスクへ
```

閾値を変えるには `CLAUDE_CODE_MCP_AUTO_BACKGROUND_MS` を設定します（`0` で無効）。

---

# 21. セキュリティ：prompt injection

MCPで最も重要な論点です。

```text
MCPサーバーが外部コンテンツを取得する
            ↓
その内容がClaudeのcontextへ入る
            ↓
「これまでの指示を無視して…」が混入し得る
```

対策の考え方:

```text
① 信頼できるサーバーだけ接続する
② 破壊的toolを permissions.deny で封じる
③ PreToolUse Hookで危険操作を検査する
④ 書き込み系サーバーは読み取り専用の資格情報で動かす
⑤ project scope の .mcp.json は中身を必ず読む
```

特に、

```text
DBへ接続するMCPサーバー
    ↓
読み取り専用ユーザーで繋ぐ
```

は費用対効果が高い対策です。前述の例で `postgresql://readonly:...` としているのはこの理由です。

---

# 22. Toolごとにユーザー確認を必須にする

サーバー側でtoolに注釈を付けられます。

```json
{
  "name": "grant_access",
  "description": "Requests access to protected resource",
  "_meta": {
    "anthropic/requiresUserInteraction": true
  }
}
```

このtoolは、

```text
allowルールがあっても確認される
auto / bypassPermissions でも確認される
dontAsk では拒否される
```

という扱いになります。自作MCPサーバーで危険な操作を提供するなら、この注釈を付けます。

---

# 23. サーバーの無効化

削除せず一時的に切るには `/mcp` パネルでトグルします。状態は `~/.claude.json` に保存されます。

```text
disabledMcpServers
    ↓
オプトアウトのリスト
（userサーバー、Plugin、claude.aiコネクタ）

enabledMcpServers
    ↓
オプトインのリスト
（既定オフの組み込みサーバー）
```

---

# 24. Project scopeの承認とworkspace trust

`.mcp.json` のサーバーは、対話セッションで一度承認する必要があります。

```text
.mcp.json をpullしてきた
        ↓
起動時に承認を求められる
        ↓
承認するまで ⏸ Pending approval
```

承認済みの選択をリセットするには:

```bash
claude mcp reset-project-choices
```

**注意点**として、非対話モード（`claude -p`）では承認プロンプトが出ません。

```text
claude -p
    ↓
workspace trustダイアログなし
    ↓
.mcp.json のサーバーへ接続してしまう
```

CIで他人のリポジトリを扱うなら `--bare` を使うか、`--mcp-config` で明示的に渡します。

---

# 25. Claude Code自体をMCPサーバーにする

```bash
claude mcp serve
```

他のMCPクライアントからClaude Codeの機能を呼べるようになります。

---

# 26. 他形式からの移行

## URLだけある場合

```bash
claude mcp add --transport http example https://mcp.example.com/mcp
```

## 起動コマンドだけある場合

```bash
claude mcp add example --env API_KEY=your-key -- npx -y @example/mcp-server
```

## Claude Desktop形式のJSONブロックがある場合

```bash
claude mcp add-json example '{"command":"npx","args":["-y","@example/mcp-server"]}'
```

移植時の修正点:

```text
url があるのに type が無い
    ↓
"type": "http" | "sse" | "ws" を追加

キーに英数字・- ・_ 以外が含まれる
    ↓
リネームする
```

---

# 27. 何をMCPにするか

設計上の判断基準です。

MCPに向いているもの:

```text
外部サービスのAPI
    ↓
GitHub、Notion、Slack、Jira

構造化されたデータソース
    ↓
DB、検索インデックス

専用のランタイム操作
    ↓
ブラウザ自動化、デザインツール
```

MCPにしなくてよいもの:

```text
CLIで叩けるもの
    ↓
gh、aws、kubectl は Bash で足りる

プロジェクト固有の定型手順
    ↓
Skill で足りる

決定論的なチェック
    ↓
Hook + スクリプトで足りる
```

```text
Bashで1行で済むことをMCP化しない
```

というのが実務上の目安です。MCPサーバーはtool listingを消費し、接続失敗の可能性も増やします。

---

# 28. トラブルシュート

```text
toolが見えない
    ↓
claude mcp list で状態を確認
    ↓
Pending approval なら承認
    ↓
Needs authentication なら /mcp でログイン

接続に失敗する
    ↓
claude mcp get <name> で設定を確認
    ↓
stdioなら -- の位置を確認
    ↓
環境変数が展開されているか確認

toolはあるが呼ばれない
    ↓
permissions.deny に入っていないか
    ↓
descriptionが不明瞭でないか

出力が途中で切れる
    ↓
MAX_MCP_OUTPUT_TOKENS
    ↓
サーバー側の maxResultSizeChars

呼び出しがタイムアウトする
    ↓
.mcp.json の timeout
    ↓
MCP_TIMEOUT
```

`claude --debug` で起動すると、接続時の詳細が確認できます。

---

# 29. 最終チェックリスト

```text
[ ] スコープの選択が正しい（共有すべきものがlocalに入っていない）
[ ] .mcp.json にトークンを直書きしていない
[ ] 環境変数展開を使っている
[ ] stdioサーバーで -- を書いている
[ ] DBサーバーは読み取り専用の資格情報で繋いでいる
[ ] 破壊的toolを permissions.deny または Hook で押さえた
[ ] 信頼できないサーバーを接続していない
[ ] 出力が巨大なtoolの上限を確認した
[ ] メイン会話に重いサーバーを常駐させていない
[ ] Subagentへスコープすべきものを検討した
[ ] CIでは --bare または --mcp-config を使っている
[ ] claude mcp list で全サーバーが Connected になっている
```

---

# 30. まとめ

MCP設定の構成を最も簡潔にまとめると、

```text
MCPサーバー定義
│
├─ スコープ
│    ↓
│  local（自分・このrepo）
│  project（.mcp.json・共有）
│  user（自分・全repo）
│
├─ transport
│    ↓
│  http（推奨） / stdio / sse（非推奨） / ws
│
├─ 認証
│    ↓
│  OAuth / Bearer header / headersHelper
│
├─ 制限
│    ↓
│  timeout / 出力トークン上限
│
└─ 権限
     ↓
   permissions（mcp__server__tool）
   Hooks（matcher で捕捉）
   Subagentへのスコープ
```

そしてClaude Code全体での位置づけは、

```text
組み込みTool
=
ファイル操作・シェル・検索

MCP
=
外部サービス・外部データへの接続

Skill
=
それらをどう使うかの手順

CLAUDE.md / rules
=
使い方の方針

permissions / Hooks
=
使わせない・必ず検査する
```

です。

MCPを増やすときは、

```text
できることが増える
    +
contextを消費する
    +
外部からの入力経路が増える
```

の3つが同時に起きる点を意識します。**接続するサーバーは、必要なものだけを、必要なスコープで、最小の権限で**というのが基本方針です。

---

# 参考資料

Anthropic公式 Claude Code Docs:

- Connect Claude Code to tools via MCP  
  https://code.claude.com/docs/en/mcp

- Configure permissions  
  https://code.claude.com/docs/en/permissions

- Create custom subagents  
  https://code.claude.com/docs/en/sub-agents

- Plugins reference  
  https://code.claude.com/docs/en/plugins-reference

特に確認するとよい項目:

- MCP installation scopes
- Server configuration formats
- Transport types
- Authentication
- Environment variable expansion in `.mcp.json`
- Tool availability
- Plugin-provided MCP servers
- Timeouts and output limits
- Permission and security
