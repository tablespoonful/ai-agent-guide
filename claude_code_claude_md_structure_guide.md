# Claude Code：CLAUDE.mdファイルの構成（中身）についての解説

> 対象: Claude Code の `CLAUDE.md`  
> 更新基準: 2026-08-29 時点のAnthropic公式ドキュメント  
> 目的: `CLAUDE.md` に「何を・どの粒度で・どのように書けばよいか」を、設計意図も含めて理解する

---

# 1. CLAUDE.mdとは何か

`CLAUDE.md` は、Claude Codeに対して

- このプロジェクトは何か
- どこに何があるか
- どのルールを常に守るべきか
- どのコマンドでbuild / testするか
- どんなarchitecture方針を採用しているか
- どんな操作を避けるべきか
- どのSkillやworkflowを使うべきか

などを、**セッションをまたいで継続的に伝えるMarkdownファイル**です。

最も重要な理解は、

```text
CLAUDE.md
=
Claude Codeへの「常設コンテキスト / 常設指示」
```

ということです。

単なるREADMEでも、設定ファイルでもありません。

---

# 2. CLAUDE.mdは「強制設定」ではない

`CLAUDE.md` はClaudeのcontextに入る**行動指示**です。

つまり、

```text
CLAUDE.md
↓
Claudeが読む
↓
判断時に参照する
↓
行動へ反映する
```

という仕組みです。

一方で、

```text
permissions
hooks
sandbox
managed settings
```

のようなクライアント側の設定は、

```text
Claudeの判断とは無関係に
実行を許可・拒否する
```

ことができます。

したがって、

```markdown
絶対に rm -rf を実行するな
```

とCLAUDE.mdに書くだけでは、**技術的な強制禁止にはなりません**。

本当に防止したい場合は、

```text
CLAUDE.md
+
permissions / hooks / sandbox
```

を併用します。

---

# 3. CLAUDE.mdは毎セッション読み込まれる

Claude Codeは新しいセッションごとにfresh contextから始まります。

そのとき、

```text
CLAUDE.md
↓
session contextへロード
```

されます。

イメージ:

```text
Claude Code起動
      │
      ▼
System instructions
      │
      ▼
Settings / Permissions
      │
      ▼
CLAUDE.md
      │
      ▼
Auto Memory
      │
      ▼
Skills metadata等
      │
      ▼
ユーザーPrompt
```

したがってCLAUDE.mdに書く内容は、

```text
毎セッションClaudeに知っていてほしいこと
```

に限定するのが基本です。

---

# 4. CLAUDE.mdに向いている情報

Anthropic公式では、例えば次のような内容が適しています。

```text
Build commands
Test commands
Coding conventions
Project architecture
Project layout
Naming conventions
Common workflows
Always do X rules
```

具体例:

```markdown
# Build and Test

- Install dependencies with `npm ci`.
- Run unit tests with `npm test`.
- Run type checks with `npm run typecheck`.
- Run the production build with `npm run build`.
```

あるいは、

```markdown
# Architecture

- API handlers live under `src/api/handlers/`.
- Business logic belongs in `src/services/`.
- Database access must go through `src/repositories/`.
- UI components must not access the database layer directly.
```

のような内容です。

---

# 5. CLAUDE.mdに向いていない情報

以下はCLAUDE.mdへ詰め込みすぎない方がよいです。

```text
長大な手順書
特定タスクでしか使わない手順
一部ディレクトリだけのルール
巨大なAPI仕様
長い参考資料
一時的な作業メモ
秘密情報
厳密に強制したい禁止事項
```

これらは別の仕組みへ分離します。

```text
長いタスク手順
↓
Skill

特定pathだけのルール
↓
.claude/rules/

詳細資料
↓
別Markdown + 必要時Read

一時的な個人メモ
↓
CLAUDE.local.md / Auto Memory

強制禁止
↓
permissions / hooks / sandbox
```

---

# 6. CLAUDE.mdの基本構造

`CLAUDE.md` は普通のMarkdownです。

`SKILL.md`のようにfrontmatterが必須という仕組みではありません。

最小例:

```markdown
# Project Overview

This is a TypeScript web application.

# Commands

- Install: `npm ci`
- Test: `npm test`
- Typecheck: `npm run typecheck`
- Build: `npm run build`

# Coding Rules

- Keep TypeScript strict mode enabled.
- Do not use `any` without justification.
- Preserve API compatibility.

# Verification

After code changes:

1. Run relevant tests.
2. Run typecheck.
3. Run build when production code changes.
```

---

# 7. おすすめの全体構成

実運用では、次の構造が扱いやすいです。

```text
# Project Overview

# Repository Structure

# Architecture

# Commands

# Coding Rules

# Testing

# Verification

# Git / Change Policy

# Security / Safety

# Skill Routing

# Do Not

# References
```

すべて必要ではありません。

プロジェクトに必要な項目だけ残します。

---

# 8. `Project Overview`

Claudeがコードベースを理解するための最小限の説明です。

例:

```markdown
# Project Overview

This repository contains a SaaS web application.

Stack:

- Next.js
- TypeScript
- PostgreSQL
- Prisma
- Playwright
- Vitest
```

ここで長い事業説明を書く必要はありません。

重要なのは、

```text
Claudeがコードからすぐ判断できないが、
全タスクで知っていた方がよい情報
```

です。

---

# 9. Project Overviewに書きすぎない

