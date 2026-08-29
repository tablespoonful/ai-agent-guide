# Claude Code：Pluginの構成（中身）についての解説

> 対象: Claude Code の Plugins  
> 更新基準: 2026-08-28 時点のAnthropic公式ドキュメント  
> 目的: Plugin（`plugin.json` とディレクトリ構成）を「何を・どこに置けばよいか」を、設計意図も含めて理解する

---

<!-- toc -->
# 目次

- [1. Pluginとは何か](#1-pluginとは何か)
- [2. 最小構成](#2-最小構成)
- [3. 最大の落とし穴：.claude-plugin/ の中身](#3-最大の落とし穴claude-plugin-の中身)
- [4. plugin.json のスキーマ](#4-pluginjson-のスキーマ)
- [5. ディレクトリ構造](#5-ディレクトリ構造)
- [6. Path Behavior Rules（重要）](#6-path-behavior-rules重要)
- [7. 環境変数](#7-環境変数)
- [8. userConfig ― 利用者に入力させる値](#8-userconfig--利用者に入力させる値)
- [9. Skillの名前空間](#9-skillの名前空間)
- [10. Skillsディレクトリ内でPluginを開発する](#10-skillsディレクトリ内でpluginを開発する)
- [11. ローカルでのテスト](#11-ローカルでのテスト)
- [12. コンポーネントの動作確認](#12-コンポーネントの動作確認)
- [13. hooks/hooks.json](#13-hookshooksjson)
- [14. .lsp.json ― LSPサーバー](#14-lspjson--lspサーバー)
- [15. monitors/monitors.json ― バックグラウンドモニタ](#15-monitorsmonitorsjson--バックグラウンドモニタ)
- [16. settings.json ― Pluginの既定設定](#16-settingsjson--pluginの既定設定)
- [17. bin/ ― 実行ファイル](#17-bin--実行ファイル)
- [18. バージョン管理と defaultEnabled](#18-バージョン管理と-defaultenabled)
- [19. dependencies ― Plugin間の依存](#19-dependencies--plugin間の依存)
- [20. Node.js依存の自動インストール](#20-nodejs依存の自動インストール)
- [21. キャッシュと更新](#21-キャッシュと更新)
- [22. 検証：claude plugin validate](#22-検証claude-plugin-validate)
- [23. 既存の .claude/ からの移行](#23-既存の-claude-からの移行)
- [24. マーケットプレイス](#24-マーケットプレイス)
- [25. デバッグ](#25-デバッグ)
- [26. Plugin設計の原則](#26-plugin設計の原則)
- [27. セキュリティ上の注意](#27-セキュリティ上の注意)
- [28. Plugin / standalone の使い分け](#28-plugin--standalone-の使い分け)
- [29. 最終チェックリスト](#29-最終チェックリスト)
- [30. まとめ](#30-まとめ)
- [参考資料](#参考資料)

<!-- /toc -->

---

# 1. Pluginとは何か

Pluginは、**Skill / Subagent / Hook / MCPサーバーなどをひとまとめにして配布できる単位**です。

```text
.claude/ に直接置く（standalone）
        ↓
自分・そのプロジェクトだけ

Plugin
        ↓
チーム・コミュニティへ配布できる
バージョン管理できる
複数プロジェクトで再利用できる
```

Claude Codeを拡張する仕組みは、この2通りに整理されます。

| 方式 | Skill名 | 向いている用途 |
|---|---|---|
| standalone（`.claude/`） | `/hello` | 個人のワークフロー、プロジェクト固有の調整、素早い試作 |
| Plugin | `/plugin-name:hello` | チーム共有、コミュニティ配布、バージョン管理、複数プロジェクトでの再利用 |

推奨の流れは、

```text
.claude/ で素早く作る
        ↓
使い物になったらPluginへ変換
        ↓
配布する
```

です。

---

# 2. 最小構成

Pluginは自分のディレクトリを持ちます。

```text
my-first-plugin/
├── .claude-plugin/
│   └── plugin.json
└── skills/
    └── hello/
        └── SKILL.md
```

`plugin.json`:

```json
{
  "name": "my-first-plugin",
  "description": "A greeting plugin to learn the basics",
  "version": "1.0.0",
  "author": {
    "name": "Your Name"
  }
}
```

`SKILL.md`:

```markdown
---
description: Greet the user with a friendly message
disable-model-invocation: true
---

Greet the user warmly and ask how you can help them today.
```

テスト:

```bash
claude --plugin-dir ./my-first-plugin
```

```text
/my-first-plugin:hello
```

で呼べます。

---

# 3. 最大の落とし穴：`.claude-plugin/` の中身

これが最も多い間違いです。

```text
.claude-plugin/ に入れてよいもの
        ↓
plugin.json だけ

skills/ commands/ agents/ hooks/ .mcp.json
        ↓
すべて「Pluginルート直下」
```

誤り:

```text
my-plugin/
└── .claude-plugin/
    ├── plugin.json
    ├── skills/        ← 読まれない
    └── agents/        ← 読まれない
```

正しい形:

```text
my-plugin/
├── .claude-plugin/
│   └── plugin.json
├── skills/
└── agents/
```

また、ここで言うPluginルートは**そのPlugin自身のディレクトリ**であって、`~/.claude/` ではありません。`~/.claude/.mcp.json` のようなファイルは読まれません。

---

# 4. `plugin.json` のスキーマ

必須は `name` だけです。

```json
{
  "$schema": "https://json.schemastore.org/claude-code-plugin-manifest.json",
  "name": "deployment-tools",
  "displayName": "Deployment Tools",
  "version": "1.2.0",
  "description": "Deployment automation tools",
  "author": {
    "name": "Dev Team",
    "email": "dev@company.com",
    "url": "https://github.com/author"
  },
  "homepage": "https://docs.example.com/plugin",
  "repository": "https://github.com/author/plugin",
  "license": "MIT",
  "keywords": ["deployment", "ci-cd"],
  "defaultEnabled": true,
  "metadata": { "catalogId": "cat-123" }
}
```

主なメタデータフィールド:

| フィールド | 説明 |
|---|---|
| `name` | 一意な識別子（kebab-case）。名前空間にも使われる |
| `displayName` | UI表示名。無ければ `name` |
| `version` | セマンティックバージョン |
| `description` | Pluginマネージャに表示される説明 |
| `author` / `homepage` / `repository` / `license` / `keywords` | 配布用の情報 |
| `defaultEnabled` | ユーザーが未設定のとき有効か（既定 `true`） |
| `metadata` | 自由なデータ。Claude Codeは読まない |

`name` は**Skillの名前空間**になります。

```text
name: my-first-plugin
        ↓
skills/hello/SKILL.md
        ↓
/my-first-plugin:hello
```

---

# 5. ディレクトリ構造

フル構成の例です。

```text
enterprise-plugin/
├── .claude-plugin/
│   └── plugin.json         # マニフェスト（省略可）
├── skills/                 # Skill（<name>/SKILL.md）
│   ├── code-reviewer/
│   │   └── SKILL.md
│   └── pdf-processor/
│       ├── SKILL.md
│       └── scripts/
├── commands/               # フラットな .md 形式のSkill
│   ├── status.md
│   └── logs.md
├── agents/                 # Subagent定義
│   ├── security-reviewer.md
│   └── compliance-checker.md
├── workflows/              # Workflowスクリプト
├── output-styles/          # Output style定義
├── themes/                 # カラーテーマ
├── monitors/
│   └── monitors.json       # バックグラウンドモニタ
├── hooks/
│   └── hooks.json          # Hook設定
├── bin/                    # PATHへ追加される実行ファイル
├── settings.json           # 既定設定
├── .mcp.json               # MCPサーバー定義
├── .lsp.json               # LSPサーバー定義
├── scripts/                # Hookやユーティリティのスクリプト
├── LICENSE
└── CHANGELOG.md
```

各ディレクトリの役割:

| ディレクトリ | 役割 |
|---|---|
| `.claude-plugin/` | `plugin.json` のみ（コンポーネントが既定位置なら省略可） |
| `skills/` | `<name>/SKILL.md` 形式のSkill |
| `commands/` | フラットなMarkdown形式のSkill。新規は `skills/` を使う |
| `agents/` | Subagent定義 |
| `hooks/` | `hooks.json` にHook設定 |
| `.mcp.json` | MCPサーバー |
| `.lsp.json` | LSPサーバー（コードインテリジェンス） |
| `monitors/` | `monitors.json` にバックグラウンドモニタ |
| `bin/` | Plugin有効中、Bash ToolのPATHへ追加される実行ファイル |
| `settings.json` | Plugin有効時に適用される既定設定 |

Skillが1つだけのPluginは、`skills/` を作らずPluginルート直下に `SKILL.md` を置けます。この場合はfrontmatterの `name` が呼び出し名になります。

---

# 6. Path Behavior Rules（重要）

マニフェストでコンポーネントのパスを指定できますが、**「既定を置き換える」ものと「既定に追加する」ものがあります**。

```text
置き換える（デフォルトディレクトリはスキャンされなくなる）
        ↓
commands / agents / workflows / outputStyles
experimental.themes / experimental.monitors

追加する（デフォルトも常にスキャンされる）
        ↓
skills

独自のマージ規則
        ↓
hooks / mcpServers / lspServers
```

したがって、

```json
{
  "commands": ["./specialized/deploy.md"]
}
```

と書くと、**`commands/` は読まれなくなります**。両方使いたいなら明示します。

```json
{
  "commands": ["./commands/", "./specialized/"]
}
```

パスの書き方:

```text
すべてPluginルートからの相対パスで、./ で始める
配列で複数指定できる
skills だけは "." も使える（"." と "./" はどちらもPluginルート）
```

---

# 7. 環境変数

Plugin内では3つの変数が展開されます。

| 変数 | 指す場所 | 用途 |
|---|---|---|
| `${CLAUDE_PLUGIN_ROOT}` | Pluginのインストールディレクトリ | スクリプト、バイナリ、同梱設定 |
| `${CLAUDE_PLUGIN_DATA}` | 更新をまたぐ永続ディレクトリ | 依存関係、生成物、キャッシュ |
| `${CLAUDE_PROJECT_DIR}` | プロジェクトルート | プロジェクト側のスクリプトや設定 |

展開される場所:

| コンポーネント | 展開されるフィールド |
|---|---|
| Skill / Agent の本文 | どこでも |
| Hook / Monitor の command | どこでも |
| MCP の stdio サーバー | `command`、`args`、`env` |
| MCP の http / sse / ws サーバー | `url`、`headers`、`headersHelper` |
| LSP サーバー | `command`、`args`、`env`、`workspaceFolder` |

Hookで同梱スクリプトを呼ぶ例:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}\"/scripts/process.sh"
          }
        ]
      }
    ]
  }
}
```

`${CLAUDE_PLUGIN_DATA}` は `~/.claude/plugins/data/{id}/` に解決されます（`{id}` はPlugin IDの非英数字を `-` に置換したもの）。

```text
Pluginのバージョンをまたいで残る
最後のスコープからアンインストールすると削除される
```

Python venvや言語ランタイムの依存をここへ入れる、といった使い方をします。

---

# 8. `userConfig` ― 利用者に入力させる値

Plugin有効化時に値を尋ねられます。

```json
{
  "userConfig": {
    "api_endpoint": {
      "type": "string",
      "title": "API endpoint",
      "description": "Your team's API endpoint"
    },
    "api_token": {
      "type": "string",
      "title": "API token",
      "description": "API authentication token",
      "sensitive": true
    }
  }
}
```

フィールド:

| フィールド | 必須 | 説明 |
|---|---|---|
| `type` | ○ | `string` / `number` / `boolean` / `directory` / `file` |
| `title` | ○ | 設定ダイアログのラベル |
| `description` | ○ | ヘルプテキスト |
| `sensitive` | × | `true` で入力をマスクし安全な保管先へ |
| `required` | × | `true` で空欄をエラーに |
| `default` | × | 未入力時の値 |
| `multiple` | × | `string` で配列を許可 |
| `min` / `max` | × | `number` の範囲 |

参照方法:

```text
非機密の値
    ↓
user settings.json の pluginConfigs[<plugin-id>].options

機密の値
    ↓
macOS Keychain または ~/.claude/.credentials.json

環境変数
    ↓
CLAUDE_PLUGIN_OPTION_<KEY> としてhook/MCP/LSPプロセスへ

インライン置換
    ↓
${user_config.KEY}（MCP/LSP設定、Skill/Agent本文）
```

**注意**として、shellコマンドでは `${user_config.KEY}` の置換が拒否されます。`args` を使うexec形式にするか、環境変数から読みます。

---

# 9. Skillの名前空間

Plugin由来のSkillは常に名前空間が付きます。

```text
my-plugin/skills/review/SKILL.md
        ↓
/my-plugin:review
```

frontmatterの `name` は、Pluginでは**コマンド名の最後のセグメントを置き換えます**。

```text
name: fancy
        ↓
/my-plugin:fancy
```

さらに、他のコマンドと衝突しなければ素の `/fancy` でも呼べます。

名前空間があるおかげで、

```text
Plugin由来のSkill
    ↓
プロジェクトの同名Skillと衝突しない
    ↓
両方が併存する
```

となります。Subagentは事情が異なり、プロジェクトやuserの `.claude/agents/` が**同名のPlugin agentを上書きします**。

---

# 10. Skillsディレクトリ内でPluginを開発する

`--plugin-dir` を毎回渡さずに開発する方法があります。

```bash
claude plugin init my-tool
```

これで `~/.claude/skills/my-tool/` に `.claude-plugin/plugin.json` と `SKILL.md` の雛形が作られます。

```text
次のセッションから
    ↓
my-tool@skills-dir として自動ロード
    ↓
マーケットプレイスもインストールも不要
```

既存のSkillフォルダに `.claude-plugin/plugin.json` を足すだけでも同じ扱いになり、agents / hooks / MCPサーバーを同梱できるようになります。

プロジェクトの `.claude/skills/` でこれを行う場合は、workspace trustダイアログの承認が必要です。

---

# 11. ローカルでのテスト

```bash
claude --plugin-dir ./my-plugin
```

`.zip` も渡せます。

```bash
claude --plugin-dir ./my-plugin.zip
```

複数指定もできます。

```bash
claude --plugin-dir ./plugin-one --plugin-dir ./plugin-two
```

URL上のzipを読む場合:

```bash
claude --plugin-url https://example.com/my-plugin.zip
```

```text
--plugin-dir のPluginは
インストール済みの同名Pluginより優先される
    ↓
アンインストールせずに変更を試せる
```

ただし、managed settingsが強制的に有効化・無効化しているPluginは上書きできません。

変更を反映するには:

```text
/reload-plugins
```

これでPlugin、Skill、Agent、Hook、Plugin由来のMCP/LSPサーバーが再読み込みされます。

---

# 12. コンポーネントの動作確認

```text
Skill
    ↓
/plugin-name:skill-name で呼ぶ

Agent
    ↓
/context の Custom Agents に出るか
または @メンションで呼ぶ

Hook
    ↓
対応するイベントを起こして効果を確認
--debug でマッチと終了コードを見る

MCP
    ↓
claude mcp list で Connected か確認

LSP
    ↓
/plugin の Errors タブを確認
```

LSPサーバーが起動に失敗した場合は Errors タブに出ます（例: `Executable not found in $PATH`）。設定が無効な場合はスキップされるだけなので、`claude --debug` で理由を確認します。

---

# 13. `hooks/hooks.json`

`settings.json` の `hooks` オブジェクトと同じ形式です。

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.file_path' | xargs npm run lint:fix"
          }
        ]
      }
    ]
  }
}
```

`.claude/settings.json` の `hooks` をそのままコピーできます。

---

# 14. `.lsp.json` ― LSPサーバー

コードインテリジェンスをClaudeへ提供します。

```json
{
  "go": {
    "command": "gopls",
    "args": ["serve"],
    "extensionToLanguage": {
      ".go": "go"
    }
  }
}
```

利用者側に言語サーバーのバイナリが必要です。

TypeScript / Python / Rust など主要言語には公式マーケットプレイスに用意があるため、自作は**公式が対応していない言語のみ**にします。

---

# 15. `monitors/monitors.json` ― バックグラウンドモニタ

ログやファイルを監視して、イベントをClaudeへ通知します。

```json
[
  {
    "name": "error-log",
    "command": "tail -F ./logs/error.log",
    "description": "Application error log"
  }
]
```

```text
command の標準出力1行ごとに
        ↓
セッション中のClaudeへ通知として届く
```

Plugin有効時に自動で起動するので、「監視を始めて」と指示する必要はありません。

---

# 16. `settings.json` ― Pluginの既定設定

Pluginルートに置くと、有効化時に設定が適用されます。

```json
{
  "agent": "security-reviewer"
}
```

現在サポートされるのは `agent` と `subagentStatusLine` の2キーだけです。

`agent` を設定すると、そのPluginのagentがメインスレッドとして動きます。

```text
Pluginを有効化する
        ↓
Claude Code全体の振る舞いが変わる
```

という強い効果があるため、配布時は説明を明記します。

`settings.json` は `plugin.json` 内の `settings` 宣言より優先されます。未知のキーは黙って無視されます。

---

# 17. `bin/` ― 実行ファイル

`bin/` に置いた実行ファイルは、Plugin有効中にBash ToolのPATHへ追加されます。

```text
my-plugin/
└── bin/
    └── my-tool
```

```text
Skill本文に「my-tool を実行する」と書ける
        ↓
絶対パスを書かなくてよい
```

ただし、claude.aiの組織設定経由で配布するPluginには含められません。

---

# 18. バージョン管理と `defaultEnabled`

```json
{
  "version": "2.1.0",
  "defaultEnabled": false
}
```

```text
version を設定する
    ↓
これを上げたときだけ利用者へ更新が届く
（command source を除く）

version を省略する
    ↓
別のソースからバージョンが決まる
```

`defaultEnabled: false` にすると、インストールしても無効の状態で配られます。

有効・無効の優先順位:

```text
① ユーザーの enabledPlugins 設定
② Plugin依存関係による要求
③ defaultEnabled フィールド
④ マーケットプレイスエントリの設定
```

---

# 19. `dependencies` ― Plugin間の依存

```json
{
  "dependencies": [
    "helper-lib",
    { "name": "secrets-vault", "version": "~2.1.0" }
  ]
}
```

semverの制約を付けられます。依存が満たされない場合はロード時エラーとして報告されます。

---

# 20. Node.js依存の自動インストール

Pluginのインストール時・更新時・未キャッシュ時に、Node.jsの依存が自動インストールされます。

| ロックファイル | 実行されるコマンド |
|---|---|
| `bun.lock` / `bun.lockb` | `bun install --frozen-lockfile --ignore-scripts` |
| `npm-shrinkwrap.json` / `package-lock.json` | `npm ci --ignore-scripts` |

優先順位は `bun.lock` → `bun.lockb` → `npm-shrinkwrap.json` → `package-lock.json` です。

Yarn / pnpm のロックファイルはスキップされます（解決時のconfig hookが `--ignore-scripts` を迂回するため）。

制約:

```text
ロックファイルの厳密なバージョンのみ
ライフサイクルスクリプトは実行されない
60秒でタイムアウト（失敗してもPluginのロードは止まらない）
```

ライフサイクルスクリプトが必要な依存は、`SessionStart` Hookで `${CLAUDE_PLUGIN_DATA}` へインストールします。

---

# 21. キャッシュと更新

マーケットプレイス由来のPluginは `~/.claude/plugins/cache/` へバージョン別にコピーされます。

```text
各バージョンが
    ↓
独立したファイルのコピー
    ↓
独立したnpm依存
```

旧バージョンは並行セッションのために**約14日間**保持されてから削除されます。

symlinkの扱い:

```text
Pluginディレクトリ内     → 相対symlinkを保持
マーケットプレイス内の別所 → 対象の内容をキャッシュへコピー
マーケットプレイス外     → セキュリティのためスキップ
```

---

# 22. 検証：`claude plugin validate`

```bash
claude plugin validate ./your-plugin
```

```text
✔ Validation passed
✔ Validation passed with warnings
```

警告をエラー扱いにするには:

```bash
claude plugin validate ./your-plugin --strict
```

未知のトップレベルフィールドは無視されるため（他エコシステムとのマニフェスト共用のため）、**フィールド名のtypoは `--strict` でないと見つかりません**。CIに入れるなら `--strict` を推奨します。

型が合わない場合:

```text
ほとんどのフィールド → Pluginがロードに失敗
experimental / metadata → 非オブジェクト値は無視され、validateで警告
```

---

# 23. 既存の `.claude/` からの移行

## 手順

```bash
mkdir -p my-plugin/.claude-plugin
```

`my-plugin/.claude-plugin/plugin.json`:

```json
{
  "name": "my-plugin",
  "description": "Migrated from standalone configuration",
  "version": "1.0.0"
}
```

ファイルをコピー:

```bash
cp -r .claude/commands my-plugin/
cp -r .claude/agents   my-plugin/
cp -r .claude/skills   my-plugin/
```

Hookを移す:

```bash
mkdir my-plugin/hooks
```

`.claude/settings.json` の `hooks` オブジェクトをそのまま `my-plugin/hooks/hooks.json` へ写します。

テスト:

```bash
claude --plugin-dir ./my-plugin
```

## 何が変わるか

| standalone（`.claude/`） | Plugin |
|---|---|
| 1プロジェクトでのみ利用可能 | マーケットプレイス経由で共有可能 |
| `.claude/commands/` にファイル | `plugin-name/commands/` にファイル |
| `settings.json` にHook | `hooks/hooks.json` にHook |
| 共有には手動コピー | `/plugin install` でインストール |

移行後は元のファイルを `.claude/` から削除します。

```text
Subagent
    ↓
プロジェクト/userの .claude/agents/ が
同名のPlugin agentを上書きする
    ↓
元を消さないとPlugin版が効かない

Skill
    ↓
名前空間が付くので両方残る
    ↓
/skill-name と /plugin-name:skill-name が併存
```

---

# 24. マーケットプレイス

配布するには、

```text
① README.md を書く
② version の運用方針を決める
③ マーケットプレイスを作る、または既存のものへ登録する
④ チームでテストする
```

社内限定にしたい場合は、プライベートリポジトリでマーケットプレイスをホストします。

Anthropicの公開マーケットプレイス:

```text
claude-plugins-official
    ↓
Anthropicがキュレーションする公式セット
初回の対話起動時に自動登録される

claude-community
    ↓
第三者の提出がレビューを経て掲載される
```

コミュニティマーケットプレイスへの追加:

```bash
/plugin marketplace add anthropics/claude-plugins-community
```

提出前には必ず `claude plugin validate` を通します（レビュー側も同じチェックを行います）。

---

# 25. デバッグ

```text
Pluginが読み込まれない
        ↓
① ディレクトリが .claude-plugin/ の中に入っていないか
② plugin.json が正しいJSONか
③ claude plugin validate --strict を実行
④ /plugin の Errors タブを確認
⑤ claude --debug で起動

Skillが呼べない
        ↓
名前空間を確認（/plugin-name:skill）
/reload-plugins を実行

Hookが効かない
        ↓
hooks/hooks.json の場所を確認
--debug でマッチを確認

MCPが繋がらない
        ↓
claude mcp list
${CLAUDE_PLUGIN_ROOT} が展開されているか
```

`--plugin-dir` でPluginがロードできなかった場合も、`/plugin` の Errors タブに記録されます。

---

# 26. Plugin設計の原則

```text
① 1 Plugin = 1つのまとまった目的
     「便利ツール詰め合わせ」にしない

② standaloneで検証してから移行する
     最初からPluginにすると反復が遅い

③ 名前空間を意識する
     name がそのままコマンド接頭辞になる

④ 破壊的なSkillは disable-model-invocation: true
     配布物ほど事故の影響が広い

⑤ allowed-tools を広げない
     利用者は中身を精査しないことがある

⑥ settings.json の agent は慎重に
     Plugin有効化で全体の挙動が変わる

⑦ 依存を明示する
     必要なバイナリを compatibility や README に書く

⑧ version を上げる
     上げないと利用者へ更新が届かない
```

---

# 27. セキュリティ上の注意

Pluginは**任意のコードを実行できます**。

```text
hooks
    ↓
シェルコマンド

bin/
    ↓
PATHへ追加される実行ファイル

.mcp.json
    ↓
ローカルプロセス起動

SessionStart hook
    ↓
起動のたびに実行
```

したがって、

```text
インストールするPluginは信頼できるものだけ
配布するPluginはスクリプトの内容を明示する
機密値は userConfig の sensitive を使う
利用者に秘密を平文で書かせない
```

を守ります。

組織で制限したい場合は、managed settingsで、

```json
{
  "allowManagedHooksOnly": true
}
```

とすると、管理者が配ったHookと強制有効化されたPluginのHookだけが動きます。

---

# 28. Plugin / standalone の使い分け

```text
自分だけが使う
    ↓
~/.claude/skills/ など standalone

このプロジェクトだけ
    ↓
<project>/.claude/ の standalone

チームへ配る
    ↓
Plugin

複数の仕組みをまとめて配る
    ↓
Plugin（Skill + Agent + Hook + MCP）

バージョン管理して更新を届けたい
    ↓
Plugin + マーケットプレイス
```

判断の目安は、

```text
「他人のマシンで動く必要があるか」
```

です。必要ならPlugin、不要ならstandaloneで十分です。

---

# 29. 最終チェックリスト

```text
[ ] skills/ agents/ hooks/ が .claude-plugin/ の外にある
[ ] plugin.json の name がkebab-case
[ ] version を設定した（更新を届ける場合）
[ ] description がPluginマネージャで意味をなす
[ ] commands / agents を明示指定して既定を潰していない
[ ] パスが ./ で始まる相対パスになっている
[ ] スクリプト参照が ${CLAUDE_PLUGIN_ROOT} を使っている
[ ] 機密値が userConfig の sensitive になっている
[ ] shellコマンドで ${user_config.*} を使っていない
[ ] 破壊的Skillに disable-model-invocation を付けた
[ ] allowed-tools が広すぎない
[ ] claude plugin validate --strict が通る
[ ] --plugin-dir で全コンポーネントを動作確認した
[ ] README.md にインストールと使い方を書いた
[ ] 必要な外部バイナリを明記した
```

---

# 30. まとめ

Pluginの構成を最も簡潔にまとめると、

```text
my-plugin/
│
├── .claude-plugin/plugin.json
│     ↓
│   名前 / バージョン / 配布情報 / userConfig
│
├── skills/          … Skill（名前空間付き）
├── agents/          … Subagent
├── hooks/hooks.json … Hook
├── .mcp.json        … MCPサーバー
├── .lsp.json        … LSPサーバー
├── monitors/        … バックグラウンド監視
├── bin/             … PATHへ追加する実行ファイル
├── settings.json    … 既定設定（agent など）
└── scripts/         … 決定論的処理
```

そしてClaude Code全体での位置づけは、

```text
.claude/（standalone）
=
手元で速く作る

Plugin
=
まとめて配る・バージョンを付ける

マーケットプレイス
=
配布と更新の経路
```

です。

Pluginの本質は「新しい機能」ではなく、

```text
Skill / Agent / Hook / MCP を
1つの単位にまとめ
名前空間を与え
バージョンを付けて配布可能にする
```

というパッケージングの仕組みです。したがって、まずstandaloneで中身を作り込み、**共有する必要が出てから**Pluginへ変換するのが最も無駄がありません。

---

# 参考資料

Anthropic公式 Claude Code Docs:

- Create plugins  
  https://code.claude.com/docs/en/plugins

- Plugins reference  
  https://code.claude.com/docs/en/plugins-reference

- Discover and install plugins  
  https://code.claude.com/docs/en/discover-plugins

- Create and distribute a plugin marketplace  
  https://code.claude.com/docs/en/plugin-marketplaces

特に確認するとよい項目:

- Plugin structure overview
- Plugin manifest schema
- Plugin directory structure
- Path behavior rules
- Environment variables
- User configuration
- Skills-directory plugins
- Test your plugins locally
- Node.js package dependencies
- Convert existing configurations to plugins
