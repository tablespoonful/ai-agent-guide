# ai-agent-guide

> 対象: Claude Code の設定・拡張のしくみ全般  
> 更新基準: 2026-08-28 時点のAnthropic公式ドキュメント  
> 目的: `.claude/` 配下の各仕組みを「いつ・どれを・どう使うか」で引けるようにする

---

<!-- toc -->
# 目次

- [1. このリポジトリについて](#1-このリポジトリについて)
- [2. ドキュメント一覧](#2-ドキュメント一覧)
- [3. 全体マップ](#3-全体マップ)
- [4. 決定表：どの仕組みを使うか](#4-決定表どの仕組みを使うか)
- [5. .claude/ ディレクトリの全体像](#5-claude-ディレクトリの全体像)
- [6. スコープ早見表](#6-スコープ早見表)
- [7. ロードタイミング早見表](#7-ロードタイミング早見表)
- [8. 強制力の階層](#8-強制力の階層)
- [9. よくある症状から引く索引](#9-よくある症状から引く索引)
- [10. 読む順番](#10-読む順番)
- [11. 最小構成のおすすめ](#11-最小構成のおすすめ)
- [12. このリポジトリの更新方針](#12-このリポジトリの更新方針)
- [参考資料](#参考資料)

<!-- /toc -->

---

# 1. このリポジトリについて

Claude Codeには、振る舞いを制御する仕組みが複数あります。

```text
CLAUDE.md
.claude/rules/
.claude/skills/
.claude/agents/
.claude/settings.json（permissions / hooks）
.mcp.json
Plugin
```

それぞれ「いつロードされるか」「どのくらい強制力があるか」が違います。ここを混同すると、

```text
CLAUDE.mdが肥大化する
危険操作をお願いベースで防ごうとする
毎セッション不要なcontextを消費する
```

といった状態になります。

このリポジトリは、**各仕組みの構成（中身の書き方）と、仕組み同士の使い分け**を1本ずつ解説したドキュメント集です。すべて公式ドキュメント（code.claude.com/docs）と突き合わせて記述しています。

---

# 2. ドキュメント一覧

## 常時の指示

- [CLAUDE.mdの構成](claude_code_claude_md_structure_guide.md)  
  毎セッションロードされる常設コンテキスト。何を書き、何を書かないか。

- [.claude/rules/の構成](claude_code_rules_guide.md)  
  テーマ別・path別にルールを分割する仕組み。`paths` によるglob指定。

## タスクの手順

- [Skillの登録方法](claude_code_skill_registration.md)  
  Skillをどこに置き、どう認識させるか。Personal / Project の使い分け。

- [SKILL.mdの構成](claude_code_skill_md_structure_guide.md)  
  frontmatter全項目と本文の書き方。発火制御、引数、Tool事前承認、Subagent実行。

## 実行主体

- [Subagent（.claude/agents/）の構成](claude_code_subagents_guide.md)  
  独立contextで動く専用エージェント。Tool制限、モデル、永続メモリ、worktree分離。

## 強制と制御

- [settings.jsonとpermissionsの構成](claude_code_settings_permissions_guide.md)  
  設定のスコープと優先順位。allow / ask / deny のルール構文と落とし穴。

- [Hooksの構成](claude_code_hooks_guide.md)  
  ライフサイクルイベントで決定論的な処理を差し込む仕組み。ブロック、検証、自動整形。

## 外部連携と配布

- [MCPサーバー設定の構成](claude_code_mcp_guide.md)  
  外部ツール・データソースの接続。スコープ、transport、認証、セキュリティ。

- [Pluginの構成](claude_code_plugins_guide.md)  
  Skill / Agent / Hook / MCP をまとめて配布する仕組み。

## 自動実行

- [非対話モード（headless / CI）の構成](claude_code_headless_ci_guide.md)  
  `claude -p` の使い方。構造化出力、権限の事前指定、CI特有の注意点。

---

# 3. 全体マップ

```text
                     ┌─────────────────────┐
                     │      CLAUDE.md      │
                     │   常時コンテキスト   │
                     └──────────┬──────────┘
                                │
        ┌───────────────┬───────┴───────┬───────────────┐
        │               │               │               │
        ▼               ▼               ▼               ▼
  .claude/rules/    .claude/skills/  .claude/agents/  settings.json
   path別ルール      task別手順        実行主体       permissions
        │               │               │             + hooks
        │               │               │               │
        └───────────────┴───────┬───────┴───────────────┘
                                │
                                ▼
                          Claude Agent
                                │
                     ┌──────────┴──────────┐
                     │                     │
                     ▼                     ▼
              組み込みTool              MCP tool
              Read / Bash / Edit      外部サービス
                     │                     │
                     └──────────┬──────────┘
                                ▼
                          Gather Context
                                │
                                ▼
                             Action
                                │
                                ▼
                             Verify
                          （Hookで強制可）
                                │
                         ┌──────┴──────┐
                         │             │
                       修正           完了
                         │
                         └────→ 再検証
```

---

# 4. 決定表：どの仕組みを使うか

やりたいことから引く表です。

| やりたいこと | 使う仕組み | 参照 |
|---|---|---|
| どのファイルを触っていても守ってほしい方針 | `CLAUDE.md` | [CLAUDE.md](claude_code_claude_md_structure_guide.md) |
| 特定の種類のファイルでだけ守ってほしい方針 | `.claude/rules/`（`paths` 付き） | [rules](claude_code_rules_guide.md) |
| 特定のサブディレクトリでだけ必要な方針 | nested `CLAUDE.md` | [CLAUDE.md](claude_code_claude_md_structure_guide.md) |
| 決まったタスクの手順を再現させたい | `SKILL.md` | [SKILL.md](claude_code_skill_md_structure_guide.md) |
| ユーザーが `/コマンド` で呼びたい | Skill（既定で呼べる） | [Skill登録](claude_code_skill_registration.md) |
| Claudeに自動で使わせたくない | `disable-model-invocation: true` | [SKILL.md](claude_code_skill_md_structure_guide.md) |
| ユーザーには呼ばせず知識として持たせたい | `user-invocable: false` | [SKILL.md](claude_code_skill_md_structure_guide.md) |
| 調査を別contextでやらせたい | Subagent、または Skill の `context: fork` | [Subagent](claude_code_subagents_guide.md) |
| Toolを限定した専用エージェントが欲しい | Subagent の `tools` | [Subagent](claude_code_subagents_guide.md) |
| 特定のコマンドを実行させたくない | `permissions.deny` | [permissions](claude_code_settings_permissions_guide.md) |
| 確認を減らしたい | `permissions.allow` | [permissions](claude_code_settings_permissions_guide.md) |
| 実行前に動的に検査したい | `PreToolUse` Hook | [Hooks](claude_code_hooks_guide.md) |
| 編集後に必ずlint/formatしたい | `PostToolUse` Hook | [Hooks](claude_code_hooks_guide.md) |
| テストが通るまで終わらせたくない | `Stop` Hook | [Hooks](claude_code_hooks_guide.md) |
| 毎回のセッションに動的な情報を入れたい | `SessionStart` Hook | [Hooks](claude_code_hooks_guide.md) |
| 外部サービスのAPIを使わせたい | MCPサーバー | [MCP](claude_code_mcp_guide.md) |
| チームに設定一式を配りたい | Plugin | [Plugin](claude_code_plugins_guide.md) |
| CIから自動実行したい | `claude -p --bare` | [headless/CI](claude_code_headless_ci_guide.md) |
| 出力を機械的に扱いたい | `--output-format json --json-schema` | [headless/CI](claude_code_headless_ci_guide.md) |

---

# 5. `.claude/` ディレクトリの全体像

```text
~/.claude/                          # ユーザースコープ
├── CLAUDE.md                       # 全プロジェクト共通の個人方針
├── settings.json                   # 個人設定・permissions・hooks
├── rules/                          # 個人のテーマ別ルール
├── skills/
│   └── <skill-name>/SKILL.md       # Personal Skill
├── agents/
│   └── <agent-name>.md             # Personal Subagent
├── agent-memory/                   # Subagentの永続メモリ（userスコープ）
├── plugins/                        # インストール済みPluginのキャッシュとデータ
└── projects/<project>/memory/      # Auto Memory

<project>/
├── CLAUDE.md                       # プロジェクト共通方針（コミット）
├── CLAUDE.local.md                 # 個人用（.gitignore）
├── AGENTS.md                       # 他エージェント用（CLAUDE.mdから@import）
├── .mcp.json                       # Project scopeのMCPサーバー（コミット）
└── .claude/
    ├── CLAUDE.md                   # ルート直下の代わりにここでも可
    ├── settings.json               # チーム共有設定（コミット）
    ├── settings.local.json         # 個人の上書き（.gitignore）
    ├── rules/
    │   ├── testing.md
    │   └── frontend/react.md
    ├── skills/
    │   └── <skill-name>/SKILL.md
    ├── agents/
    │   └── <agent-name>.md
    ├── commands/                   # 旧形式。新規はskills/を使う
    └── agent-memory/               # Subagentの永続メモリ（projectスコープ）
```

---

# 6. スコープ早見表

「誰に効くか」の比較です。

| 仕組み | 自分・全プロジェクト | チーム・1プロジェクト | 自分・1プロジェクト | 組織全体 |
|---|---|---|---|---|
| CLAUDE.md | `~/.claude/CLAUDE.md` | `./CLAUDE.md` | `./CLAUDE.local.md` | managed CLAUDE.md |
| rules | `~/.claude/rules/` | `.claude/rules/` | — | — |
| Skill | `~/.claude/skills/` | `.claude/skills/` | — | managed |
| Subagent | `~/.claude/agents/` | `.claude/agents/` | — | managed |
| settings | `~/.claude/settings.json` | `.claude/settings.json` | `.claude/settings.local.json` | `managed-settings.json` |
| MCP | `--scope user` | `.mcp.json`（`--scope project`） | `--scope local` | — |

注意すべき優先順位の違い:

```text
Skill
    ↓
enterprise > personal > project
（personalがprojectに勝つ）

Subagent
    ↓
managed > --agents > project > personal > plugin
（projectがpersonalに勝つ）

settings.json
    ↓
managed > CLI > local > project > user
```

**SkillとSubagentで逆**なので、同名を複数箇所に置くときは注意します。

---

# 7. ロードタイミング早見表

contextをいつ消費するかの比較です。

| 仕組み | ロードのタイミング |
|---|---|
| `CLAUDE.md`（作業ディレクトリと上位） | セッション開始時 |
| `CLAUDE.md`（サブディレクトリ） | そのディレクトリのファイルを読んだとき |
| `@import` されたファイル | セッション開始時（import元と一緒） |
| `.claude/rules/`（`paths` なし） | セッション開始時 |
| `.claude/rules/`（`paths` あり） | マッチするファイルを読んだとき |
| Skillの一覧（name + description） | セッション開始時 |
| Skillの本文 | 発火したとき。以降ターンをまたいで残る |
| Subagentの本文 | そのSubagentを起動したとき（別context） |
| MCPのtool一覧 | 接続時（tool searchで遅延取得あり） |
| Auto Memory の `MEMORY.md` | セッション開始時（先頭200行 / 25KB） |

`@import` はcontext削減にはならない点が要注意です。

```text
整理・再利用・保守性
    ↓
@import の目的

token削減
    ↓
@import の目的ではない
    ↓
rules（paths付き）または Skill を使う
```

---

# 8. 強制力の階層

同じ「やらせない」でも、確実性が違います。

```text
弱い（Claudeの判断に依存）
  │
  │  CLAUDE.md / rules / SKILL.md
  │      ↓
  │  「〜しないでください」と書く
  │
  │  Subagent の tools
  │      ↓
  │  そもそもToolを渡さない
  │
  │  permissions.deny
  │      ↓
  │  Claude Codeが認識する経路をブロック
  │
  │  PreToolUse Hook
  │      ↓
  │  実行時に検査してブロック
  │
  │  sandbox
  │      ↓
  │  OSレベルで全プロセスを隔離
  │
強い（Claudeの判断と無関係）
```

設計の原則:

```text
危険度が高いものほど下の層で止める
    +
上の層にも書いて「なぜ失敗したか」を伝える
```

`permissions.deny` だけだとClaudeは理由を知らず別の手段を試すので、CLAUDE.mdにも書くのが実務的です。

---

# 9. よくある症状から引く索引

| 症状 | 疑うところ | 参照 |
|---|---|---|
| CLAUDE.mdの指示が守られない | 曖昧な表現／矛盾／未ロード。`/context` で確認 | [CLAUDE.md](claude_code_claude_md_structure_guide.md) |
| CLAUDE.mdが200行を超えた | rules・Skill・nested CLAUDE.md へ分割 | [rules](claude_code_rules_guide.md) |
| rulesに移したがcontextが減らない | `paths` を書き忘れている | [rules](claude_code_rules_guide.md) |
| rulesが発火しない | `*.tsx` はルート直下のみ。`**/*.tsx` にする | [rules](claude_code_rules_guide.md) |
| Skillが誤発火する | `description` が広すぎる。`paths` で絞る | [SKILL.md](claude_code_skill_md_structure_guide.md) |
| Skillが発火しない | `description` に発火条件がない。listing budget超過 | [SKILL.md](claude_code_skill_md_structure_guide.md) |
| Skillが勝手に実行される | `disable-model-invocation: true` を付ける | [SKILL.md](claude_code_skill_md_structure_guide.md) |
| denyしたのに例外を許可できない | denyに例外は書けない。allow + PreToolUse Hook | [permissions](claude_code_settings_permissions_guide.md) |
| `Write(path)` のルールが効かない | ファイルパスは `Edit()` / `Read()` で書く | [permissions](claude_code_settings_permissions_guide.md) |
| 絶対パスのルールが効かない | 絶対パスは `//path`。`/path` は設定ソース基準 | [permissions](claude_code_settings_permissions_guide.md) |
| Hookが効かない | イベント選択ミス（PostToolUseでブロック不可）。`--debug` | [Hooks](claude_code_hooks_guide.md) |
| Hookのmatcherが意図と違う | 記号を含むと正規表現扱いになる | [Hooks](claude_code_hooks_guide.md) |
| Subagentが推測で進む | `AskUserQuestion` が使えない。既定動作を明記 | [Subagent](claude_code_subagents_guide.md) |
| forkしたSkillが何も返さない | 指示のないガイドラインSkillをforkしている | [SKILL.md](claude_code_skill_md_structure_guide.md) |
| MCP toolが見えない | `claude mcp list` で承認待ち／未認証を確認 | [MCP](claude_code_mcp_guide.md) |
| MCPの出力が切れる | `MAX_MCP_OUTPUT_TOKENS`、サーバー側の上限 | [MCP](claude_code_mcp_guide.md) |
| Pluginが読み込まれない | `.claude-plugin/` の中にコンポーネントを置いている | [Plugin](claude_code_plugins_guide.md) |
| CIでマシンごとに結果が違う | `--bare` を付ける | [headless/CI](claude_code_headless_ci_guide.md) |
| CIで設定ミスが素通りする | `plugin_errors` / `mcp_server_errors` を確認 | [headless/CI](claude_code_headless_ci_guide.md) |

---

# 10. 読む順番

はじめて整えるなら、この順が無理がありません。

```text
① CLAUDE.md
      ↓
   まずここだけで動く状態を作る

② settings.json / permissions
      ↓
   危険操作を止める。確認の煩わしさを減らす

③ SKILL.md
      ↓
   繰り返す作業を手順化する

④ .claude/rules/
      ↓
   CLAUDE.mdが長くなってきたら分割する

⑤ Hooks
      ↓
   守られないルールを機械的に強制する

⑥ Subagent
      ↓
   contextを守りたい作業を切り出す

⑦ MCP
      ↓
   外部サービスが必要になったら

⑧ Plugin
      ↓
   チームへ配る段階になったら

⑨ headless / CI
      ↓
   自動実行する段階になったら
```

①〜③だけでも実用になります。④以降は「困ってから」で十分です。

---

# 11. 最小構成のおすすめ

新しいプロジェクトで最初に置くもの:

```text
project/
├── CLAUDE.md
│     ↓
│   Project概要 / Commands / Architecture / Verification
│   （200行以内）
│
└── .claude/
    └── settings.json
          ↓
        permissions.deny に危険操作
        permissions.allow に定型コマンド
```

これで、

```text
Claudeがbuild/testの方法を知っている
アーキテクチャ境界を知っている
完了条件を知っている
危険操作が技術的に止まる
確認の回数が減る
```

という状態になります。ここから、繰り返す作業が見えてきたらSkillへ、守られないルールが見えてきたらHookへ、と足していきます。

---

# 12. このリポジトリの更新方針

```text
公式ドキュメントと突き合わせて事実確認したうえで書く
    ↓
仕様が変わりやすい部分は「更新基準」の日付を確認する
    ↓
各ドキュメント末尾の「参考資料」から公式ページへ辿れるようにする
```

Claude Codeは更新が速く、フィールド名や既定値が変わることがあります。挙動が説明と違う場合は、まず各ドキュメント末尾の公式リンクを確認してください。

---

# 参考資料

Anthropic公式 Claude Code Docs:

- ドキュメント一覧  
  https://code.claude.com/docs/en/overview

- How Claude remembers your project  
  https://code.claude.com/docs/en/memory

- Extend Claude with skills  
  https://code.claude.com/docs/en/skills

- Create custom subagents  
  https://code.claude.com/docs/en/sub-agents

- Claude Code settings  
  https://code.claude.com/docs/en/settings

- Configure permissions  
  https://code.claude.com/docs/en/permissions

- Hooks reference  
  https://code.claude.com/docs/en/hooks

- Connect Claude Code to tools via MCP  
  https://code.claude.com/docs/en/mcp

- Create plugins  
  https://code.claude.com/docs/en/plugins

- Run Claude Code programmatically  
  https://code.claude.com/docs/en/headless