悪い例:

```markdown
# Project Overview

2019年にこのプロジェクトは開始され……
当初はVueを使っていたが……
その後2021年に……
チーム再編により……
```

こうした歴史が現在の作業に影響しないなら、常時contextに入れる価値は低いです。

良い例:

```markdown
# Project Overview

This is a Next.js SaaS application.

The current architecture is a modular monolith.
Legacy code under `legacy/` is read-only unless explicitly requested.
```

です。

---

# 10. `Repository Structure`

Claudeが毎回探索しなくてもよい主要ディレクトリを伝えます。

例:

```markdown
# Repository Structure

- `src/app/`: Next.js routes
- `src/components/`: reusable UI components
- `src/services/`: business logic
- `src/repositories/`: database access
- `tests/`: integration tests
- `e2e/`: Playwright tests
```

特にモノレポでは有効です。

---

# 11. Repository Structureの目的

目的は、

```text
全ファイル一覧を書くこと
```

ではありません。

Claudeに、

```text
「まずどこを見るべきか」
```

を教えることです。

悪い例:

```text
全フォルダ・全ファイルを列挙
```

良い例:

```text
主要な境界だけ示す
```

です。

---

# 12. `Architecture`

重要な設計上の境界を書きます。

例:

```markdown
# Architecture

- HTTP handlers should only parse requests and return responses.
- Business logic belongs in `src/services/`.
- Database access belongs in `src/repositories/`.
- Cross-module imports must go through each module's public interface.
```

特に、

```text
コードを見ただけでは暗黙的で分かりにくい設計ルール
```

を書く価値があります。

---

# 13. Architectureには「禁止境界」も書く

例:

```markdown
# Architecture Boundaries

- UI components must not import database code.
- API handlers must not contain raw SQL.
- Do not create direct imports between `billing` and `analytics`.
- Shared code belongs under `src/shared/`.
```

このようなルールはClaudeが設計を崩すのを防ぐのに有効です。

---

# 14. `Commands`

Claudeが毎回package.jsonやMakefileを探さなくてもよいよう、主要コマンドを明示します。

例:

```markdown
# Commands

- Install dependencies: `npm ci`
- Dev server: `npm run dev`
- Unit tests: `npm test`
- Single test: `npm test -- <path>`
- Type check: `npm run typecheck`
- Lint: `npm run lint`
- Production build: `npm run build`
```

---

# 15. コマンドは具体的に書く

悪い例:

```markdown
- テストすること
```

良い例:

```markdown
- Run `npm test` for unit tests.
- Run `npm run typecheck` after TypeScript changes.
```

Claudeにとって、

```text
検証可能な指示
```

になっていることが重要です。

---

# 16. `Coding Rules`

コード規約を書きます。

例:

```markdown
# Coding Rules

- Use 2-space indentation.
- Keep TypeScript strict mode enabled.
- Prefer explicit return types for exported functions.
- Avoid `any`; use `unknown` and narrow types instead.
- Reuse existing utilities before creating new helpers.
```

---

# 17. 曖昧なルールを避ける

悪い例:

```markdown
- 綺麗なコードを書く
- 適切にテストする
- なるべくシンプルにする
```

良い例:

```markdown
- Do not introduce a new abstraction unless it is reused in at least two places or isolates an external dependency.
- Run `npm test` after changing business logic.
```

Claudeが、

```text
守れたか / 守れていないか
```

を判断できるレベルまで具体化します。

---

# 18. `Testing`

テスト方針を書きます。

例:

```markdown
# Testing

- Business logic requires unit tests.
- API changes require integration tests.
- User-visible flows require Playwright coverage when practical.
- Do not delete failing tests to make CI pass.
```

---

# 19. テストの「どこまで」を書く

例:

