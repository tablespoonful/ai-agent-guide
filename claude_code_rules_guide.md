# Claude Code：.claude/rules/の構成（中身）についての解説

> 対象: Claude Code の `.claude/rules/`  
> 更新基準: 2026-08-29 時点のAnthropic公式ドキュメント  
> 目的: ルールをテーマ別・path別にどう分割し、`paths` をどう書けばよいかを、設計意図も含めて理解する

---

<!-- toc -->
# 目次

- [1. .claude/rules/ とは何か](#1-clauderules-とは何か)
- [2. CLAUDE.mdとの違い](#2-claudemdとの違い)
- [3. Skillとの違い](#3-skillとの違い)
- [4. 置き場所](#4-置き場所)
- [5. ディレクトリ構成](#5-ディレクトリ構成)
- [6. 最小例](#6-最小例)
- [7. paths ― path限定ルール](#7-paths--path限定ルール)
- [8. globパターン](#8-globパターン)
- [9. 複数パターンとbrace展開](#9-複数パターンとbrace展開)
- [10. 展開のbudget](#10-展開のbudget)
- [11. [ の扱い](#11--の扱い)
- [12. paths を書かないルール](#12-paths-を書かないルール)
- [13. Symlinkで複数プロジェクトへ共有](#13-symlinkで複数プロジェクトへ共有)
- [14. User-level rules](#14-user-level-rules)
- [15. setting sourcesとの関係](#15-setting-sourcesとの関係)
- [16. rulesに向いている内容](#16-rulesに向いている内容)
- [17. rulesに向いていない内容](#17-rulesに向いていない内容)
- [18. 例：Reactコンポーネント](#18-例reactコンポーネント)
- [19. 例：APIハンドラ](#19-例apiハンドラ)
- [20. 例：テストファイル](#20-例テストファイル)
- [21. 例：DB migration](#21-例db-migration)
- [22. 例：ROS2 / ロボティクス](#22-例ros2--ロボティクス)
- [23. 矛盾を作らない](#23-矛盾を作らない)
- [24. compaction後の挙動](#24-compaction後の挙動)
- [25. デバッグ方法](#25-デバッグ方法)
- [26. モノレポでの構成](#26-モノレポでの構成)
- [27. 他チームのルールを除外する](#27-他チームのルールを除外する)
- [28. rulesを書くときの原則](#28-rulesを書くときの原則)
- [29. 最終チェックリスト](#29-最終チェックリスト)
- [30. まとめ](#30-まとめ)
- [参考資料](#参考資料)

<!-- /toc -->

---

# 1. `.claude/rules/` とは何か

`.claude/rules/` は、**CLAUDE.mdを分割して整理するための仕組み**です。

```text
.claude/rules/
    ↓
テーマごと・path ごとに分けたMarkdownファイル群
    ↓
条件に応じてcontextへ入る
```

最大の特徴は、

```text
paths frontmatter を書くと
        ↓
該当するファイルをClaudeが扱ったときだけロードされる
```

という点です。CLAUDE.mdが常時ロードなのに対し、rulesは**条件付きロードができる**のが本質的な違いです。

---

# 2. CLAUDE.mdとの違い

```text
CLAUDE.md
    ↓
プロジェクト全体に常時必要な基本ルール
    ↓
毎セッションcontextを消費する

.claude/rules/
    ↓
テーマ別・path別に分割したルール
    ↓
paths を書けば該当時のみ消費する
```

判断基準は単純です。

```text
どのファイルを触っていても必要
    ↓
CLAUDE.md

特定の種類のファイルでだけ必要
    ↓
.claude/rules/（paths付き）
```

例:

```text
「TypeScript strictを保つ」
    ↓
CLAUDE.md

「src/api/**/*.ts ではOpenAPIコメント必須」
    ↓
.claude/rules/api.md
```

---

# 3. Skillとの違い

3つ目の軸としてSkillがあります。

```text
CLAUDE.md
    ↓
常時必要

Rules
    ↓
特定のファイルを扱うとき必要

Skill
    ↓
特定のタスクを実行するとき必要
```

公式ドキュメントも、

```text
常にcontextに要らない task-specific な指示
    ↓
Skill を使う
```

としています。

具体例で並べると、

```text
「public APIの互換性を壊さない」
    ↓
CLAUDE.md

「*.tsx では既存Design Systemのコンポーネントを使う」
    ↓
Rules

「UI/UXレビューを12ステップで実施する」
    ↓
Skill
```

ここを混ぜると、CLAUDE.mdが肥大化するか、Skillが常時ロードされているような設計になります。

---

# 4. 置き場所

2種類あります。

| 種類 | 置き場所 | スコープ |
|---|---|---|
| Project rules | `<project>/.claude/rules/` | そのプロジェクト |
| User rules | `~/.claude/rules/` | 自分の全プロジェクト |

ロード順は、

```text
User rules（~/.claude/rules/）
        ↓
Project rules（.claude/rules/）
```

で、**後から読まれるProject rulesのほうが優先度が高く**なります。

---

# 5. ディレクトリ構成

`.md` ファイルは**再帰的に探索されます**。サブディレクトリで整理できます。

```text
your-project/
├── .claude/
│   ├── CLAUDE.md           # メインのプロジェクト指示
│   └── rules/
│       ├── code-style.md   # コードスタイル
│       ├── testing.md      # テスト規約
│       ├── security.md     # セキュリティ要件
│       ├── frontend/
│       │   └── react.md
│       └── backend/
│           └── api.md
```

1ファイル1トピックにし、ファイル名でわかるようにします。

```text
testing.md
api-design.md
error-handling.md
```

のような命名が扱いやすいです。

---

# 6. 最小例

frontmatter無しでも動きます。

`.claude/rules/code-style.md`:

```markdown
# Code Style

- Use 2-space indentation.
- Prefer explicit return types for exported functions.
- Avoid `any`; use `unknown` and narrow types instead.
```

この形（`paths` なし）は、

```text
起動時に無条件でロードされる
    ↓
.claude/CLAUDE.md と同じ優先度
```

になります。つまり**CLAUDE.mdを単に分割しただけ**の状態です。

context削減効果を得たいなら、次の `paths` を使います。

---

# 7. `paths` ― path限定ルール

YAML frontmatterで対象を指定します。

```markdown
---
paths:
  - "src/api/**/*.ts"
---

# API Development Rules

- All API endpoints must include input validation
- Use the standard error response format
- Include OpenAPI documentation comments
```

挙動:

```text
Claudeが src/api/ 配下の .ts を読む
            ↓
このルールがcontextへ入る
            ↓
以降のセッションで有効
```

重要な点として、

```text
発火するのは
「Claudeがマッチするファイルを読んだとき」
であって
「Toolを使うたび」ではない
```

です。

なお、シンボリックリンク経由でプロジェクトディレクトリに到達した場合もマッチします。

---

# 8. globパターン

| パターン | マッチする対象 |
|---|---|
| `**/*.ts` | 任意のディレクトリの全TypeScriptファイル |
| `src/**/*` | `src/` 配下の全ファイル |
| `*.md` | **プロジェクトルート直下**のMarkdown |
| `src/components/*.tsx` | 特定ディレクトリ直下のReactコンポーネント |

最も間違えやすいのが、

```text
*.tsx
    ↓
プロジェクトルート直下だけ

**/*.tsx
    ↓
すべての階層
```

です。サブディレクトリも含めたいなら必ず `**/` を付けます。

---

# 9. 複数パターンとbrace展開

複数書けます。

```markdown
---
paths:
  - "src/**/*.{ts,tsx}"
  - "lib/**/*.ts"
  - "tests/**/*.test.ts"
---
```

`{a,b}` のbrace展開が使えますが、**組み合わせ数が乗算で増えます**。

```text
src/*.{ts,tsx}
    ↓
2パターン

{a,b}/{c,d}/*.{ts,tsx}
    ↓
8パターン
```

---

# 10. 展開のbudget

無限に膨らまないよう上限があります。

```text
1つのルールの paths リスト全体で
    ↓
展開後 1,000パターン
    ↓
かつ 4 MiB
```

braceを含まないパターンはこのbudgetに数えられません。

budgetを超えるパターンは**展開されずそのまま使われ**、リテラルの `{` `}` は何にもマッチしなくなります。

つまり、

```text
braceを盛りすぎる
    ↓
静かにマッチしなくなる
```

ので、パターンは素直に列挙したほうが安全です。

---

# 11. `[` の扱い

glob構文では `[` はブラケット式（`[abc]`）の開始とみなされます。

```text
photos [2024/**
    ↓
ブラケット式として読めない
    ↓
無効なパターン
    ↓
何にもマッチしない
```

ファイル名のリテラルな `[` にマッチさせたいときはエスケープします。

```text
photos \[2024/**
```

無効なパターンがあっても、そのルールの**他のパターンは動き続けます**。

---

# 12. `paths` を書かないルール

```text
paths あり
    ↓
マッチするファイルを読んだときにロード

paths なし
    ↓
起動時に無条件でロード
    ↓
.claude/CLAUDE.md と同じ優先度
```

したがって、

```text
CLAUDE.mdを分割して読みやすくしたい
    ↓
paths なしのrules

contextを節約したい
    ↓
paths ありのrules
```

という使い分けになります。「rulesに移したのにcontextが減らない」場合は、`paths` を書き忘れていることがほとんどです。

---

# 13. Symlinkで複数プロジェクトへ共有

`.claude/rules/` はsymlinkに対応しています。

```bash
ln -s ~/shared-claude-rules .claude/rules/shared
ln -s ~/company-standards/security.md .claude/rules/security.md
```

ディレクトリ単位でもファイル単位でも張れます。循環symlinkは検出されて安全に処理されます。

```text
共通ルールを1箇所で管理
        ↓
各プロジェクトからsymlink
        ↓
更新が一斉に反映される
```

Pluginで配布するほどではないが複数リポジトリで共有したい、という場合に有効です。

なお、Windowsではsymlink作成に管理者権限またはDeveloper Modeが必要になります。

---

# 14. User-level rules

個人的な好みは `~/.claude/rules/` に置きます。

```text
~/.claude/rules/
├── preferences.md    # 個人的なコーディング嗜好
└── workflows.md      # 好みのワークフロー
```

例:

```markdown
# Preferences

- 説明は簡潔にする。
- 可能なら `grep` ではなく `rg` を使う。
- 明示的に依頼されない限りcommitしない。
```

`~/.claude/CLAUDE.md` との使い分けは、

```text
~/.claude/CLAUDE.md
    ↓
全プロジェクトで常時必要な個人方針

~/.claude/rules/
    ↓
テーマ別に整理したい、またはpath限定にしたい個人方針
```

です。

---

# 15. setting sourcesとの関係

Project rulesは、`project` setting sourceを除外すると読み込まれません。

```bash
claude --setting-sources user
```

このとき `.claude/rules/` は読まれません。CIで「ホスト側の設定に影響されたくない」場合の挙動として押さえておきます。

`--bare` を使う場合も、CLAUDE.mdやrulesは読み込まれません。

---

# 16. rulesに向いている内容

```text
言語・フレームワーク固有の規約
    ↓
React、Python、Go の書き方

領域固有の規約
    ↓
APIルール、DB migrationルール

ファイル種別固有の規約
    ↓
テストファイル、設定ファイル、スキーマ

サブシステム固有の規約
    ↓
ROS2 package、決済モジュール
```

共通するのは、

```text
プロジェクト全体ではないが
特定の種類のファイルでは常に必要
```

という性質です。

---

# 17. rulesに向いていない内容

```text
多段の作業手順
    ↓
Skill へ

一度きりの指示
    ↓
プロンプトへ

強制したい禁止事項
    ↓
permissions / hooks へ

プロジェクト全体の基本方針
    ↓
CLAUDE.md へ

秘密情報
    ↓
どこにも書かない
```

特に「手順」をrulesに書くと、該当ファイルを開くたびに長い手順がロードされ、CLAUDE.md肥大化と同じ問題が起きます。

---

# 18. 例：Reactコンポーネント

`.claude/rules/frontend/react.md`:

```markdown
---
paths:
  - "src/components/**/*.tsx"
  - "src/app/**/*.tsx"
---

# React Component Rules

- 既存のDesign Systemコンポーネントを再利用する。新規に同等のものを作らない。
- Server Component をデフォルトとし、`"use client"` は必要な場合のみ付ける。
- データ取得はコンポーネント内で行わず、`src/services/` 経由にする。
- スタイルは Tailwind のユーティリティを使い、独自CSSファイルを追加しない。
- `useEffect` で状態を同期しない。導出値は描画時に計算する。
```

---

# 19. 例：APIハンドラ

`.claude/rules/backend/api.md`:

```markdown
---
paths:
  - "src/api/**/*.ts"
---

# API Rules

- すべてのエンドポイントで入力を validation する。
- エラーレスポンスは共通フォーマットを使う。
- ハンドラにビジネスロジックを書かない。`src/services/` へ委譲する。
- 生SQLをハンドラに書かない。`src/repositories/` 経由にする。
- OpenAPIコメントを付ける。
- 既存のレスポンス契約を変更しない。変更が必要なら新しいバージョンを追加する。
```

---

# 20. 例：テストファイル

`.claude/rules/testing.md`:

```markdown
---
paths:
  - "**/*.test.ts"
  - "**/*.test.tsx"
  - "**/*.spec.ts"
  - "tests/**/*"
  - "e2e/**/*"
---

# Test Rules

- テストを通すためにアサーションを弱めない。
- 失敗しているテストを削除・skipしない。
- 1テスト1振る舞い。
- fixtureは `tests/fixtures/` に置き、テスト間で共有する。
- 外部サービスは必ずmockする。実ネットワークへ出ない。
- E2Eはユーザー視点のフローで書き、内部実装に依存しない。
```

---

# 21. 例：DB migration

`.claude/rules/migration.md`:

```markdown
---
paths:
  - "prisma/migrations/**/*"
  - "db/migrate/**/*"
---

# Migration Rules

- 適用済みのmigrationを編集しない。新しいmigrationを追加する。
- 破壊的変更は2段階に分ける（追加 → 移行 → 削除）。
- NOT NULL列の追加はデフォルト値かバックフィルとセットにする。
- 大きなテーブルへのindex追加は CONCURRENTLY を使う。
- migrationにはロールバック手順をコメントで書く。
```

---

# 22. 例：ROS2 / ロボティクス

`.claude/rules/ros2.md`:

```markdown
---
paths:
  - "src/**/*.launch.py"
  - "src/**/config/*.yaml"
  - "src/**/*_node.cpp"
  - "src/**/*_node.py"
---

# ROS2 Node Rules

- topic名とframe_idを勝手に変更しない。
- QoS設定は既存のプロファイルに合わせる。
- センサーのtimestampを再生成せず、受信値をそのまま伝播する。
- 速度・加速度・作業領域のリミットを無効化しない。
- ハードウェアインタフェースとポリシーコードを混在させない。
- パラメータはlaunchファイルではなくYAMLで宣言する。
```

---

# 23. 矛盾を作らない

rulesは複数ファイルに分かれるため、矛盾が生まれやすくなります。

```text
CLAUDE.md
    ↓
「生成ファイルは変更しない」

rules/codegen.md
    ↓
「schema変更後は生成ファイルを手動更新する」
```

このような矛盾があると、Claudeはどちらかを任意に選ぶ可能性があります。

```text
CLAUDE.md / rules は
priority engine で機械的に解決される仕組みではない
```

条件分岐が必要なら、**同じ場所に Default と Exception を書く**のが安全です。

```markdown
# Generated Files

- Default: 生成ファイルを手で編集しない。
- Exception: `generated/schema/` は `npm run generate` による再生成のみ許可する。
```

定期的に、

```text
root CLAUDE.md
nested CLAUDE.md
.claude/rules/
~/.claude/CLAUDE.md
~/.claude/rules/
```

を見直します。

---

# 24. compaction後の挙動

```text
プロジェクトルートのCLAUDE.md
    ↓
compaction後にディスクから読み直され再注入される

paths付きのrules
    ↓
該当ファイルを再び読んだときに再ロードされる
```

したがって、

```text
compaction後にルールが効かなくなった
```

と感じたら、

```text
① 会話の中だけで与えた指示だった
② まだ該当ファイルを読んでいない path-scoped rule
③ まだロードされていない nested CLAUDE.md
```

のいずれかを疑います。

「必ず効いていてほしいルール」は、rulesではなくCLAUDE.mdに置くか、Hookで強制します。

---

# 25. デバッグ方法

## `/context`

```text
/context
```

の Memory files 欄で、実際にロードされたファイルを確認できます。

```text
ここに出ていない
    ↓
Claudeは見ていない
```

## `InstructionsLoaded` Hook

より詳細に追うなら、Hookでログを取ります。

```json
{
  "hooks": {
    "InstructionsLoaded": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "jq -c . >> /tmp/claude-instructions.log"
          }
        ]
      }
    ]
  }
}
```

matcherでロード理由を絞れます。

```text
session_start
nested_traversal
path_glob_match
include
compact
```

`path_glob_match` を見れば、path-scoped ruleがいつ発火したかがわかります。

---

# 26. モノレポでの構成

大きなリポジトリでは、rulesとnested CLAUDE.mdを併用します。

```text
repo/
├── CLAUDE.md                    # 全体の基本ルール
├── .claude/
│   └── rules/
│       ├── security.md          # paths なし（常時）
│       ├── testing.md           # paths: **/*.test.ts
│       └── git.md               # paths なし（常時）
│
├── apps/
│   ├── web/
│   │   ├── CLAUDE.md            # web固有（lazy load）
│   │   └── .claude/
│   │       └── rules/
│   │           └── react.md
│   │
│   └── api/
│       ├── CLAUDE.md
│       └── .claude/
│           └── rules/
│               └── endpoints.md
│
└── packages/
```

使い分け:

```text
リポジトリ全体で常に必要
    ↓
root CLAUDE.md

リポジトリ全体だがテーマ別に整理したい
    ↓
root .claude/rules/（paths なし）

ファイル種別で決まる
    ↓
.claude/rules/（paths あり）

サブプロジェクト全体
    ↓
nested CLAUDE.md
```

---

# 27. 他チームのルールを除外する

モノレポでは、関係ない祖先のCLAUDE.mdやrulesが読み込まれることがあります。

```json
{
  "claudeMdExcludes": [
    "**/monorepo/CLAUDE.md",
    "/home/user/monorepo/other-team/.claude/rules/**"
  ]
}
```

`.claude/settings.local.json` に置けば、自分のマシンだけの設定になります。

パターンは絶対パスに対してglobで照合されます。managed policyのCLAUDE.mdは除外できません。

---

# 28. rulesを書くときの原則

```text
① 1ファイル1トピック
     ファイル名で内容がわかる

② paths を必ず検討する
     paths なしは CLAUDE.md と同じコスト

③ ** を忘れない
     *.tsx はルート直下だけ

④ 検証可能な粒度で書く
     「綺麗に書く」ではなく「2スペースで書く」

⑤ 手順を書かない
     手順は Skill へ

⑥ 矛盾を作らない
     Default と Exception は同じ場所に

⑦ 短く保つ
     該当ファイルを開くたびにロードされる
```

---

# 29. 最終チェックリスト

```text
[ ] 1ファイル1トピックになっている
[ ] ファイル名で内容が推測できる
[ ] paths を書くべきものに書いてある
[ ] paths なしのルールが本当に常時必要
[ ] サブディレクトリを含めたいpathに ** がある
[ ] braceを使いすぎていない
[ ] ファイル名の [ をエスケープした（該当する場合）
[ ] 曖昧な表現を避けた
[ ] 手順をrulesに書いていない
[ ] CLAUDE.mdやnested CLAUDE.mdと矛盾していない
[ ] 秘密情報が入っていない
[ ] /context または InstructionsLoaded でロードを確認した
[ ] 強制したいものをrulesだけに頼っていない
```

---

# 30. まとめ

`.claude/rules/` の構成を最も簡潔にまとめると、

```text
.claude/rules/
│
├── frontmatter
│   │
│   └─ paths
│        ↓
│      いつロードするか
│      （書かなければ常時）
│
└── Markdown Body
     ↓
   そのファイル群を扱うときに守るルール
```

そしてClaude Code全体での位置づけは、

```text
CLAUDE.md
=
どのファイルを触っていても必要

.claude/rules/（paths なし）
=
同上。テーマ別に整理したもの

.claude/rules/（paths あり）
=
特定のファイルを扱うときだけ必要

nested CLAUDE.md
=
特定のディレクトリで作業するときだけ必要

SKILL.md
=
特定のタスクを実行するときだけ必要

Hooks / permissions
=
Claudeの判断に依存させたくないもの
```

です。

`.claude/rules/` の価値は、

```text
CLAUDE.md を200行以内に保ちながら
プロジェクト固有の細かい規約を失わない
```

点にあります。CLAUDE.mdが長くなってきたら、まず「特定のファイル種別でしか要らない行」を探して `paths` 付きのrulesへ移すのが、最も効果が出やすい整理です。

---

# 参考資料

Anthropic公式 Claude Code Docs:

- How Claude remembers your project  
  https://code.claude.com/docs/en/memory

- Extend Claude with skills  
  https://code.claude.com/docs/en/skills

- Monorepos and large repos  
  https://code.claude.com/docs/en/large-codebases

特に確認するとよい項目:

- Organize rules with `.claude/rules/`
- Set up rules
- Path-specific rules
- Share rules across projects with symlinks
- User-level rules
- Exclude specific CLAUDE.md files
- How CLAUDE.md files load
