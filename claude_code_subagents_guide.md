# Claude Code：Subagent（.claude/agents/）の構成（中身）についての解説

> 対象: Claude Code の Subagents  
> 更新基準: 2026-08-29 時点のAnthropic公式ドキュメント  
> 目的: `.claude/agents/*.md` に「何を・どのように書けばよいか」を、設計意図も含めて理解する

---

<!-- toc -->
# 目次

- [1. Subagentとは何か](#1-subagentとは何か)
- [2. なぜSubagentを使うのか](#2-なぜsubagentを使うのか)
- [3. ファイル形式](#3-ファイル形式)
- [4. 置き場所と優先順位](#4-置き場所と優先順位)
- [5. frontmatter一覧](#5-frontmatter一覧)
- [6. description ― 委譲判断の要](#6-description--委譲判断の要)
- [7. tools ― 許可リスト](#7-tools--許可リスト)
- [8. disallowedTools ― 拒否リスト](#8-disallowedtools--拒否リスト)
- [9. Subagentが使えるTool](#9-subagentが使えるtool)
- [10. Agent(agent_type) で入れ子を制限](#10-agentagent_type-で入れ子を制限)
- [11. model](#11-model)
- [12. permissionMode](#12-permissionmode)
- [13. maxTurns](#13-maxturns)
- [14. skills ― Skillのプリロード](#14-skills--skillのプリロード)
- [15. mcpServers](#15-mcpservers)
- [16. hooks](#16-hooks)
- [17. memory ― 永続メモリ](#17-memory--永続メモリ)
- [18. isolation: worktree](#18-isolation-worktree)
- [19. 組み込みSubagent](#19-組み込みsubagent)
- [20. Explore / Plan はCLAUDE.mdを読まない](#20-explore--plan-はclaudemdを読まない)
- [21. Subagentの呼び出し方](#21-subagentの呼び出し方)
- [22. Subagentの再開](#22-subagentの再開)
- [23. 入れ子と同時実行数](#23-入れ子と同時実行数)
- [24. バックグラウンドとフォアグラウンド](#24-バックグラウンドとフォアグラウンド)
- [25. Fork（会話の分岐）](#25-fork会話の分岐)
- [26. Skillの context: fork との関係](#26-skillの-context-fork-との関係)
- [27. CLIでSubagentを定義する](#27-cliでsubagentを定義する)
- [28. Subagentを無効化する](#28-subagentを無効化する)
- [29. 本文（system prompt）の書き方](#29-本文system-promptの書き方)
- [30. Subagentは質問できない](#30-subagentは質問できない)
- [31. テンプレート：調査Subagent](#31-テンプレート調査subagent)
- [32. テンプレート：レビューSubagent](#32-テンプレートレビューsubagent)
- [33. テンプレート：実装Subagent](#33-テンプレート実装subagent)
- [34. Subagentの設計原則](#34-subagentの設計原則)
- [35. アンチパターン](#35-アンチパターン)
- [36. Subagent / Skill / CLAUDE.md の使い分け](#36-subagent--skill--claudemd-の使い分け)
- [37. 最終チェックリスト](#37-最終チェックリスト)
- [38. まとめ](#38-まとめ)
- [参考資料](#参考資料)

<!-- /toc -->

---

# 1. Subagentとは何か

Subagentは、**独立したcontext windowで動く専用のアシスタント**です。

```text
Main Conversation
      │
      ├──────────────┐
      │              ▼
      │        Subagent
      │        （別context）
      │              │
      │        独自system prompt
      │        独自Tool制限
      │        独自permission
      │              │
      ◀───── 結果 ───┘
```

メインの会話履歴を引き継がず、終了時に**結果だけ**を返します。

`SKILL.md` が「手順」を定義するのに対し、Subagentは「実行主体」を定義します。

---

# 2. なぜSubagentを使うのか

主な効果は5つです。

```text
① contextの保全
     調査の中間結果をメイン会話へ持ち込まない

② 制約の強制
     使えるToolを限定できる

③ 再利用
     user-levelに置けば全プロジェクトで使える

④ 専門化
     領域特化のsystem promptを与えられる

⑤ コスト制御
     軽いタスクを安価なモデルへ回せる
```

特に①が大きく、

```text
50ファイル読んで調査
        ↓
メイン会話に全部残る
        ↓
context圧迫
```

を、

```text
Subagentが50ファイル読む
        ↓
要約だけメインへ返る
```

に変えられます。

---

# 3. ファイル形式

`SKILL.md` と同じく、YAML frontmatter + Markdown本文です。

```markdown
---
name: code-reviewer
description: Reviews code for quality and best practices
tools: Read, Glob, Grep
model: sonnet
---

You are a code reviewer. When invoked, analyze the code and provide
specific, actionable feedback on quality, security, and best practices.
```

`SKILL.md` との決定的な違いは、

```text
SKILL.md の本文
    ↓
Claudeへの指示（プロンプト）

Subagentの本文
    ↓
そのSubagentのsystem prompt
```

という点です。

---

# 4. 置き場所と優先順位

| 置き場所 | スコープ | 優先度 |
|---|---|---|
| Managed settings | 組織全体 | 1（最優先） |
| `--agents` CLIフラグ | そのセッション | 2 |
| `.claude/agents/` | そのプロジェクト | 3 |
| `~/.claude/agents/` | 自分の全プロジェクト | 4 |
| Pluginの `agents/` | Plugin有効時 | 5（最低） |

`.claude/agents/` と `~/.claude/agents/` は**再帰的にスキャン**されるので、サブフォルダで整理できます。

```text
agents/
├── review/
│   └── security.md
└── research/
    └── documentation.md
```

Skillとは優先順位が逆（Skillはpersonalがprojectに勝つ）なので、混同しないよう注意します。

---

# 5. frontmatter一覧

| フィールド | 必須 | 説明 |
|---|---|---|
| `name` | ○ | 小文字とハイフンによる一意な識別子 |
| `description` | ○ | どんなときに委譲すべきか |
| `tools` | × | 使えるTool（省略時は継承） |
| `disallowedTools` | × | 継承リストから除外するTool |
| `model` | × | `sonnet` / `opus` / `haiku` / `fable` / モデルID / `inherit`（既定） |
| `permissionMode` | × | `default` / `acceptEdits` / `auto` / `dontAsk` / `bypassPermissions` / `plan` / `manual` |
| `maxTurns` | × | 停止までの最大ターン数 |
| `skills` | × | 起動時にプリロードするSkill |
| `mcpServers` | × | このSubagentで使えるMCPサーバー |
| `hooks` | × | このSubagentの実行中だけ有効なHook |
| `memory` | × | 永続メモリのスコープ（`user` / `project` / `local`） |
| `background` | × | `true` でバックグラウンド維持 |
| `effort` | × | `low` / `medium` / `high` / `xhigh` / `max` |
| `isolation` | × | `worktree` で独立したgit worktree |
| `color` | × | 表示色（`red` / `blue` / `green` ほか） |
| `initialPrompt` | × | 最初のユーザーターンとして自動送信 |
| `experimental` | × | `cacheTtl` などの実験的オプション |

`name` と `description` だけが必須です。

---

# 6. `description` ― 委譲判断の要

Claudeが「このタスクをこのSubagentへ渡すか」を決める情報です。

悪い例:

```yaml
description: コードを見るagent
```

良い例:

```yaml
description: >
  実装済みコードの品質レビューを行う。
  correctness、error handling、テスト不足の確認を
  依頼されたとき、またはコード変更後に使用する。
```

`SKILL.md` の `description` と同じ原則です。

```text
何をするか
    +
どんなときに使うか
```

を具体的に書きます。「Use proactively after code changes」のように、能動的に使ってよいかを書くのも有効です。

---

# 7. `tools` ― 許可リスト

省略すると、メイン会話のTool群を継承します。

```yaml
---
name: safe-researcher
description: Research agent with restricted capabilities
tools: Read, Grep, Glob, Bash
---
```

`permissions.allow` と違い、これは**利用可能なToolそのものを絞る**指定です。

```text
permissions.allow
    ↓
確認をスキップする

tools
    ↓
そもそも使えるToolを決める
```

---

# 8. `disallowedTools` ― 拒否リスト

継承したうえで一部だけ外したいときに使います。

```yaml
---
name: no-writes
description: Inherits all tools except file writes
disallowedTools: Write, Edit
---
```

MCPサーバー単位でも書けます。

```yaml
disallowedTools: mcp__github
```

`tools` と `disallowedTools` は併用でき、

```text
tools で絞る
    ↓
disallowedTools でさらに外す
```

という順になります。

---

# 9. Subagentが使えるTool

Subagentは、メイン会話のToolを2段階のフィルタを通して継承します。

```text
第1フィルタ（全Subagent）
    ↓
以下を除外
    Agent
    AskUserQuestion
    EndConversation
    EnterPlanMode / ExitPlanMode
    ScheduleWakeup
    TaskOutput
    WaitForMcpServers
    Workflow
```

```text
第2フィルタ（バックグラウンドSubagentのみ）
    ↓
以下だけを残す
    Read / Grep / Glob
    Bash / PowerShell
    Edit / Write / NotebookEdit
    WebFetch / WebSearch
    TodoWrite / Skill / ToolSearch
    EnterWorktree / ExitWorktree
    Monitor / TaskStop / SendMessage
    MCP tools
```

つまり、

```text
AskUserQuestion は使えない
    ↓
Subagentはユーザーに聞き返せない
```

という点が設計上重要です。曖昧なタスクをSubagentへ丸投げすると、確認できないまま推測で進みます。

また、バックグラウンド実行ではTool群が狭くなるため、広いToolが必要ならフォアグラウンドにします。

---

# 10. `Agent(agent_type)` で入れ子を制限

Subagentがどのagentを起動できるかも制御できます。

```yaml
---
name: coordinator
description: Coordinates work across specialized agents
tools: Agent(worker, researcher), Read, Bash
---
```

`Agent` は第1フィルタで除外されますが、この形で明示すると特定のagentだけ起動できるようになります。

---

# 11. `model`

Subagentごとにモデルを変えられます。

```yaml
model: sonnet
```

考え方:

```text
分類・抽出・定型調査
    ↓
軽量モデル

設計レビュー・難しい原因分析
    ↓
高性能モデル

メインと揃えたい
    ↓
inherit（既定）
```

コスト制御の主要な手段です。

---

# 12. `permissionMode`

Subagent単位でpermissionの既定を変えられます。

| モード | 挙動 |
|---|---|
| `default`（`manual`） | 確認する |
| `acceptEdits` | 作業ディレクトリ内の編集を自動承認 |
| `auto` | 分類器が確認して自動承認 |
| `dontAsk` | 明示的に許可されたもの以外を自動拒否 |
| `bypassPermissions` | 確認をスキップ |
| `plan` | 読み取り専用の探索 |

Subagentは確認を返せないため、

```text
読み取り専用Subagent
    ↓
tools を絞る + permissionMode: plan

自動編集させたいSubagent
    ↓
permissionMode: acceptEdits
```

という組み合わせが実用的です。

`bypassPermissions` は影響範囲が広いので、`tools` を絞ったうえで使います。

---

# 13. `maxTurns`

暴走の上限です。

```yaml
maxTurns: 20
```

```text
探索系Subagent
    ↓
上限が無いと延々と読み続けることがある
```

調査agentには付けておくと安全です。

---

# 14. `skills` ― Skillのプリロード

Subagentの起動時にSkillの内容を注入します。

```yaml
---
name: api-developer
description: Implement API endpoints following team conventions
skills:
  - api-conventions
  - error-handling-patterns
---

Implement API endpoints. Follow the conventions and patterns from the preloaded skills.
```

これは `context: fork` の逆方向です。

```text
Skillの context: fork
    ↓
Skillがタスク、agentが実行環境

Subagentの skills
    ↓
Subagentが主体、Skillが参照資料
```

「規約を必ず読んだ状態で始めてほしい」ときに使います。

なお `disable-model-invocation: true` のSkillはプリロードされません。

---

# 15. `mcpServers`

Subagent専用のMCPサーバーを定義できます。

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

インライン定義と、既存サーバー名の参照の両方が書けます。

```text
重いMCPサーバー
    ↓
メイン会話には常時繋がない
    ↓
必要なSubagentだけに持たせる
```

とすると、メイン側のtool listingを圧迫しません。

---

# 16. `hooks`

Subagentの実行中だけ有効なHookを登録できます。

```yaml
---
name: code-reviewer
description: Review code changes with automatic linting
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/validate-command.sh"
  PostToolUse:
    - matcher: "Edit|Write"
      hooks:
        - type: command
          command: "./scripts/run-linter.sh"
---
```

スコープの違いを押さえます。

```text
Subagentのhooks
    ↓
そのSubagentの実行中のみ
    ↓
終了時に解除

Skillのhooks
    ↓
発火後、セッションの残り全体
```

逆に「Subagentが起動したとき」に何かしたいなら、settings.json側の `SubagentStart` を使います。

```json
{
  "hooks": {
    "SubagentStart": [
      {
        "matcher": "db-agent",
        "hooks": [
          { "type": "command", "command": "./scripts/setup-db.sh" }
        ]
      }
    ]
  }
}
```

---

# 17. `memory` ― 永続メモリ

Subagentがセッションをまたいで学習内容を保持できます。

```yaml
---
name: code-reviewer
description: Reviews code for quality and best practices
memory: project
---

You are a code reviewer. As you review code, update your agent memory with
patterns, conventions, and recurring issues you discover.
```

スコープと保存先:

```text
user
    ↓
~/.claude/agent-memory/<name>/
全プロジェクトで共有

project
    ↓
.claude/agent-memory/<name>/
プロジェクト固有・バージョン管理可能

local
    ↓
.claude/agent-memory-local/<name>/
プロジェクト固有・バージョン管理外
```

メイン会話のAuto MemoryはSubagentへは読み込まれません。Subagentのメモリは独立したディレクトリです。

---

# 18. `isolation: worktree`

並列で編集させるときに使います。

```yaml
isolation: worktree
```

```text
複数Subagentが同時にファイルを書く
        ↓
衝突する

worktree
        ↓
各自が独立したgit worktreeで作業
```

セットアップに時間とディスクを使うため、**並列編集で衝突する場合だけ**使います。変更がなければ自動で削除されます。

---

# 19. 組み込みSubagent

Claude Codeには最初からいくつか用意されています。

## Explore

```text
モデル  … メイン会話から継承
Tool    … 読み取り専用（Write / Edit は拒否）
用途    … ファイル探索、コード検索、コードベース調査
```

「thoroughness（quick / medium / very thorough）」を指定できます。

## Plan

```text
モデル  … メイン会話から継承
Tool    … 読み取り専用
用途    … plan mode中の調査
```

## general-purpose

```text
モデル  … メイン会話から継承
Tool    … Subagentが使えるすべて
用途    … 複雑な調査、多段の操作、コード変更
```

## その他

```text
claude              … 専門agentに当てはまらないタスク全般
statusline-setup    … /statusline の設定用
claude-code-guide   … Claude Code自体の機能に関する質問
```

---

# 20. Explore / Plan はCLAUDE.mdを読まない

重要な仕様です。

```text
一般のSubagent
    ↓
CLAUDE.md を読む

Explore / Plan
    ↓
CLAUDE.md と git status をスキップ
    ↓
contextを小さく保つ
```

したがって、

```text
agent: Explore で Skill を fork する
        ↓
Subagentが見るのは
  SKILL.md本文
  + Exploreのsystem prompt
だけ
```

になります。プロジェクト規約を前提とした指示を書くなら、SKILL.md本文に必要な情報を含めるか、`general-purpose` を選びます。

---

# 21. Subagentの呼び出し方

## 自然言語

```text
Use the code-improver agent to suggest improvements in this project
```

## @メンション

```text
@"code-reviewer (agent)" look at the auth changes
```

`@agent-<name>` の形でも指定できます。

## セッション既定にする

```bash
claude --agent code-reviewer
```

または `.claude/settings.json` で:

```json
{
  "agent": "code-reviewer"
}
```

こうするとメインスレッド自体がそのagentの設定（system prompt / Tool制限 / モデル）で動きます。

---

# 22. Subagentの再開

Subagentは会話履歴を保ったまま継続できます。

```text
Use the code-reviewer subagent to review the authentication module
        ↓
（Subagent完了）
        ↓
Continue that code review and now analyze the authorization logic
```

内部的には `SendMessage` にagentのIDまたは名前を渡します。

```text
新しく Agent を呼ぶ
    ↓
まっさらな状態から

SendMessage で続ける
    ↓
前の調査結果を保ったまま
```

同じ対象を掘り下げるなら、再起動より継続のほうが効率的です。

---

# 23. 入れ子と同時実行数

```text
入れ子の深さ
    ↓
既定 3層

同時実行数
    ↓
既定 20
```

変更するには環境変数を設定します。

```json
{
  "env": {
    "CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH": "2",
    "CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS": "50"
  }
}
```

深い入れ子は追跡が難しくなるため、実務では2〜3層に抑えるのが扱いやすいです。

---

# 24. バックグラウンドとフォアグラウンド

```text
フォアグラウンド
    ↓
完了までメイン会話が待つ

バックグラウンド
    ↓
並行して動き、完了時に結果が届く
```

`background: true` を書くと、Claudeがフォアグラウンドを要求してもバックグラウンドを維持します。

注意点として、

```text
バックグラウンド実行
    ↓
Toolセットが狭くなる（第2フィルタ）

バックグラウンドのforkが行った編集
    ↓
checkpointの外なので /rewind で戻せない
    ↓
gitで戻す
```

があります。

---

# 25. Fork（会話の分岐）

Subagentとは別に、**会話をそのまま引き継ぐ**forkがあります。

```text
/subtask draft unit tests for the parser changes so far
```

forkが引き継ぐもの:

```text
会話履歴すべて
メインと同じsystem promptとTool
メインと同じモデル
prompt cacheの共有
```

使い分け:

```text
文脈が要る作業
    ↓
fork

文脈を持ち込みたくない作業
    ↓
Subagent
```

---

# 26. Skillの `context: fork` との関係

両者は逆方向の組み合わせです。

| 方式 | system prompt | タスク | 追加でロードされるもの |
|---|---|---|---|
| Skill の `context: fork` | agent種別から | SKILL.mdの本文 | CLAUDE.md（Explore / Plan を除く） |
| Subagent の `skills` | Subagentの本文 | Claudeの委譲メッセージ | プリロードSkill + CLAUDE.md |

判断基準:

```text
タスクの手順が決まっている
    ↓
Skill + context: fork

役割・振る舞いを定義したい
    ↓
Subagent
```

なお `context: fork` は**明確な指示を持つSkill**にしか意味がありません。「このAPI規約に従え」のようなガイドラインだけのSkillをforkすると、Subagentは実行すべきタスクを受け取れず、意味のある結果を返しません。

---

# 27. CLIでSubagentを定義する

ファイルを置かずに渡すこともできます。

```bash
claude --agents '{
  "code-reviewer": {
    "description": "Expert code reviewer. Use proactively after code changes.",
    "prompt": "You are a senior code reviewer. Focus on code quality, security, and best practices.",
    "tools": ["Read", "Grep", "Glob", "Bash"],
    "model": "sonnet"
  }
}'
```

CIやスクリプトで「毎回同じagentを使う」ときに便利です。

---

# 28. Subagentを無効化する

```json
{
  "permissions": {
    "deny": ["Agent(Explore)", "Agent(my-custom-agent)"]
  }
}
```

CLIからは:

```bash
claude --disallowedTools "Agent(Explore)"
```

環境変数でまとめて切ることもできます。

```bash
# 組み込みagentをすべて無効化
export CLAUDE_AGENT_SDK_DISABLE_BUILTIN_AGENTS=1

# Explore と Plan だけ無効化
export CLAUDE_CODE_DISABLE_EXPLORE_PLAN_AGENTS=1
```

---

# 29. 本文（system prompt）の書き方

Subagentの本文はsystem promptなので、「振る舞いの定義」として書きます。

おすすめ構造:

```text
# Role

# Scope

# Procedure

# Decision Rules

# Output

# Constraints
```

`SKILL.md` との書き分け:

```text
SKILL.md
    ↓
「この手順を実行せよ」という命令文

Subagent
    ↓
「あなたはこういう存在である」という定義文
```

例:

```markdown
You are a security reviewer for this codebase.

# Scope

対象:
- Authentication / Authorization
- Input validation
- Injection
- Secrets

対象外:
- 実サービスへの攻撃
- 負荷試験

# Procedure

1. 変更差分を確認する。
2. 外部入力の経路を特定する。
3. source-to-sink を追跡する。
4. 根拠となるコードを引用する。

# Output

各Findingについて Severity / File / Line / Evidence / Fix を報告する。

# Constraints

- 推測で脆弱性と断定しない。
- Secretの値を出力しない。
- ファイルを編集しない。
```

---

# 30. Subagentは質問できない

設計上、最も重要な制約です。

```text
AskUserQuestion が使えない
        ↓
曖昧な指示を渡すと
        ↓
推測で進む
```

対策:

```text
① descriptionとsystem promptで前提を固定する
② 曖昧な場合の既定動作を明記する
③ 「不明な点はOpen Questionsとして報告する」と書く
```

例:

```markdown
# Constraints

- 判断できない点は推測で埋めず、Open Questions として列挙する。
- 仕様が不明な場合は変更を行わず、選択肢を提示する。
```

---

# 31. テンプレート：調査Subagent

```markdown
---
name: deep-researcher
description: >
  コードベースを横断して詳細調査する。
  architecture、依存関係、既存実装の把握が必要なときに使用する。
tools: Read, Grep, Glob
model: inherit
maxTurns: 30
color: blue
---

You are a codebase research specialist.

# Procedure

1. Glob で関連ファイルを探索する。
2. Grep でシンボルと参照箇所を特定する。
3. 主要実装を読む。
4. データフローと依存関係を追跡する。
5. 根拠となる file path と行番号を記録する。

# Output

- Summary
- Relevant files（path付き）
- Architecture
- Data flow
- Risks
- Open questions

# Constraints

- ファイルを変更しない。
- 読んでいないコードについて断定しない。
- 推測は Open questions に分離する。
```

---

# 32. テンプレート：レビューSubagent

```markdown
---
name: code-reviewer
description: >
  実装済みコードの品質レビューを行う。
  コード変更後、またはレビュー依頼時に proactive に使用する。
tools: Read, Grep, Glob, Bash(git diff *), Bash(git log *)
model: inherit
effort: high
memory: project
color: green
---

You are a senior code reviewer for this repository.

# Procedure

1. `git diff` で変更範囲を確認する。
2. correctness を確認する。
3. error handling を確認する。
4. 既存パターンとの整合性を確認する。
5. API互換性を確認する。
6. test coverage を確認する。

# Output

Findings を以下で分類する。

- Blocking
- Important
- Suggestion

各Findingに file path と根拠を付ける。

# Constraints

- スタイルの好みだけの指摘はしない。
- 根拠のない推測を Blocking にしない。
- ファイルを編集しない（指摘のみ）。

レビュー中に見つけた繰り返し出る問題やこのリポジトリ固有の規約は、
agent memory に記録して次回以降に活かす。
```

---

# 33. テンプレート：実装Subagent

```markdown
---
name: implementer
description: >
  指定されたタスクの実装を行う。
  変更範囲が明確で、独立して進められる作業に使用する。
disallowedTools: WebFetch, WebSearch
permissionMode: acceptEdits
model: inherit
maxTurns: 40
skills:
  - project-conventions
isolation: worktree
---

You implement well-scoped changes in this repository.

# Procedure

1. 関連する既存実装とテストを読む。
2. 最小の変更方針を決める。
3. 実装する。
4. 関連テストを実行する。
5. 失敗したら原因を特定して修正し、再実行する。

# Output

- 変更したファイル
- 変更内容の要約
- 実行したテストと結果
- 未対応事項

# Constraints

- 依頼範囲を超える変更をしない。
- 無関係なファイルを整形しない。
- テストが失敗したまま完了を報告しない。
- 判断できない点は推測せず Open questions に書く。
```

---

# 34. Subagentの設計原則

```text
① 1 Subagent = 1つの役割
     何でもできるagentは委譲判断を難しくする

② Tool は必要最小限
     読み取り専用で足りるなら書かせない

③ 出力形式を決める
     メインへ返るのは要約だけなので形式が効く

④ 曖昧さを残さない
     聞き返せないため既定動作を明記する

⑤ maxTurns で上限を置く
     特に探索系

⑥ 重い依存はSubagentへ寄せる
     MCPサーバーやSkillをメインに常駐させない
```

---

# 35. アンチパターン

```text
× 万能agent
     description が広すぎて誤委譲する

× メイン会話と同じTool・同じ設定のagent
     わざわざ分ける意味がない

× 文脈が必要な作業をSubagentへ渡す
     会話履歴を持たないので前提が抜ける
     → fork を使う

× ガイドラインだけのSkillを context: fork する
     タスクが無いので何も返らない

× 出力形式を決めずに調査させる
     長文が返ってきてcontext節約にならない

× bypassPermissions + tools無制限
     制約強制というSubagentの利点を捨てている
```

---

# 36. Subagent / Skill / CLAUDE.md の使い分け

```text
CLAUDE.md
    ↓
常時必要な方針

.claude/rules/
    ↓
特定pathの方針

SKILL.md
    ↓
特定タスクの手順

Subagent
    ↓
独立contextで動く実行主体
```

例:

```text
「TypeScript strictを保つ」
    ↓
CLAUDE.md

「security reviewを12ステップで実施」
    ↓
Skill

「読み取り専用で、Sonnetで、20ターン以内に調査する存在」
    ↓
Subagent
```

Skillとの組み合わせも有効です。

```text
Skill（手順）
    +
context: fork + agent（実行環境）
    ↓
手順と実行主体を分離できる
```

---

# 37. 最終チェックリスト

新しいSubagentを作ったら確認します。

```text
[ ] 役割が1つに絞られている
[ ] description に「いつ委譲するか」がある
[ ] tools が必要最小限
[ ] 書き込みが不要なら Write / Edit を外した
[ ] permissionMode が役割に合っている
[ ] maxTurns を設定した（特に探索系）
[ ] 出力形式を指定した
[ ] 曖昧なときの既定動作を書いた
[ ] 質問できない前提で指示が閉じている
[ ] Explore / Plan を使う場合、CLAUDE.mdが無くても成立する
[ ] 並列編集するなら isolation: worktree を検討した
[ ] メイン会話と役割が重複していない
[ ] 実タスクで委譲されるか確認した
[ ] 誤委譲しないか確認した
```

---

# 38. まとめ

Subagentの構成を最も簡潔にまとめると、

```text
.claude/agents/<name>.md
│
├── YAML Frontmatter
│   │
│   ├─ name / description       … 何者で、いつ委譲するか
│   ├─ tools / disallowedTools  … 何ができるか
│   ├─ model / effort           … どのくらいの能力で動くか
│   ├─ permissionMode / maxTurns … どこまで自律的に動くか
│   ├─ skills / mcpServers      … 何を前提知識として持つか
│   ├─ hooks                    … 実行中に何を強制するか
│   ├─ memory                   … 何を覚えておくか
│   └─ isolation / background   … どこでどう動くか
│
└── Markdown Body
    │
    ▼
  そのSubagentの system prompt
  「あなたはこういう存在である」
```

そしてClaude Code全体では、

```text
CLAUDE.md
=
常時の方針

SKILL.md
=
タスクの手順

Subagent
=
独立contextの実行主体

fork
=
文脈を引き継いだ分岐

Hooks / permissions
=
どの主体にも効く強制層
```

という関係になります。

Subagentを導入する最大の理由は、

```text
メイン会話のcontextを守りつつ
制約を確実にかけた状態で
別の作業を並行させる
```

ことです。この3つが不要なら、Skillやforkのほうが単純で扱いやすい場合が多くあります。

---

# 参考資料

Anthropic公式 Claude Code Docs:

- Create custom subagents  
  https://code.claude.com/docs/en/sub-agents

- Extend Claude with skills  
  https://code.claude.com/docs/en/skills

- Hooks reference  
  https://code.claude.com/docs/en/hooks

特に確認するとよい項目:

- Subagent file format
- Supported frontmatter fields
- Subagent scope and priority
- Built-in subagents
- Tool access control
- Permission modes
- Preload skills into subagents
- Enable persistent memory
- Run subagents in foreground or background
- Fork the current conversation
- What loads at startup