```markdown
# Test Selection

For changes under:

- `src/services/**`: run unit tests and typecheck.
- `src/api/**`: run unit + integration tests.
- `src/components/**`: run relevant component tests.
- authentication flows: run Playwright auth tests.
```

ただし、pathごとの差が大きいなら、

```text
.claude/rules/
```

へ分離した方がよい場合があります。

---

# 20. `Verification`

Claude CodeをAgentとして使うなら非常に重要です。

例:

```markdown
# Verification

After making code changes:

1. Run the narrowest relevant test first.
2. Fix any failure caused by the change.
3. Run typecheck.
4. Run lint when applicable.
5. Run the production build for build-system or dependency changes.

Do not report completion while known relevant tests are failing.
```

これにより、

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

# 21. `Git / Change Policy`

必要な場合だけ書きます。

例:

```markdown
# Change Policy

- Keep changes scoped to the requested task.
- Do not reformat unrelated files.
- Preserve public APIs unless the task explicitly requires a breaking change.
- Do not modify generated files manually.
```

特にAgentは改善範囲を広げがちなので、

```text
不要な変更をしない
```

ルールは有効です。

---

# 22. Git操作について

例:

```markdown
# Git

- Do not push unless explicitly requested.
- Do not force-push.
- Do not rewrite published history.
- Do not commit generated credentials or secrets.
```

ただし、

```text
絶対にpushさせたくない
```

ならCLAUDE.mdだけでなくpermissions等でも制限すべきです。

---

# 23. `Security / Safety`

例:

```markdown
# Security

- Never commit secrets.
- Do not print full API keys or tokens.
- Treat production data as read-only unless explicitly authorized.
- Prefer local or test environments for verification.
```

---

# 24. Safetyルールの限界

CLAUDE.mdは、

```text
behavioral guidance
```

です。

したがって、

```markdown
Never run destructive SQL.
```

と書いていても、

```text
DB操作を技術的にblockする
```

わけではありません。

強制したい場合:

```text
permissions.deny
PreToolUse Hook
sandbox
```

などを使います。

---

# 25. `Skill Routing`

CLAUDE.mdからSkill利用方針を示せます。

例:

```markdown
# Skill Routing

- Use `security-review` for security-sensitive changes.
- Use `ui-ux-review` for user-facing UI changes.
- Use `architecture-review` before large structural refactors.
- Production deployment must use the `production-deploy` skill.
```

役割:

```text
CLAUDE.md
↓
いつどのSkillを使うべきか

SKILL.md
↓
そのタスクをどう実行するか
```

です。

---

# 26. CLAUDE.mdにSkill本文を書かない

悪い例:

```markdown
# Security

Security reviewでは次の40項目を……
1.
2.
3.
...
40.
```

これは、

```text
毎セッション
↓
セキュリティ作業でなくても全項目ロード
```

されます。

より良い構成:

```markdown
# Skill Routing

Security-sensitive changes should use the `security-review` skill.
```

詳細は、

```text
.claude/skills/security-review/SKILL.md
```

へ置きます。

---

# 27. `Do Not`

重要な禁止事項を短く明示する方法です。

例:

```markdown
# Do Not

- Do not modify production configuration without explicit instruction.
- Do not replace established libraries without justification.
- Do not weaken tests to make them pass.
- Do not silently change public API behavior.
```

数を増やしすぎると重要度が薄れるので、本当に重要なものに限定します。

---

# 28. `References`

常時ロードしたい資料がある場合は参照できます。

例:

```markdown
# References

- Architecture overview: @docs/architecture.md
- Git workflow: @docs/git-workflow.md
```

ここで使われている `@...` は、CLAUDE.md独自のimport機能です。

---

# 29. `@import` の仕組み

CLAUDE.mdでは、

```text
@path/to/file
```

という記法で別ファイルをimportできます。

例:

```markdown
See @README.md for the project overview.

# Development

Follow @docs/development.md.

# Git

Follow @docs/git-workflow.md.
```

Claude Codeはこれらを展開し、

```text
CLAUDE.md
+
import先ファイル
```

をcontextへ入れます。

---

# 30. 相対パスの基準

相対パスは、

```text
Claude Codeを起動したdirectory
```

ではなく、

```text
@importを書いたファイル自身の場所
```

を基準に解決されます。

例えば、

```text
project/
├── .claude/
│   └── CLAUDE.md
└── docs/
    └── architecture.md
```

`.claude/CLAUDE.md`からなら、

```markdown
@../docs/architecture.md
```

です。

---

# 31. importは再帰可能

importしたファイルがさらに別のファイルをimportできます。

例:

```text
CLAUDE.md
↓
@docs/development.md
↓
@docs/testing.md
↓
@docs/integration-tests.md
```

公式仕様では、recursive importには深さ制限があります。

最大:

```text
4 hops
```

です。

---

# 32. `@`を文字として書きたい場合

例えば、

```text
@README
```

をimportではなく単なる文字として見せたい場合は、

```markdown
`@README`
```

のようにコード表記にします。

Markdown code spanやfenced code block内ではimport解析されません。

---

# 33. importはコンテキスト節約にはならない

ここは重要です。

```text
CLAUDE.mdを100行
+
別ファイル500行を@import
```

した場合、

```text
500行が必要時だけロードされる
```

わけではありません。

importされたファイルは、

```text
session開始時に一緒にロード
```

されます。

つまりimportの主目的は、

```text
整理
再利用
保守性
```

であり、

```text
token削減
```

ではありません。

---

# 34. 長い情報はSkillまたはRulesへ

もし、

```text
特定タスクでしか不要
```

ならSkillへ。

```text
特定pathでしか不要
```

なら`.claude/rules/`へ。

```text
常に必要だが別ファイルで整理したい
```

なら`@import`です。

判断:

```text
常に必要
↓
CLAUDE.md / @import

特定fileで必要
↓
.claude/rules/

特定taskで必要
↓
Skill
```

---

# 35. `AGENTS.md`との関係

Claude Codeは標準では、

```text
CLAUDE.md
```

を読みます。

既にrepositoryに、

```text
AGENTS.md
```

がある場合、

```markdown
@AGENTS.md

# Claude Code

- Claude固有ルール...
```

のようにCLAUDE.mdからimportできます。

これにより複数coding agent向けのルールを重複管理せずに済みます。

---

# 36. CLAUDE.mdをsymlinkにする方法

Claude固有の追加指示が不要なら、

```bash
ln -s AGENTS.md CLAUDE.md
```

のようにsymlinkする構成も可能です。

ただしWindowsではsymlink作成に権限やDeveloper Modeが必要になる場合があるため、

```text
@AGENTS.md
```

importの方が扱いやすいケースがあります。

---

# 37. `.claude/rules/`との使い分け

大規模projectでは、

```text
CLAUDE.md
```

だけに全部書くのではなく、

```text
.claude/rules/
```

へ分けられます。

例:

```text
project/
├── CLAUDE.md
└── .claude/
    └── rules/
        ├── code-style.md
        ├── testing.md
        ├── security.md
        ├── frontend/
        │   └── react.md
        └── backend/
            └── api.md
```

---

# 38. Rulesの役割

概念:

```text
CLAUDE.md
↓
project全体に常時必要な基本ルール

.claude/rules/
↓
テーマ別・path別に分割したルール
```

Rulesは複数ファイルへ分割できるので、

```text
CLAUDE.md 1枚が巨大化する
```

問題を避けやすくなります。

---

# 39. Path-specific Rules

`.claude/rules/*.md`にはYAML frontmatterで、

```yaml
---
paths:
  - "src/api/**/*.ts"
