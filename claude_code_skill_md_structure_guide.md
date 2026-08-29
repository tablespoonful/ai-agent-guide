# Claude Code：SKILL.mdファイルの構成（中身）についての解説

> 対象: Claude Code Skills  
> 更新基準: 2026-08-29 時点のAnthropic公式ドキュメント  
> 目的: `SKILL.md` に「何を・どのように書けばよいか」を、設計意図も含めて理解する

---

<!-- toc -->
# 目次

- [1. SKILL.mdとは何か](#1-skillmdとは何か)
- [2. SKILL.mdの基本構造](#2-skillmdの基本構造)
- [3. YAML frontmatterとは](#3-yaml-frontmatterとは)
- [4. frontmatterは何のためにあるのか](#4-frontmatterは何のためにあるのか)
- [5. name](#5-name)
- [6. description ― 最重要項目](#6-description--最重要項目)
- [7. when_to_use](#7-when_to_use)
- [8. descriptionを長くしすぎない](#8-descriptionを長くしすぎない)
- [9. disable-model-invocation](#9-disable-model-invocation)
- [10. user-invocable](#10-user-invocable)
- [11. 呼び出し設定の整理](#11-呼び出し設定の整理)
- [12. argument-hint](#12-argument-hint)
- [13. $ARGUMENTS](#13-arguments)
- [14. 引数を位置で参照する](#14-引数を位置で参照する)
- [15. 名前付き引数](#15-名前付き引数)
- [16. allowed-tools](#16-allowed-tools)
- [17. allowed-toolsを安易に広げない](#17-allowed-toolsを安易に広げない)
- [18. disallowed-tools](#18-disallowed-tools)
- [19. model](#19-model)
- [20. effort](#20-effort)
- [21. context: fork](#21-context-fork)
- [22. agent](#22-agent)
- [23. background](#23-background)
- [24. paths](#24-paths)
- [25. shell](#25-shell)
- [26. hooks](#26-hooks)
- [27. metadata](#27-metadata)
- [28. license](#28-license)
- [29. compatibility](#29-compatibility)
- [30. frontmatter全体例](#30-frontmatter全体例)
- [31. Markdown本文には何を書くべきか](#31-markdown本文には何を書くべきか)
- [32. Purpose](#32-purpose)
- [33. Scope](#33-scope)
- [34. Procedure](#34-procedure)
- [35. 分岐条件を書く](#35-分岐条件を書く)
- [36. Verification](#36-verification)
- [37. Output](#37-output)
- [38. Constraints](#38-constraints)
- [39. Reference SkillとTask Skill](#39-reference-skillとtask-skill)
- [40. SKILL.mdを巨大化させない](#40-skillmdを巨大化させない)
- [41. 補助ファイルを使う](#41-補助ファイルを使う)
- [42. 補助ファイルを分ける利点](#42-補助ファイルを分ける利点)
- [43. ${CLAUDE_SKILL_DIR}](#43-claude_skill_dir)
- [44. ${CLAUDE_PROJECT_DIR}](#44-claude_project_dir)
- [45. その他の主な変数](#45-その他の主な変数)
- [46. Dynamic Context Injection](#46-dynamic-context-injection)
- [47. Dynamic Context Injectionの注意](#47-dynamic-context-injectionの注意)
- [48. Scriptを直接本文に大量に書かない](#48-scriptを直接本文に大量に書かない)
- [49. Skill設計で重要な「決定論」と「LLM判断」の分離](#49-skill設計で重要な決定論とllm判断の分離)
- [50. 良いSKILL.mdの構造例](#50-良いskillmdの構造例)
- [51. 悪いSKILL.md例](#51-悪いskillmd例)
- [52. 改善するとこうなる](#52-改善するとこうなる)
- [53. Skillに「自分で改善し続けろ」と書く場合の注意](#53-skillに自分で改善し続けろと書く場合の注意)
- [54. Skillを評価する](#54-skillを評価する)
- [55. Fresh Sessionで評価する](#55-fresh-sessionで評価する)
- [56. SKILL.mdがロードされた後のライフサイクル](#56-skillmdがロードされた後のライフサイクル)
- [57. /compactとSkill](#57-compactとskill)
- [58. 推奨テンプレート：汎用Task Skill](#58-推奨テンプレート汎用task-skill)
- [59. 推奨テンプレート：Knowledge / Reference Skill](#59-推奨テンプレートknowledge--reference-skill)
- [60. 推奨テンプレート：危険操作Skill](#60-推奨テンプレート危険操作skill)
- [61. 推奨テンプレート：Subagent Skill](#61-推奨テンプレートsubagent-skill)
- [62. 推奨テンプレート：Script同梱Skill](#62-推奨テンプレートscript同梱skill)
- [63. SKILL.md設計のおすすめ順序](#63-skillmd設計のおすすめ順序)
- [64. 一番重要な設計原則](#64-一番重要な設計原則)
- [65. 最終チェックリスト](#65-最終チェックリスト)
- [66. まとめ](#66-まとめ)
- [参考資料](#参考資料)

<!-- /toc -->

---

# 1. SKILL.mdとは何か

Claude CodeのSkillは、Claudeに対して

- どんなタスクで使うか
- 何を知っておくべきか
- どんな手順で処理するか
- どのツールを使ってよいか
- ユーザーから直接呼べるか
- Claudeが自動的に呼んでよいか

などを定義する仕組みです。

Skillの中心となるファイルが、

```text
SKILL.md
```

です。

典型的な構成は以下です。

```text
my-skill/
├── SKILL.md
├── reference.md
├── examples.md
└── scripts/
    └── helper.py
```

この中で必須なのは、

```text
SKILL.md
```

です。

---

# 2. SKILL.mdの基本構造

`SKILL.md` は大きく2つの領域に分かれます。

```text
SKILL.md
│
├── ① YAML frontmatter
│      ↓
│   Skillのメタ情報・発火条件・実行設定
│
└── ② Markdown本文
       ↓
    Claudeが実際に従う指示
```

最小例:

```markdown
---
description: Webアプリのセキュリティレビューを行う。
---

# Security Review

コードベースを調査し、セキュリティ上の問題を検出する。

1. 認証を確認する
2. 認可を確認する
3. 入力validationを確認する
4. 脆弱性を報告する
```

---

# 3. YAML frontmatterとは

`SKILL.md` の先頭に書く設定部分です。

```yaml
---
name: security-review
description: Webアプリのセキュリティレビューを行う
---
```

重要な点として、開始の

```text
---
```

は**ファイルの1行目でなければなりません**。

例えばこれは正しいです。

```markdown
---
description: Security review
---

# Instructions
...
```

一方、

```markdown
# My Skill

---
description: Security review
---
```

のように前に何か書くと、frontmatterは一切読まれず、`---` マーカーを含む**ファイル全体がそのままSkill本文として扱われます**。

---

# 4. frontmatterは何のためにあるのか

frontmatterは大きく分けると次の役割を持ちます。

```text
発見・選択
├─ name
├─ description
├─ when_to_use
└─ paths

呼び出し方法
├─ disable-model-invocation
├─ user-invocable
├─ argument-hint
└─ arguments

実行環境
├─ model
├─ effort
├─ context
├─ agent
└─ background

Tool / Security
├─ allowed-tools
├─ disallowed-tools
├─ hooks
└─ shell

その他
├─ metadata
├─ license
└─ compatibility
```

すべて必須ではありません。

Anthropic公式では、**`description` を付けることが推奨**されています。

---

# 5. `name`

例:

```yaml
name: security-review
```

Skill一覧などに表示する名前です。

ただしPersonal Skill / Project Skillでは、実際に入力するコマンド名は基本的に**ディレクトリ名**で決まります。

例えば、

```text
.claude/skills/security-review/SKILL.md
```

なら、

```text
/security-review
```

です。

つまり、

```yaml
name: fancy-security-check
```

と書いていても、Personal / Project Skillの場合はフォルダ名が

```text
security-review
```

ならコマンドは基本的に

```text
/security-review
```

です。

`name` は主に表示名と考えると分かりやすいです。

---

# 6. `description` ― 最重要項目

例:

```yaml
description: >
  Webアプリのセキュリティレビューを行う。
  認証、認可、入力検証、XSS、CSRF、SQL Injection、
  Secret漏洩などの確認を依頼された場合に使用する。
```

Claudeが、

```text
このSkillを今使うべきか？
```

を判断する主要な情報です。

---

## 悪いdescription

```yaml
description: コードを改善する
```

問題:

```text
対象範囲が広すぎる
↓
いろいろな依頼にマッチしてしまう
↓
誤発火しやすい
```

---

## 良いdescription

```yaml
description: >
  Webアプリのセキュリティレビューを行う。
  ユーザーが脆弱性チェック、OWASP、認証・認可の安全性、
  XSS、CSRF、SQL Injection等の確認を依頼した場合に使用する。
```

良いdescriptionには、

```text
① Skillが何をするか
② どんな対象に使うか
③ どんなユーザー要求で発火させるか
```

を入れると安定します。

---

# 7. `when_to_use`

`description`を補足して、

```text
いつ発火させるか
```

をさらに具体化できます。

例:

```yaml
description: Webアプリのセキュリティレビューを行う。

when_to_use: >
  ユーザーが「脆弱性チェック」「security review」
  「OWASP」「認証の安全性」「XSS」「CSRF」
  「SQL Injection」などを依頼した場合。
```

概念的には、

```text
description
    ↓
何をするSkillか

when_to_use
    ↓
いつ使うSkillか
```

です。

`description` と `when_to_use` はSkill一覧用の情報として連結されます。

そのため、重要な発火条件は前半に書く方が安全です。

---

# 8. descriptionを長くしすぎない

Claude Codeは、Skill一覧としてClaudeに見せる際、

```text
description + when_to_use
```

を無制限には渡しません。

そのため、

```yaml
description: >
  このSkillは非常に高度な……
  背景として……
  歴史的には……
  設計思想として……
```

のような長い背景説明を書くより、

```yaml
description: >
  ROS2 + Franka実機制御のデバッグを行う。
  ノード、topic、controller、MoveIt、ros2_control、
  通信・制御周期の問題を調査するときに使用する。
```

のように**用途を先に書く**方が有効です。

---

# 9. `disable-model-invocation`

```yaml
disable-model-invocation: true
```

を付けると、

```text
Claude
↓
自動的には呼べない

ユーザー
↓
/skill-name で明示実行可能
```

になります。

---

## 使うべきSkill

副作用が大きい処理です。

例:

```text
deploy
release
git push
DB migration
production操作
外部サービスへの送信
データ削除
```

例:

```yaml
---
name: deploy
description: Productionへデプロイする
disable-model-invocation: true
---
```

これによりClaudeが、

```text
実装が終わったからproductionへdeployしておこう
```

と自発的にSkillを実行することを防げます。

---

# 10. `user-invocable`

```yaml
user-invocable: false
```

とすると、

```text
ユーザー
↓
直接 /skill-name で呼ばない

Claude
↓
必要に応じて呼べる
```

Skillになります。

例えば、

```text
legacy-system-context
project-domain-knowledge
internal-api-conventions
```

など、

```text
ユーザーが「実行するもの」ではないが、
Claudeには必要なときに読んでほしい知識
```

に向いています。

---

# 11. 呼び出し設定の整理

| 設定 | ユーザーから呼べる | Claudeが自動利用 |
|---|---:|---:|
| デフォルト | ○ | ○ |
| `disable-model-invocation: true` | ○ | × |
| `user-invocable: false` | × | ○ |

考え方としては、

```text
通常Skill
↓
両方OK

危険・副作用あり
↓
disable-model-invocation: true

内部知識用
↓
user-invocable: false
```

が分かりやすいです。

---

# 12. `argument-hint`

Skillに引数を渡す場合に、入力候補として表示するヒントです。

例:

```yaml
argument-hint: "[issue-number]"
```

あるいは、

```yaml
argument-hint: "[filename] [format]"
```

例えば、

```text
/fix-issue 123
```

のようなSkillで、

```text
何を入力すればよいか
```

をユーザーに伝えるために使います。

---

# 13. `$ARGUMENTS`

Skill呼び出し時の引数を本文に渡せます。

例:

```markdown
---
name: fix-issue
description: GitHub issueを修正する
disable-model-invocation: true
argument-hint: "[issue-number]"
---

GitHub issue $ARGUMENTS を修正する。

1. Issue内容を確認する
2. 関連コードを調査する
3. 修正する
4. Testを追加する
5. Testを実行する
```

ユーザーが、

```text
/fix-issue 123
```

とすると、Claudeには概念的に、

```text
GitHub issue 123 を修正する。
```

として渡されます。

---

# 14. 引数を位置で参照する

利用可能な形式:

```text
$ARGUMENTS
$ARGUMENTS[0]
$ARGUMENTS[1]

$0
$1
$2
```

例えば、

```markdown
Migrate $0 from $1 to $2.
```

に対して、

```text
/migrate-component SearchBar JavaScript TypeScript
```

なら、

```text
$0 = SearchBar
$1 = JavaScript
$2 = TypeScript
```

です。

---

# 15. 名前付き引数

frontmatterで、

```yaml
arguments:
  - issue
  - branch
```

と定義できます。

本文では、

```markdown
Fix issue $issue on branch $branch.
```

のように使えます。

例えば、

```text
/fix-issue 123 feature/auth
```

なら、

```text
$issue  → 123
$branch → feature/auth
```

です。

引数が多いSkillでは、`$0`, `$1` より名前付き引数の方が読みやすくなります。

---

# 16. `allowed-tools`

例:

```yaml
allowed-tools:
  - Read
  - Grep
  - Glob
```

Skill実行ターンで、指定Toolを事前許可（確認プロンプトのスキップ）できます。

事前許可するBashコマンドを絞り込むこともできます。

例:

```yaml
allowed-tools: Bash(git status *) Bash(git diff *)
```

これは、

```text
このSkill実行時のみ
↓
特定のTool / commandを許可
```

という意味です。

重要なのは、**Skill本文が後続ターンに残っても、この権限は次のユーザーメッセージで解除される**点です。

また `allowed-tools` は**使えるToolを制限するフィールドではありません**。記載しなかったToolも引き続き呼び出し可能で、それらには通常のpermissions設定が適用されます。使わせたくないToolがある場合は次の `disallowed-tools` を使います。

---

# 17. `allowed-tools`を安易に広げない

例えば、

```yaml
allowed-tools: Bash(*)
```

のような非常に広い許可は、Skillの便利さは増えますが安全性が下がります。

特にProject Skillはrepositoryにコミットされることがあるため、

```text
Skillをrepoから取得
↓
allowed-toolsを十分確認しない
↓
Skillが広いTool権限を持つ
```

というリスクがあります。

推奨は、

```text
必要最小限のToolだけ許可
```

です。

---

# 18. `disallowed-tools`

Skill実行中に使わせたくないToolを指定できます。

例:

```yaml
disallowed-tools:
  - AskUserQuestion
```

あるいは、

```yaml
disallowed-tools:
  - Bash
  - Write
```

例えば完全なread-only分析Skillなら、

```yaml
allowed-tools:
  - Read
  - Grep
  - Glob

disallowed-tools:
  - Write
  - Edit
  - Bash
```

のようにすることもできます。

---

# 19. `model`

Skill実行ターンだけ、使用モデルを変更できます。

例:

```yaml
model: opus
```

または、

```yaml
model: inherit
```

などです。

考え方:

```text
軽い分類Skill
↓
軽量モデル

難しいarchitecture review
↓
高性能モデル
```

という使い分けも可能です。

ただし組織側で利用可能モデルが制限されている場合は、その制限が優先されます。

---

# 20. `effort`

推論強度をSkill単位で指定できます。

例:

```yaml
effort: high
```

利用可能なレベルはモデルに依存しますが、Claude Codeでは例として、

```text
low
medium
high
xhigh
max
```

などがあります。

例えば、

```yaml
---
name: architecture-review
description: システム設計の詳細レビュー
effort: high
---
```

のようにできます。

---

# 21. `context: fork`

かなり重要な設定です。

```yaml
context: fork
```

を指定すると、そのSkillを**独立したSubagent context**で実行できます。

通常:

```text
Main Conversation
      │
      ▼
SKILL.mdロード
      │
      ▼
Main context内で実行
```

`context: fork`:

```text
Main Conversation
      │
      ├──────────────┐
      │              ▼
      │        Forked Subagent
      │              │
      │          SKILL.md
      │              │
      │          調査・実行
      │              │
      ◀───── 結果 ───┘
```

大規模な調査や独立作業で便利です。

---

# 22. `agent`

`context: fork` と一緒に使います。

例:

```yaml
context: fork
agent: Explore
```

例えば、

```markdown
---
name: deep-research
description: コードベースを詳細調査する
context: fork
agent: Explore
---

$ARGUMENTSについて調査する。

1. Glob / Grepで関連ファイルを探す
2. 実装を読む
3. 依存関係を整理する
4. file path付きで結果を報告する
```

という構成です。

利用できるagentには、

```text
Explore
Plan
general-purpose
Custom Subagent
```

などがあります。

`agent` を省略した場合は `general-purpose` が使われます。

なお、組み込みの `Explore` / `Plan` はCLAUDE.mdを読み込まないため、この場合Subagentが見るのはSKILL.md本文とagent自身のsystem promptだけになります。

---

# 23. `background`

`context: fork` 時の実行方法を制御します。

例:

```yaml
context: fork
background: false
```

概念的には、

```text
background: true
↓
Subagentを非同期的な作業として走らせる

background: false
↓
そのSubagentの結果を待ってから処理を続ける
```

です。

デフォルトは `background: true` で、`context: fork` のSkillはバックグラウンドで実行され、完了時に結果が会話へ戻ります。

ただし以下のような場合は、`background: false` を書かなくてもClaude Codeは結果を待ちます。

```text
-p フラグなどの非対話モード / Agent SDK
同じSkillの前回の実行がまだ継続中
CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1
```

またバックグラウンド実行時はSubagentのToolセットが狭くなるため、幅広いToolが必要な手順では `background: false` を指定します。

---

# 24. `paths`

Skillを特定ファイル群に限定できます。

例:

```yaml
paths:
  - "frontend/**"
  - "**/*.tsx"
  - "**/*.css"
```

この場合、

```text
frontend
React
CSS
```

に関連するファイルを扱っている場合に自動発火候補になります。

globの書き方には注意が必要で、`*.tsx` は**project root直下のファイルだけ**にマッチします。サブディレクトリも含めたい場合は `**/*.tsx` のように書きます。

モノレポでは特に便利です。

例:

```yaml
---
name: frontend-review
description: Frontendコードをレビューする
paths:
  - "apps/web/**"
  - "**/*.tsx"
  - "**/*.css"
---
```

これによりBackend作業中の誤発火を抑えやすくなります。

---

# 25. `shell`

Skill本文に埋め込むshell commandで使用するshellを指定できます。

例:

```yaml
shell: bash
```

または、

```yaml
shell: powershell
```

Windows環境などで有用です。

---

# 26. `hooks`

Skillが実行された時にHookを登録することもできます。

```yaml
hooks:
  ...
```

これは高度な機能です。

例えば、

```text
Skill実行
↓
特定Hook登録
↓
Tool実行前後を検査
↓
ルールを強制
```

のような設計が可能です。

単なる「Claudeへのお願い」ではなく、

```text
Hookで決定論的に制御
```

したい場合に使います。

---

# 27. `metadata`

独自ツール向けのメタデータを格納できます。

例:

```yaml
metadata:
  category: security
  owner: platform-team
  version: "1.3"
```

Claude Code自体は基本的にこの中身を実行制御には使いません。

独自Skill managerやcatalogなどで利用する情報です。

---

# 28. `license`

例:

```yaml
license: MIT
```

Agent Skills標準に含まれるフィールドです。

Skillを公開・配布する場合に利用できます。

---

# 29. `compatibility`

Skillの前提環境を示すための情報です。

例:

```yaml
compatibility: Requires git, Python 3.12, and Linux or WSL.
```

Claude Codeでは情報として受け付けますが、このフィールド自体が環境チェックを自動実行するわけではありません。

必要ならSkill本文で、

```text
最初に環境を確認する
```

処理を明示します。

---

# 30. frontmatter全体例

高度な例:

```yaml
---
name: security-review
description: >
  Webアプリのセキュリティレビューを行う。
  認証、認可、入力validation、OWASP系脆弱性、
  secret、dependencyの安全性を確認するときに使用する。

when_to_use: >
  ユーザーが脆弱性チェック、security review、
  OWASP、認証・認可の安全性確認を依頼した場合。

argument-hint: "[target-path]"

arguments:
  - target

allowed-tools:
  - Read
  - Grep
  - Glob

disallowed-tools:
  - Write
  - Edit

effort: high

metadata:
  category: security
---
```

全項目を毎回書く必要はありません。

むしろ、

```text
必要な設定だけ書く
```

方が管理しやすいです。

---

# 31. Markdown本文には何を書くべきか

frontmatterの後に、Claudeが実際に従う指示を書きます。

おすすめ構造:

```text
# Purpose

# Scope

# Procedure

# Verification

# Output

# Constraints

# Additional resources
```

必ずこの見出し名である必要はありません。

重要なのは、

```text
目的
↓
対象範囲
↓
実行手順
↓
検証方法
↓
完了条件
↓
出力形式
```

が明確になっていることです。

---

# 32. `Purpose`

Skillの目的を1〜数文で書きます。

例:

```markdown
# Purpose

コードベースのセキュリティリスクを検出し、
具体的な修正方法を提示する。
```

ここでは長い背景説明は不要です。

---

# 33. `Scope`

対象範囲を限定します。

例:

```markdown
# Scope

対象:

- Authentication
- Authorization
- API endpoints
- Input validation
- Secrets
- Dependencies

対象外:

- Production penetration testing
- DoS負荷試験
- 外部サービスへの攻撃
```

Skillの暴走や責務拡大を防ぐために有効です。

---

# 34. `Procedure`

Skillの中心です。

Claudeが実行すべき手順を順番に書きます。

例:

```markdown
# Procedure

1. Repository構成を確認する。
2. Entry pointを特定する。
3. Authentication実装を確認する。
4. Authorization境界を確認する。
5. User inputの流れを追跡する。
6. Injection脆弱性を確認する。
7. Secret漏洩を確認する。
8. Dependenciesを確認する。
9. Findingsをseverity別に分類する。
```

曖昧な、

```text
適切に確認する
注意深くレビューする
良い感じに修正する
```

より、

```text
何を、どの順序で、何を確認するか
```

を具体化した方が再現性が高くなります。

---

# 35. 分岐条件を書く

Agent Skillでは単純なチェックリストだけでなく、

```text
条件 → 行動
```

を明示すると強くなります。

例:

```markdown
If authentication uses JWT:

- Check signature validation.
- Check expiration handling.
- Check algorithm restrictions.

If session cookies are used:

- Check Secure.
- Check HttpOnly.
- Check SameSite.
```

日本語でも、

```markdown
認証がJWTの場合:

- 署名検証を確認する。
- expirationを確認する。
- algorithm制約を確認する。

Cookie sessionの場合:

- Secureを確認する。
- HttpOnlyを確認する。
- SameSiteを確認する。
```

のように書けます。

---

# 36. `Verification`

Agentらしく動かすには非常に重要です。

```markdown
# Verification

変更を行った場合は、以下を実行する。

1. Unit test
2. Type check
3. Lint
4. Build
5. 関連するintegration test

失敗した場合:

1. Errorを分析する。
2. 原因を特定する。
3. 修正する。
4. 再度testする。
```

これにより、

```text
実装して終わり
```

ではなく、

```text
実装
↓
検証
↓
失敗
↓
修正
↓
再検証
```

というAgent loopを誘導できます。

---

# 37. `Output`

最終回答形式を指定します。

例:

```markdown
# Output

以下の形式で報告する。

## Summary

全体評価を3〜5行で記載する。

## Findings

各問題について:

- Severity
- File
- Line
- Problem
- Risk
- Recommended fix

## Verification

- 実行したtest
- 結果
- 未確認事項
```

出力フォーマットを決めておくと、Skillを何度使っても結果が比較しやすくなります。

---

# 38. `Constraints`

Claudeに守らせたい制約を書きます。

例:

```markdown
# Constraints

- Production DBを変更しない。
- Secretを表示しない。
- テスト目的で外部システムを攻撃しない。
- ユーザーデータを削除しない。
- 不明な場合は推測せず、コードから確認する。
```

安全性や品質に関わるものは明示しておく価値があります。

ただし、本当に絶対に防ぎたい操作はSkill本文だけでなく、

```text
permissions
hooks
sandbox
```

などでも強制した方が安全です。

---

# 39. Reference SkillとTask Skill

AnthropicはSkill本文を大きく2タイプとして考えると整理しやすいとしています。

---

## Reference Skill

知識・規約・設計パターンをClaudeに与えるSkill。

例:

```markdown
---
name: api-conventions
description: このコードベースのAPI設計規約
user-invocable: false
---

API endpointを書く場合:

- RESTful namingを使用する。
- Error response形式を統一する。
- Request validationを必須にする。
- API versioning規則に従う。
```

用途:

```text
Coding conventions
Architecture rules
Domain knowledge
Legacy system knowledge
Style guide
```

---

## Task Skill

明確な処理を実行させるSkill。

例:

```markdown
---
name: deploy
description: Productionへdeployする
disable-model-invocation: true
---

Deployを実行する。

1. Testを実行する。
2. Buildする。
3. Deploymentを実行する。
4. Health checkする。
5. Deployment結果を報告する。
```

用途:

```text
Deploy
Commit
Migration
Code generation
Release
Review
Testing
```

---

# 40. SKILL.mdを巨大化させない

Anthropic公式は、

```text
SKILL.mdは500行未満に保ち、
詳細資料は別ファイルへ分ける
```

ことを推奨しています。

理由:

```text
SKILL.md発火
↓
本文がcontextに入る
↓
後続ターンにも残る
↓
長いほどtokenコストが継続する
```

からです。

したがって、

```text
SKILL.md
↓
概要 / 判断 / Procedure

reference.md
↓
詳細仕様

examples.md
↓
具体例

scripts/
↓
実行コード
```

という構造が有効です。

---

# 41. 補助ファイルを使う

例:

```text
security-review/
├── SKILL.md
├── checklist.md
├── owasp-reference.md
├── examples.md
└── scripts/
    └── dependency_check.py
```

SKILL.mdから、

```markdown
# Additional resources

- 詳細チェックリストが必要な場合は [checklist.md](checklist.md) を読む。
- OWASP詳細が必要な場合は [owasp-reference.md](owasp-reference.md) を読む。
- 出力例は [examples.md](examples.md) を参照する。
```

と書きます。

Claudeは必要な場合だけ補助ファイルを読みます。

---

# 42. 補助ファイルを分ける利点

巨大なSkill:

```text
SKILL.md
3000行
```

より、

```text
SKILL.md
200行

reference.md
800行

examples.md
500行
```

の方が効率的です。

実行時:

```text
Skill発火
↓
SKILL.mdのみロード
↓
必要？
  │
  ├─ YES → reference.mdを読む
  │
  └─ NO  → 読まない
```

となるためです。

---

# 43. `${CLAUDE_SKILL_DIR}`

Skillフォルダ自身を参照できます。

例:

```markdown
Run:

`${CLAUDE_SKILL_DIR}/scripts/check.py`
```

Skillを、

```text
~/.claude/skills/security-review/
```

に置いていても、

```text
project/.claude/skills/security-review/
```

に置いていても、正しいSkillディレクトリへ展開されます。

Skill内にscriptsを同梱する場合に非常に便利です。

---

# 44. `${CLAUDE_PROJECT_DIR}`

現在のproject rootを参照できます。

例:

```markdown
Read:

`${CLAUDE_PROJECT_DIR}/pyproject.toml`
```

Skill自身の場所ではなく、

```text
今作業しているproject
```

を参照したい場合に使います。

---

# 45. その他の主な変数

```text
${CLAUDE_SESSION_ID}
${CLAUDE_EFFORT}
${CLAUDE_SKILL_DIR}
${CLAUDE_PROJECT_DIR}
${CLAUDE_PLUGIN_ROOT}
${CLAUDE_PLUGIN_DATA}
```

用途例:

```text
Session ID
↓
log filename

Skill directory
↓
Skill付属script

Project directory
↓
Repository内のfile

Plugin root
↓
Plugin共有resource
```

---

# 46. Dynamic Context Injection

Skill本文からshell commandを実行し、その結果をSkill promptへ埋め込めます。

例:

```markdown
## Current changes

!`git diff HEAD`
```

Skillが実行されると、

```text
git diff HEAD
↓
Claude Codeが先に実行
↓
実行結果をSKILL.md内へ挿入
↓
Claudeが結果込みでSkillを受け取る
```

となります。

例えば、

```markdown
---
name: summarize-changes
description: 現在のGit変更内容を要約する
---

## Current diff

!`git diff HEAD`

## Instructions

上記diffを要約し、リスクを報告する。
```

というSkillを作れます。

---

# 47. Dynamic Context Injectionの注意

Shell commandはClaudeがSkill本文を読む前に実行されます。

したがって、

```text
Skill実行
↓
Shell command失敗
↓
Skill invocation自体が失敗
```

するケースがあります。

また、

```text
外部入力
↓
shell command
```

を安易に組み合わせると危険です。

必要最小限のコマンドだけ使う方が安全です。

---

# 48. Scriptを直接本文に大量に書かない

例えばSKILL.mdに、

```text
Python 300行
```

を書くより、

```text
scripts/check.py
```

として保存し、

```markdown
Run:

`${CLAUDE_SKILL_DIR}/scripts/check.py`
```

とする方がよいです。

理由:

```text
SKILL.md
↓
Claudeへのinstruction

scripts/
↓
実際のdeterministic処理
```

と責務を分離できるためです。

---

# 49. Skill設計で重要な「決定論」と「LLM判断」の分離

Claudeに向いている処理:

```text
コード理解
設計判断
分類
レビュー
説明
原因分析
修正案
```

Script / Hookに向いている処理:

```text
lint
format
schema validation
secret scanning
特定コマンドの強制
出力形式の機械検証
危険操作のblock
```

理想:

```text
LLM
↓
判断・計画

Script
↓
機械処理

LLM
↓
結果分析

Hook
↓
安全制約を強制
```

です。

---

# 50. 良いSKILL.mdの構造例

```markdown
---
name: security-review
description: >
  Webアプリのセキュリティレビューを行う。
  認証、認可、入力validation、OWASP系脆弱性、
  secret、dependencyの確認時に使用する。

when_to_use: >
  ユーザーが脆弱性チェック、security review、
  OWASP、認証・認可の安全性確認を依頼した場合。

allowed-tools:
  - Read
  - Grep
  - Glob

effort: high
---

# Purpose

コードベースのセキュリティリスクを検出し、
具体的な修正案を提示する。

# Scope

確認対象:

- Authentication
- Authorization
- Input validation
- Injection
- XSS / CSRF / SSRF
- Secrets
- Dependencies

# Procedure

1. Repository構成を確認する。
2. Entry pointと外部入力経路を特定する。
3. Authenticationを確認する。
4. Authorization boundaryを確認する。
5. Inputのsource-to-sinkを追跡する。
6. Injection系脆弱性を確認する。
7. Secret漏洩を確認する。
8. Dependencyリスクを確認する。
9. Findingsをseverity別に分類する。

# Verification

Findingごとに、実際のコード上の根拠を確認する。
推測だけで脆弱性と断定しない。

# Output

各Findingについて:

- Severity
- File
- Line
- Problem
- Evidence
- Risk
- Recommended fix

最後に、

- Critical数
- High数
- Medium数
- Low数
- 未確認事項

をまとめる。

# Constraints

- Production systemへ攻撃を行わない。
- User dataを削除しない。
- Secretの値を出力しない。

# Additional resources

詳細チェック項目が必要な場合は [checklist.md](checklist.md) を読む。
```

---

# 51. 悪いSKILL.md例

```markdown
---
description: コードを良くする
---

コードをよく見て、
問題があれば適切に直してください。
できるだけ品質を高くしてください。
必要に応じてテストしてください。
```

問題:

```text
Skillの責務が曖昧
発火条件が曖昧
Procedureがない
Verificationが曖昧
完了条件がない
出力形式がない
```

Claudeの自由度が高すぎます。

---

# 52. 改善するとこうなる

```markdown
---
name: code-review
description: >
  実装済みコードの品質レビューを行う。
  correctness、maintainability、error handling、
  test不足を確認するときに使用する。
---

# Procedure

1. 変更差分を確認する。
2. correctnessを確認する。
3. Error handlingを確認する。
4. 重複コードを確認する。
5. API互換性を確認する。
6. Test coverageを確認する。

# Output

問題を以下で分類する。

- Blocking
- Important
- Suggestion

各Findingにfile pathと根拠を付ける。
```

これだけでも再現性が大きく上がります。

---

# 53. Skillに「自分で改善し続けろ」と書く場合の注意

例えば、

```markdown
失敗した場合、このSKILL.mdを自動更新して学習すること。
```

のような設計は慎重に扱うべきです。

理由:

```text
一度の特殊ケース
↓
Skillへ恒久ルール追加
↓
次の別ケースで誤ったルールが適用
↓
Skillが劣化
```

する可能性があるからです。

推奨:

```text
失敗
↓
原因分析
↓
改善候補を提案
↓
test / eval
↓
改善が一般化可能か確認
↓
SKILL.md更新
```

です。

Skillの自己修正は、

```text
自動学習
```

ではなく、

```text
versioned configuration improvement
```

として扱った方が安全です。

---

# 54. Skillを評価する

Skillが存在するだけでは、

```text
正しく機能している
```

とは限りません。

最低でも、

```text
① 発火精度
② 実行品質
```

を別々に評価します。

---

## 発火精度

Should trigger:

```text
このWebアプリの脆弱性をチェックして
OWASP観点でレビューして
認証実装が安全か確認して
```

Should NOT trigger:

```text
CSSの余白を直して
READMEを書いて
画像サイズを変更して
```

---

## 実行品質

Skillあり / なしで比較します。

```text
同じtask
├─ Skill OFF
└─ Skill ON

比較
├─ 正答率
├─ 見落とし数
├─ 誤検知
├─ token
├─ 実行時間
└─ 再現性
```

---

# 55. Fresh Sessionで評価する

Skillを書いた直後の同一セッションでは、

```text
Skillを作った会話内容
```

自体がcontextに残っているため、本当にSkillだけで正しく動いているのか判断しにくくなります。

そのため、

```text
新しいClaude Code session
↓
Skillだけ利用可能
↓
テストprompt
```

で確認する方が正確です。

---

# 56. SKILL.mdがロードされた後のライフサイクル

Skillが発火すると、

```text
SKILL.md
↓
render
↓
conversation contextへ追加
↓
後続ターンにも残る
```

という動きになります。

Claude Codeは毎ターンSKILL.mdをディスクから読み直すわけではありません。

したがって、

```text
この回答だけで守る
```

より、

```text
このタスク全体で守るルール
```

として書く方が適しています。

---

# 57. `/compact`とSkill

Context compaction後も、最近使用したSkillは一定のtoken budget内で再添付されます。

ただし、

```text
大量Skill
巨大SKILL.md
長いsession
```

では古いSkillや後半部分が落ちる可能性があります。

その意味でも、

```text
重要な指示は前半
SKILL.mdは簡潔
詳細は補助ファイル
```

が重要です。

---

# 58. 推奨テンプレート：汎用Task Skill

```markdown
---
name: my-task
description: >
  このSkillが何をするか。
  どんな依頼のときに使うか。

when_to_use: >
  具体的なtrigger条件を書く。

argument-hint: "[target]"
---

# Purpose

このSkillの目的。

# Scope

対象:
- ...

対象外:
- ...

# Inputs

入力として確認するもの:
- ...

# Procedure

1. ...
2. ...
3. ...

# Decision Rules

条件Aの場合:
- ...

条件Bの場合:
- ...

# Verification

1. ...
2. ...

失敗した場合:
1. 原因を分析する。
2. 修正する。
3. 再検証する。

# Output

以下を報告する:
- ...
- ...

# Constraints

- ...
- ...

# Additional resources

必要な場合:
- [reference.md](reference.md)
- [examples.md](examples.md)
```

---

# 59. 推奨テンプレート：Knowledge / Reference Skill

```markdown
---
name: project-conventions
description: >
  このプロジェクト固有の設計規約。
  API、architecture、naming、error handlingの実装時に参照する。

user-invocable: false
---

# Architecture

- ...

# API conventions

- ...

# Error handling

- ...

# Naming

- ...

# Testing conventions

- ...

# References

詳細仕様が必要な場合は [reference.md](reference.md) を読む。
```

---

# 60. 推奨テンプレート：危険操作Skill

```markdown
---
name: production-deploy
description: Production環境へdeployする。
disable-model-invocation: true
argument-hint: "[environment]"
---

# Preconditions

実行前に必ず:

1. Git statusを確認する。
2. Testを実行する。
3. Buildを実行する。
4. Target environmentを確認する。

# Procedure

1. ...
2. ...
3. ...

# Verification

1. Deployment statusを確認する。
2. Health checkを確認する。
3. Error logを確認する。

# Abort Conditions

以下の場合はdeployしない:

- Test failure
- Build failure
- Target不明
- Secret不足
- Production設定の不整合

# Output

- Deployment target
- Version
- Result
- Verification result
```

---

# 61. 推奨テンプレート：Subagent Skill

```markdown
---
name: deep-code-research
description: >
  コードベース全体を横断して詳細調査する。
  architecture、依存関係、既存実装の調査時に使用する。

context: fork
agent: Explore
effort: high
---

Research `$ARGUMENTS` thoroughly.

# Procedure

1. Globで関連ファイルを探索する。
2. Grepでsymbolと参照箇所を探す。
3. 主要実装を読む。
4. Dependency flowを追跡する。
5. Edge caseを確認する。
6. 根拠となるfile pathを記録する。

# Output

- Summary
- Relevant files
- Architecture
- Data flow
- Risks
- Open questions
```

---

# 62. 推奨テンプレート：Script同梱Skill

Directory:

```text
dependency-check/
├── SKILL.md
└── scripts/
    └── check_dependencies.py
```

SKILL.md:

```markdown
---
name: dependency-check
description: Dependencyの安全性と更新状況を確認する。
---

# Procedure

1. Dependency fileを確認する。
2. 以下のscriptを実行する。

`${CLAUDE_SKILL_DIR}/scripts/check_dependencies.py`

3. 結果を分析する。
4. Critical / High riskを優先して報告する。
```

---

# 63. SKILL.md設計のおすすめ順序

新しいSkillを書くときは、いきなり長文を書かず、

```text
① 責務を1文で決める
↓
② 発火条件を書く
↓
③ Scopeを決める
↓
④ Procedureを書く
↓
⑤ Verificationを書く
↓
⑥ Outputを書く
↓
⑦ Safety / Constraintsを書く
↓
⑧ 長くなった部分をreferenceへ分離
↓
⑨ 発火test
↓
⑩ Skillあり / なしで評価
```

の順が安定します。

---

# 64. 一番重要な設計原則

SKILL.mdは、

```text
「Claudeに大量の知識を詰め込むファイル」
```

ではなく、

```text
「ある種類の仕事をClaudeが再現性高く処理するための
最小限の実行仕様書」
```

と考えるのが適切です。

理想的な構造:

```text
Frontmatter
↓
いつ・誰が・どの環境で使うか

Purpose / Scope
↓
何をするか

Procedure
↓
どう進めるか

Decision Rules
↓
条件ごとにどう判断するか

Verification
↓
正しくできたかどう確認するか

Output
↓
どう報告するか

Constraints
↓
何をしてはいけないか

Supporting Files
↓
必要時だけ読む詳細情報

Scripts / Hooks
↓
LLM任せにしない決定論的処理
```

---

# 65. 最終チェックリスト

新しい`SKILL.md`を作ったら確認します。

```text
[ ] Skillの責務は1つに絞られている
[ ] descriptionに「何をするか」がある
[ ] description / when_to_useに「いつ使うか」がある
[ ] 誤発火しそうな広すぎる表現がない
[ ] Procedureが具体的
[ ] Verificationがある
[ ] 完了条件が分かる
[ ] Output形式が明確
[ ] 副作用Skillはdisable-model-invocationを検討した
[ ] allowed-toolsが広すぎない
[ ] 危険操作をSkill本文だけで防ごうとしていない
[ ] 長いreferenceを別ファイルへ分けた
[ ] SKILL.mdが過度に巨大化していない
[ ] Should-trigger promptで確認した
[ ] Should-not-trigger promptで確認した
[ ] Fresh Sessionで動作確認した
```

---

# 66. まとめ

`SKILL.md` の設計を最も簡潔にまとめると、

```text
SKILL.md
│
├── YAML Frontmatter
│   │
│   ├─ name
│   ├─ description
│   ├─ when_to_use
│   ├─ invocation control
│   ├─ arguments
│   ├─ tools
│   ├─ model / effort
│   └─ context / agent
│   │
│   ▼
│ 「いつ・誰が・どう実行するか」
│
└── Markdown Body
    │
    ├─ Purpose
    ├─ Scope
    ├─ Procedure
    ├─ Decision Rules
    ├─ Verification
    ├─ Output
    ├─ Constraints
    └─ Additional Resources
    │
    ▼
  「実際に何をするか」
```

そしてSkill directory全体としては、

```text
my-skill/
│
├── SKILL.md
│     ↓
│   実行仕様 / navigation
│
├── reference.md
│     ↓
│   詳細知識
│
├── examples.md
│     ↓
│   具体例
│
└── scripts/
      ↓
    決定論的処理
```

という役割分担にすると、Claude CodeのSkillを拡張しやすく、コンテキスト効率も良く、誤発火や挙動のばらつきも抑えやすくなります。

---

# 参考資料

Anthropic公式 Claude Code Docs:

- Extend Claude with skills  
  https://code.claude.com/docs/en/skills

特に確認するとよい項目:

- Create your first skill
- Types of skill content
- Frontmatter reference
- Available string substitutions
- Add supporting files
- Control who invokes a skill
- Skill content lifecycle
- Pre-approve tools for a skill
- Pass arguments to skills
- Inject dynamic context
- Run skills in a subagent
- Evaluate and iterate on a skill