---
```

のように対象pathを指定できます。

例:

```markdown
---
paths:
  - "src/api/**/*.ts"
---

# API Rules

- Every endpoint must validate input.
- Use the standard error response format.
- Add OpenAPI comments.
```

このルールは、

```text
Claudeが該当fileを扱ったとき
```

に適用されます。

---

# 40. Rulesに向いている内容

例:

```text
React固有ルール
Python固有ルール
APIルール
DB migrationルール
Test fileルール
ROS2 package固有ルール
Frontend / Backend固有ルール
```

つまり、

```text
project全体ではないが、
特定種類のfileでは常に必要
```

なルールです。

---

# 41. Skillとの違い

整理すると、

```text
CLAUDE.md
=
常時必要

Rules
=
特定file / 特定領域で必要

Skill
=
特定taskで必要
```

です。

例:

```text
TypeScript strictを守る
↓
CLAUDE.md

*.tsxでは既存Design Systemを使う
↓
Rules

UI/UXレビューを12ステップで実施
↓
Skill
```

---

# 42. `CLAUDE.local.md`

個人専用かつ、そのproject限定の指示です。

例:

```text
project/
├── CLAUDE.md
└── CLAUDE.local.md
```

`CLAUDE.local.md`には例えば、

```text
自分専用のsandbox URL
個人用test data
ローカル環境固有メモ
個人的なworkflow preference
```

を置けます。

---

# 43. CLAUDE.local.mdはGit管理しない

基本的には、

```text
.gitignore
```

へ追加します。

例:

```gitignore
CLAUDE.local.md
```

Team共通ruleは`CLAUDE.md`へ。

自分だけのruleは`CLAUDE.local.md`へ。

---

# 44. User-level CLAUDE.md

全project共通の個人ルールは、

```text
~/.claude/CLAUDE.md
```

に置きます。

例:

```markdown
# Personal Preferences

- Prefer concise explanations.
- Use `rg` instead of `grep` when available.
- Do not create commits unless explicitly requested.
```

これらはすべてのprojectで読み込まれます。

---

# 45. User-levelにProject固有ルールを書かない

悪い例:

```text
~/.claude/CLAUDE.md

このSaaSのbillingは……
このrobot projectでは……
このiOS appでは……
```

全projectで不要なcontextが入ります。

User-levelには、

```text
本当に全project共通の個人方針
```

だけ置く方がよいです。

---

# 46. Managed / Organization CLAUDE.md

組織全体で共通指示を配布できます。

主な場所:

```text
macOS:
 /Library/Application Support/ClaudeCode/CLAUDE.md

Linux / WSL:
 /etc/claude-code/CLAUDE.md

Windows:
 C:\Program Files\ClaudeCode\CLAUDE.md
```

例えば、

```text
Company coding standards
Security policy
Compliance rules
Data handling policy
```

などです。

---

# 47. 読み込みスコープ

大きく整理すると、

```text
Managed / Organization
        ↓
User
        ↓
Project
        ↓
Local
```

です。

ただしこれは、

```text
後のファイルが前のファイルを機械的にoverride
```

するという意味ではありません。

各内容はcontextへ連結されます。

---

# 48. 同じdirectoryではLocalが後

例えば、

```text
project/CLAUDE.md
project/CLAUDE.local.md
```

なら、

```text
CLAUDE.md
↓
CLAUDE.local.md
```

の順です。

そのため個人的な追加情報が後から入ります。

ただし、矛盾を意図的に作る設計は避けるべきです。

---

# 49. Nested CLAUDE.md

subdirectoryにも置けます。

例:

```text
project/
├── CLAUDE.md
├── frontend/
│   └── CLAUDE.md
└── backend/
    └── CLAUDE.md
```

root:

```text
全体ルール
```

frontend:

```text
Frontend固有ルール
```

backend:

```text
Backend固有ルール
```

という構成です。

---

# 50. 下位CLAUDE.mdはlazy load

`project/`でClaude Codeを起動した場合、

```text
project/CLAUDE.md
```

は起動時に入ります。

一方、

```text
frontend/CLAUDE.md
```

はClaudeがfrontend内のfileを読むときにロードされます。

つまり、

```text
project/CLAUDE.md
↓
startup

frontend/CLAUDE.md
↓
frontendを扱うとき

backend/CLAUDE.md
↓
backendを扱うとき
```

です。

モノレポで非常に有効です。

---

# 51. CLAUDE.mdの推奨サイズ

Anthropic公式では、

```text
1つのCLAUDE.md
↓
200行未満を目安
```

としています。

理由:

```text
毎session contextを消費する
↓
長すぎる
↓
他contextを圧迫
↓
ルール遵守率も下がりやすい
```

からです。

---

# 52. 200行は絶対制限ではない

200行を超えたら動かないわけではありません。

これは、

```text
adhesion / context効率を考えた推奨
```

です。

したがって、

```text
220行だから即分割
```

ではなく、

```text
毎セッション本当に全部必要か？
```

で判断します。

---

# 53. 長くなったときの分割優先順位

CLAUDE.mdが巨大化したら、

```text
① 特定task手順
↓
Skillへ

② 特定pathルール
↓
.claude/rules/へ

③ 常時必要だがテーマ別整理したい
↓
別file + @import

④ Subproject固有
↓
nested CLAUDE.md
```

の順で整理すると分かりやすいです。

---

# 54. HTMLコメントはcontextから除去される

CLAUDE.md中の、

```html
<!-- maintainer note -->
```

のようなblock-level HTML commentは、Claudeのcontextへinjectされる際に除去されます。

つまり、

```markdown
<!-- Human maintainers:
この項目はbilling teamと相談して更新すること
-->
```

のような、

```text
人間だけに見せたいメンテナンス注記
```

を置けます。

---

# 55. HTMLコメントの用途

例:

```markdown
# API Rules

<!--
Maintainer note:
If the API gateway changes, update this section.
-->

- API handlers must validate input.
```

Claudeへは、

```text
- API handlers must validate input.
```

が主に渡り、

maintainer noteはcontext tokenを消費しません。

---

# 56. `/init`

Claude CodeにはCLAUDE.mdの初期生成を支援する、

```text
/init
```

があります。

既存codebaseを分析し、

```text
build command
test command
project conventions
```

などをもとにCLAUDE.md案を作成します。

既存のCLAUDE.mdがある場合は、改善候補を提案する動作があります。

---

# 57. `/init`任せにしすぎない

Claudeがコードから発見できることだけでなく、

```text
コードから推測できないチームルール
```

を人間側で追加することが重要です。

例:

```text
このAPIは外部顧客が利用しているのでbreaking change禁止
このdirectoryはlegacyでread-only
このtest fixtureはproduction互換を前提にしている
```

などです。

---

# 58. `/context`

CLAUDE.mdが本当に読み込まれているか確認するには、

```text
/context
```

を使えます。

Memory filesの一覧で、

```text
CLAUDE.md
CLAUDE.local.md
rules
```

などを確認できます。

---

# 59. `/memory`

Claude Codeのmemory関連を確認・編集するため、

```text
/memory
```

があります。

CLAUDE.mdとAuto Memoryは別物です。

---

# 60. CLAUDE.mdとAuto Memory

整理:

```text
CLAUDE.md
↓
人間が書く

Auto Memory
↓
Claudeが学んだ内容を保存
```

CLAUDE.md:

```text
Coding standards
Architecture
Workflow
Always rules
```

Auto Memory:

```text
過去の修正から学んだ傾向
ユーザーの好み
継続projectの補足
```

です。

---

# 61. Auto Memoryと重複させない

既にCLAUDE.mdに、

```text
テストはnpm test
```

と明示しているなら、同じ内容をAuto Memory側へ重複させる必要はありません。

できるだけ、

```text
Source of Truth
```

を一つにします。

---

# 62. CLAUDE.mdとSettingsの違い

例:

```text
CLAUDE.md
↓
「Bashではproduction DBを触らないで」

settings
↓
特定command自体をdeny
```

整理:

```text
Behavioral guidance
↓
CLAUDE.md

Technical enforcement
↓
settings / permissions
```

です。

---

# 63. CLAUDE.mdとHooksの違い

例:

```text
CLAUDE.md
↓
「変更後はlintを実行する」

Hook
↓
Write/Edit後に自動lintを実行
```

つまり、

```text
Claudeに判断させる
↓
CLAUDE.md

確実に機械実行する
↓
Hook
```

です。

---

# 64. CLAUDE.mdとSkillの違い

```text
CLAUDE.md
=
いつでも持っているべきルール

SKILL.md
=
特定タスクを実行するときだけ読む手順
```

例:

```text
変更後に必ずtest
↓
CLAUDE.md

Security reviewを15ステップで行う
↓
Skill
```

---

# 65. CLAUDE.mdとRulesの違い

```text
CLAUDE.md
=
project全体

Rules
=
特定path / 特定テーマ
```

例:

```text
TypeScript strict
↓
CLAUDE.md

src/api/**/*.tsではOpenAPIコメント必須
↓
rules/api.md
```

---

# 66. CLAUDE.mdを「ルーター」にする設計

大規模projectではCLAUDE.mdを、

```text
全部入り指示書
```

ではなく、

```text
ルーティング + 最重要ルール
```

として使うと強力です。

例:

```markdown
# Core Rules

- Preserve public API compatibility.
- Run relevant tests after changes.
- Never commit secrets.

# Specialized Guidance

- API rules are in `.claude/rules/api.md`.
- Frontend rules are in `.claude/rules/frontend/`.
- Use `security-review` for security-sensitive work.
- Use `release` only when explicitly requested.
```

これによりcontextを抑えられます。

---

# 67. 推奨構成：小規模project

```markdown
# Project

TypeScript CLI application.

# Commands

- Install: `npm ci`
- Test: `npm test`
- Typecheck: `npm run typecheck`
- Build: `npm run build`

# Coding Rules

- Keep TypeScript strict.
- Avoid `any`.
- Reuse existing utilities.

# Verification

After code changes:

1. Run relevant tests.
2. Run typecheck.
3. Run build when dependencies or config change.

# Safety

- Do not commit secrets.
- Do not push unless explicitly requested.
```

小規模ならこれで十分です。

---

# 68. 推奨構成：中規模project

```markdown
# Project Overview

...

# Repository Structure

...

# Architecture

...

# Commands

...

# Coding Rules

...

# Testing

...

# Verification

...

# Change Policy

...

# Security

...

# Skill Routing

...
```

---

# 69. 推奨構成：モノレポ

Root:

```text
repo/
├── CLAUDE.md
├── .claude/
│   └── rules/
│       ├── security.md
│       └── git.md
│
├── apps/
│   ├── web/
│   │   ├── CLAUDE.md
│   │   └── .claude/
│   │       └── rules/
│   │
│   └── api/
│       └── CLAUDE.md
│
└── packages/
```

root CLAUDE.md:

```markdown
# Monorepo

- Use pnpm workspaces.
- Keep package boundaries intact.
- Shared packages live under `packages/`.

# Commands

- Install: `pnpm install`
- Full test: `pnpm test`

# Routing

- Web-specific rules live under `apps/web/`.
- API-specific rules live under `apps/api/`.
```

---

# 70. 良いCLAUDE.md例

```markdown
# Project Overview

This repository is a TypeScript SaaS backend.

# Architecture

- API handlers live in `src/api/`.
- Business logic lives in `src/services/`.
- Database access lives in `src/repositories/`.
- Do not access Prisma directly outside repositories.

# Commands

- Install: `npm ci`
- Unit tests: `npm test`
- Typecheck: `npm run typecheck`
- Lint: `npm run lint`
- Build: `npm run build`

# Change Rules

- Keep changes scoped to the request.
- Preserve public API compatibility unless explicitly instructed otherwise.
- Do not modify generated files manually.
- Reuse existing abstractions before introducing new ones.

# Verification

After changes:

1. Run the narrowest relevant test.
2. Fix failures caused by the change.
3. Run typecheck.
4. Run lint when relevant.
5. Run build for dependency, config, or bundling changes.

Do not report completion while known relevant tests are failing.

# Security

- Never commit secrets.
- Do not expose full tokens in output.
- Treat production data as read-only.

# Skill Routing

- Use `security-review` for auth, permissions, secrets, or external-input changes.
- Use `architecture-review` before cross-module refactors.
- Use `release` only when the user explicitly requests deployment.
```

---

# 71. 悪いCLAUDE.md例

```markdown
# Instructions

コードを綺麗にしてください。
できるだけ良い設計にしてください。
テストも適切にしてください。
必要ならリファクタしてください。
セキュリティにも気をつけてください。
最新技術を使ってください。
できるだけ高速にしてください。
```

問題:

```text
曖昧
検証不能
責務が広い
優先順位が不明
完了条件が不明
```

Claudeごとの解釈差が大きくなります。

---

# 72. 改善版

```markdown
# Coding

- Keep existing architecture unless the task requires a structural change.
- Do not introduce new dependencies without a concrete benefit.
- Do not refactor unrelated code.

# Verification

- Run `npm test` after business-logic changes.
- Run `npm run typecheck` after TypeScript changes.
- Run `npm run build` after dependency or build-config changes.
```

このように、

```text
曖昧な価値観
↓
具体的な行動ルール
```

へ変換します。

---

# 73. 矛盾するルールを作らない

例えば、

root:

```markdown
Never modify generated files.
```

subdirectory:

```markdown
Update generated files manually after schema changes.
```

のような矛盾があると、Claudeはどちらかを任意に選ぶ可能性があります。

CLAUDE.mdは、

```text
priority engineで機械的に解決
```

する仕組みではありません。

定期的に、

```text
root CLAUDE.md
nested CLAUDE.md
rules
user CLAUDE.md
```

を見直します。

---

# 74. 優先度を文章で表現する場合

どうしても条件分岐が必要なら、

```markdown
# Rules

- Default: do not modify generated files.
- Exception: files under `generated/schema/` may be regenerated only with `npm run generate`.
```

のように、

```text
Default
Exception
```

を同じ場所で明示する方が安全です。

---

# 75. 一時的なタスクを書かない

悪い例:

```markdown
今日中にbilling bugを直す。
次はUserControllerを見直す。
今週はAWS migration中。
```

CLAUDE.mdはpersistentなので、

```text
数日後には古い情報
```

になりやすいです。

短期タスクは、

```text
Issue tracker
TODO
会話context
Auto Memory
```

などを使います。

---

# 76. 秘密情報を書かない

CLAUDE.mdは通常repositoryに含まれます。

したがって、

```text
API key
password
production credential
private token
個人情報
```

を書いてはいけません。

ローカル固有情報でもsecretは、

```text
.env
secret manager
OS keychain
```

などで扱います。

---

# 77. 実装詳細を重複させない

例えばコードを見れば明らかな、

```text
UserService has createUser()
```

のような情報をCLAUDE.mdへ書くと、

```text
コード変更
↓
CLAUDE.mdだけ古くなる
```

可能性があります。

CLAUDE.mdには、

```text
コードだけでは分かりにくい「規則・意図」
```

を優先します。

---

# 78. CLAUDE.mdを自己更新させる場合の注意

例えば、

```markdown
失敗したらCLAUDE.mdへ学びを追加する。
```

というルールは慎重に使うべきです。

問題:

```text
特殊ケースで失敗
↓
恒久rule追加
↓
次の別taskでも適用
↓
ルールが肥大化・矛盾
```

します。

推奨:

```text
失敗
↓
原因分析
↓
再発可能性を評価
↓
本当にproject共通ならCLAUDE.md候補
↓
特定taskならSkill
↓
特定pathならRules
```

です。

---

# 79. CLAUDE.mdへ追加する判断基準

公式の考え方に近い判断は、

```text
同じ説明をまたClaudeにする必要があるか？
```

です。

追加候補:

```text
Claudeが同じミスを2回した
Code reviewで「Claudeが知っているべきだった」と判明
毎session同じ説明をしている
新人にも同じ説明が必要
```

ならCLAUDE.md候補です。

---

# 80. 追加しない方がよい判断基準

次なら別の場所を検討します。

```text
1回だけ必要
↓
Prompt

特定taskで必要
↓
Skill

特定directoryだけ
↓
Rules / nested CLAUDE.md

Claudeが自分で覚える補足
↓
Auto Memory

強制制御
↓
Settings / Hooks
```

---

# 81. 推奨テンプレート：汎用

```markdown
# Project Overview

<!-- 2〜5行でprojectの目的と主要stack -->

# Repository Structure

- `...`: ...
- `...`: ...

# Architecture

- ...
- ...
- ...

# Commands

- Install: `...`
- Test: `...`
- Typecheck: `...`
- Lint: `...`
- Build: `...`

# Coding Rules

- ...
- ...
- ...

# Testing

- ...
- ...

# Verification

After code changes:

1. ...
2. ...
3. ...

# Change Policy

- Keep changes scoped to the requested task.
- Do not modify unrelated files.
- Preserve public APIs unless explicitly requested.

# Security

- Never commit secrets.
- ...

# Skill Routing

- Use `...` when ...
- Use `...` when ...

# Do Not

- ...
- ...
```

---

# 82. 推奨テンプレート：AI Agent重視

```markdown
# Core Operating Rules

- Gather sufficient context before editing.
- Prefer the smallest change that solves the task.
- Do not modify unrelated code.
- Verify changes before reporting completion.

# Investigation

Before changing unfamiliar code:

1. Locate the relevant implementation.
2. Read callers and tests.
3. Identify architecture constraints.
4. Confirm the root cause.

# Implementation

- Preserve existing public behavior unless explicitly changing it.
- Reuse established patterns in the repository.
- Do not add dependencies without justification.

# Verification

After implementation:

1. Run the narrowest relevant test.
2. Fix failures caused by the change.
3. Run static checks.
4. Run broader tests only when justified.

Never claim success if relevant verification is failing or was not run.

# Escalation

If the task requires:

- security review → use `security-review`
- architecture review → use `architecture-review`
- production deployment → require explicit user request and use `release`
```

---

# 83. 推奨テンプレート：ロボティクス / ROS2

```markdown
# Project Overview

This repository controls a ROS2-based robotic system.

# Safety

- Default to simulation before hardware execution.
- Do not command real hardware unless explicitly requested.
- Keep velocity, acceleration, and workspace limits enabled.
- Never bypass emergency-stop related logic.

# Architecture

- ROS2 nodes live under `src/`.
- Hardware interfaces must remain isolated from policy code.
- Sensor timestamps and coordinate frames must be preserved.

# Commands

- Build: `colcon build --symlink-install`
- Test: `colcon test`
- Test results: `colcon test-result --verbose`

# Verification

For controller changes:

1. Build the affected package.
2. Run unit/integration tests.
3. Verify topic and frame assumptions.
4. Test in simulation before recommending hardware execution.

# Skill Routing

- Use `ros2-debug` for node/topic/controller diagnosis.
- Use `robot-safety-review` for hardware-control changes.
```

---

# 84. 推奨テンプレート：Web Application

```markdown
# Project Overview

...

# Architecture

- UI components must not access the database.
- API routes must validate external input.
- Business logic belongs in services.
- Database access belongs in repositories.

# Commands

...

# Frontend

- Reuse the existing design system.
- Do not introduce duplicate components.

# Backend

- Preserve API response contracts.
- Use the standard error format.

# Testing

- Business logic: unit tests.
- API changes: integration tests.
- Critical user flows: E2E tests.

# Security

- Treat all external input as untrusted.
- Never log secrets.
- Use the `security-review` skill for auth or permission changes.
```

---

# 85. CLAUDE.mdを作る順番

おすすめ:

```text
① Projectの絶対ルールを洗い出す
↓
② Build / Test commandを書く
↓
③ Architecture boundaryを書く
↓
④ Verificationを書く
↓
⑤ 「してはいけない」を厳選
↓
⑥ Skill routingを書く
↓
⑦ 200行前後を超えるなら分割
↓
⑧ /contextでロード確認
↓
⑨ 実taskで評価
↓
⑩ 繰り返しミスだけ追記
```

---

# 86. CLAUDE.md評価方法

CLAUDE.mdを書いたら、実taskで評価します。

見るもの:

```text
指示遵守率
不要な変更数
テスト実行率
誤ったarchitecture変更数
確認なしの危険操作数
token/context消費
```

---

# 87. CLAUDE.mdあり / なしで比較する

同じtaskを、

```text
A: CLAUDE.mdあり
B: CLAUDE.mdなし
```

で比較すると効果が分かりやすいです。

例えば:

```text
既存APIを壊さずfeature追加
```

を依頼し、

```text
Architecture遵守
Test
変更範囲
Output
```

を比較します。

---

# 88. CLAUDE.mdは「量」ではなく「情報密度」

良いCLAUDE.md:

```text
100行
↓
毎行が実際の判断に効く
```

悪いCLAUDE.md:

```text
500行
↓
一般論
重複
README転載
長い背景説明
```

です。

重要なのは、

```text
tokenあたりの有用な制約・文脈
```

です。

---

# 89. 「全部Claudeに覚えさせる」は逆効果

大量のルールを常時投入すると、

```text
重要rule
+
低重要rule
+
例外
+
古いrule
+
長いreference
```

が同時に入り、

```text
重要指示が埋もれる
```

可能性があります。

そのため、

```text
常時必要なコア
↓
CLAUDE.md

path限定
↓
Rules

task限定
↓
Skills

機械的強制
↓
Hooks / Settings
```

という分離が重要です。

---

# 90. 最も重要な設計原則

`CLAUDE.md` は、

```text
「Claudeにこのrepositoryのすべてを説明するファイル」
```

ではありません。

より正確には、

```text
「Claudeがこのrepositoryで毎回正しく判断するために、
常に持っているべき最小限の運用・設計コンテキスト」
```

です。

理想:

```text
Project identity
↓
何のrepositoryか

Architecture boundaries
↓
何を守るか

Commands
↓
どうbuild/testするか

Verification
↓
完了をどう判定するか

Safety
↓
何に注意するか

Routing
↓
詳細ルール・Skillをどこで使うか
```

です。

---

# 91. CLAUDE.md・Rules・Skills・Hooksの役割分担

```text
                  ┌──────────────────┐
                  │    CLAUDE.md     │
                  │ 常時コンテキスト │
                  └────────┬─────────┘
                           │
          ┌────────────────┼─────────────────┐
          │                │                 │
          ▼                ▼                 ▼
   .claude/rules/       Skills          Settings/Hooks
   path別ルール       task別手順          強制制御
          │                │                 │
          └────────────────┼─────────────────┘
                           ▼
                     Claude Agent
                           │
                           ▼
                  Gather Context
                           │
                           ▼
                       Action
                           │
                           ▼
                      Verify
                           │
                    ┌──────┴──────┐
                    │             │
                  修正            完了
                    │
                    └────→ 再検証
```

---

# 92. 最終チェックリスト

新しい`CLAUDE.md`を作ったら確認します。

```text
[ ] 毎session必要な内容だけになっている
[ ] Project概要は短い
[ ] Build / Test commandが具体的
[ ] Architecture boundaryが明確
[ ] 曖昧な「良いコードを書く」系指示を避けた
[ ] Verification手順がある
[ ] 完了条件が分かる
[ ] 不要な変更を防ぐルールがある
[ ] 秘密情報が入っていない
[ ] 一時的なTODOが入っていない
[ ] 長いtask手順をSkillへ分離した
[ ] path限定ルールをRulesへ分離した
[ ] 強制禁止をCLAUDE.mdだけに頼っていない
[ ] 矛盾するnested CLAUDE.md / Rulesがない
[ ] @import先が本当に常時必要
[ ] 目安として200行前後に収まっている
[ ] /contextでロードを確認した
[ ] 実taskで効果を評価した
```

---

# 93. まとめ

`CLAUDE.md` の構成を最も簡潔にまとめると、

```text
CLAUDE.md
│
├─ Project Overview
│    ↓
│  何のprojectか
│
├─ Repository Structure
│    ↓
│  どこを見るか
│
├─ Architecture
│    ↓
│  何を守るか
│
├─ Commands
│    ↓
│  どうbuild/testするか
│
├─ Coding Rules
│    ↓
│  どう実装するか
│
├─ Verification
│    ↓
│  正しくできたか
│
├─ Safety / Change Policy
│    ↓
│  何を避けるか
│
└─ Skill / Rules Routing
     ↓
   詳細指示をどこへ委譲するか
```

そしてClaude Code全体としては、

```text
CLAUDE.md
=
常時必要なProject指示

CLAUDE.local.md
=
個人かつProject固有の常時指示

~/.claude/CLAUDE.md
=
個人の全Project共通指示

.claude/rules/
=
特定path・テーマのルール

SKILL.md
=
特定taskの実行手順

Auto Memory
=
Claude自身が保持する学習・補足

Settings / Permissions
=
技術的な許可・拒否

Hooks
=
決定論的な自動処理・強制チェック
```

という役割分担にすると、Claude Codeを長期運用しても指示が肥大化しにくく、Agentの判断精度と安全性を両立しやすくなります。

---

# 参考資料

Anthropic公式 Claude Code Docs:

- How Claude remembers your project  
  https://code.claude.com/docs/en/memory

- How Claude Code works  
  https://code.claude.com/docs/en/how-claude-code-works

- Claude Code settings  
  https://code.claude.com/docs/en/settings

特に確認するとよい項目:

- CLAUDE.md files
- When to add to CLAUDE.md
- Choose where to put CLAUDE.md files
- Set up a project CLAUDE.md
- Write effective instructions
- Import additional files
- How CLAUDE.md files load
- Organize rules with `.claude/rules/`
- Path-specific rules
- CLAUDE.md vs auto memory
- Manage CLAUDE.md for large teams
