# Claude CodeのSkill登録方法

Claude CodeのSkill登録は、基本的に**決められた場所にフォルダを作り、その中に `SKILL.md` を置く**だけです。

別途「登録コマンド」を実行する必要はありません。

---

# 1. Skillを置く場所

用途によって主に2種類あります。

| 種類 | 置き場所 | 適用範囲 |
|---|---|---|
| Personal Skill | `~/.claude/skills/<skill-name>/SKILL.md` | 自分の全プロジェクト |
| Project Skill | `<project>/.claude/skills/<skill-name>/SKILL.md` | そのプロジェクトだけ |

---

# 2. 全プロジェクト共通で使う場合

ホームディレクトリ配下の `~/.claude/skills/` に置きます。Windows でも場所は同じで、実体は `C:\Users\<user>\.claude\skills\` です。

```text
~/.claude/
└── skills/
    └── security-review/
        └── SKILL.md
```

例えば、

```text
~/.claude/
└── skills/
    ├── security-review/
    │   └── SKILL.md
    │
    ├── ui-ux-review/
    │   └── SKILL.md
    │
    ├── debug/
    │   └── SKILL.md
    │
    └── architecture-review/
        └── SKILL.md
```

この場所に置いたSkillは、基本的にどのプロジェクトでClaude Codeを起動しても利用できます。

---

# 3. 特定プロジェクトだけで使う場合

プロジェクト直下の `.claude/skills/` に置きます。

```text
my-project/
├── CLAUDE.md
├── src/
└── .claude/
    └── skills/
        └── security-review/
            └── SKILL.md
```

例えば複数Skillを持つ場合は、

```text
my-project/
│
├── CLAUDE.md
│
└── .claude/
    └── skills/
        │
        ├── architecture-review/
        │   └── SKILL.md
        │
        ├── security-review/
        │   └── SKILL.md
        │
        ├── ui-ux-review/
        │   └── SKILL.md
        │
        ├── testing/
        │   └── SKILL.md
        │
        ├── debug/
        │   └── SKILL.md
        │
        └── release/
            └── SKILL.md
```

のようにします。

---

# 4. 最小構成

Skillは最低限これだけで動かせます。

```text
.claude/
└── skills/
    └── security-review/
        └── SKILL.md
```

`SKILL.md` の例:

```markdown
---
description: Webアプリのセキュリティレビューを行う。脆弱性チェック、OWASP、認証や認可の安全性確認を依頼された場合に使用する。
---

# Security Review

以下を確認する。

1. 認証
2. 認可
3. 入力validation
4. SQL Injection
5. XSS
6. CSRF
7. SSRF
8. Secret漏洩
9. Dependency脆弱性
10. Error handling
11. Logging
12. Test

問題を以下で分類する。

- Critical
- High
- Medium
- Low

各問題について、

- 問題箇所
- リスク
- 攻撃シナリオ
- 修正方法

を報告する。
```

---

# 5. Skill名とフォルダ名

基本的に、Skillはフォルダ単位で管理します。

```text
.claude/skills/security-review/SKILL.md
```

であれば、

```text
/security-review
```

として呼び出せます。

同様に、

```text
.claude/skills/ui-ux-review/SKILL.md
```

なら、

```text
/ui-ux-review
```

です。

実運用では、

```text
1 Skill = 1フォルダ
```

にしておくと管理しやすいです。

---

# 6. おすすめのSKILL.md構成

最低限、以下のような構成にしておくと使いやすくなります。

```markdown
---
name: security-review
description: >
  Webアプリのセキュリティレビューを行う。
  脆弱性チェック、OWASP、認証、認可、
  XSS、CSRF、SQL Injection等を確認するときに使用する。

when_to_use: >
  ユーザーがセキュリティレビュー、
  脆弱性チェック、security review、
  OWASP、認証・認可の安全性確認を依頼した場合。
---

# Purpose

コードベースのセキュリティリスクを検出する。

# Procedure

1. プロジェクト構成を確認する。
2. attack surfaceを特定する。
3. authenticationを確認する。
4. authorizationを確認する。
5. input validationを確認する。
6. injection脆弱性を確認する。
7. secret漏洩を確認する。
8. dependencyを確認する。
9. security testを実施する。
10. 結果をseverity別に報告する。

# Output

各問題について以下を出力する。

- Severity
- File
- Problem
- Risk
- Recommended fix
```

---

# 7. `description` は特に重要

ClaudeがSkillを自動選択するとき、Skillの説明情報が重要になります。

悪い例:

```yaml
description: コードを改善するSkill
```

これだと範囲が広すぎます。

良い例:

```yaml
description: >
  Webアプリのセキュリティレビューを行う。
  ユーザーが「脆弱性」「security review」
  「OWASP」「認証の安全性」を依頼した場合に使用する。
```

このように、

- 何をするSkillか
- どんな依頼で使うか

を具体的に書くと、誤発火が減ります。

---

# 8. `when_to_use` を使う場合

例えば、

```yaml
when_to_use: >
  脆弱性チェック、OWASP、認証、
  XSS、CSRF、SQL injectionについて
  ユーザーが確認を求めた場合。
```

のように書きます。

役割としては、

```text
description
    ↓
Skillの目的

when_to_use
    ↓
発火条件
```

と考えると分かりやすいです。

---

# 9. 自動発火させたくないSkill

例えば、

```text
/deploy
/git-push
/delete-db
/release
```

のような副作用のあるSkillは、Claudeに自動選択させない方が安全です。

その場合は、

```markdown
---
name: deploy
description: Production環境へデプロイする
disable-model-invocation: true
---

# Deploy

1. test
2. build
3. deploy
```

のようにします。

挙動は、

```text
Claude自身
    ↓
自動発火しない

ユーザー
    ↓
/deploy
    ↓
実行
```

です。

副作用があるSkillでは非常に重要です。

---

# 10. Claude専用Skill

逆に、

```yaml
user-invocable: false
```

とすると、ユーザーが直接呼ばず、Claudeが必要に応じて利用するSkillにできます。

例:

```markdown
---
name: project-conventions
description: このプロジェクト固有の設計規約
user-invocable: false
---
```

イメージ:

```text
ユーザー
    ↓
直接実行不可

Claude
    ↓
必要時に自動利用
```

---

# 11. 主な設定の違い

| 設定 | ユーザーから直接実行 | Claudeが自動利用 |
|---|---:|---:|
| デフォルト | ○ | ○ |
| `disable-model-invocation: true` | ○ | × |
| `user-invocable: false` | × | ○ |

副作用のある操作は、

```yaml
disable-model-invocation: true
```

を付けるのが安全です。

---

# 12. Skillフォルダには補助ファイルも置ける

Skillは `SKILL.md` だけでなく、関連する資料やスクリプトもまとめて管理できます。

例えば、

```text
.claude/
└── skills/
    └── security-review/
        ├── SKILL.md
        ├── checklist.md
        ├── owasp.md
        └── examples/
            ├── xss.md
            └── sql-injection.md
```

とできます。

役割を分けると、

```text
SKILL.md
    ↓
メイン手順

checklist.md
    ↓
詳細チェックリスト

references/
    ↓
参考資料

scripts/
    ↓
必要に応じて使うスクリプト
```

のようにできます。

巨大な `SKILL.md` 1枚に全部詰め込む必要はありません。

---

# 13. CLAUDE.mdとの役割分担

おすすめは、

```text
CLAUDE.md
    =
常時守るルール

SKILL.md
    =
特定作業の実行手順
```

です。

例えばCLAUDE.mdには、

```markdown
- TypeScript strictを維持する
- production DBを直接変更しない
- 新機能追加後は必ずtestを実行する
- API互換性を壊さない
- secretをrepositoryへ保存しない
```

のような常時適用ルールを置きます。

Skillには、

```text
security-review
testing
ui-ux-review
debug
release
architecture-review
```

など、特定タスク時の具体的な手順を置きます。

---

# 14. SkillへのルーティングをCLAUDE.mdに書く

必要であれば、CLAUDE.md側からSkill利用方針を指定できます。

例えば、

```markdown
# Skill Routing

- Securityに関係する変更では `security-review` Skillを使用する。
- UI変更では `ui-ux-review` Skillを使用する。
- 実装後は `testing` Skillを使用する。
- Production deploymentはユーザーが明示的に `/release` を実行した場合のみ行う。
```

こうすると、

```text
CLAUDE.md
    ↓
どのSkillを使うべきか判断

Skill
    ↓
具体的な手順を実行
```

という構成になります。

---

# 15. Personal SkillとProject Skillの使い分け

## Personal Skill向き

```text
~/.claude/skills/
```

に置くもの:

- security-review
- code-review
- debugging
- research
- documentation
- architecture-review
- git関連の共通操作
- 共通テスト手順

つまり、

```text
どのプロジェクトでも使う
```

ものです。

---

## Project Skill向き

```text
project/.claude/skills/
```

に置くもの:

- そのプロジェクト固有のdeploy
- DB migration
- 特殊なbuild
- 社内API操作
- 固有architecture
- 固有テスト環境
- 特殊なデータ変換

つまり、

```text
そのrepoでしか意味がない
```

ものです。

---

# 16. おすすめの実運用構成

```text
~/.claude/
│
└── skills/
    │
    ├── security-review/
    ├── architecture-review/
    ├── ui-ux-review/
    ├── debug/
    └── code-review/
```

をPersonal Skillとして持ち、

各プロジェクト側では、

```text
project/
│
├── CLAUDE.md
│
└── .claude/
    └── skills/
        │
        ├── project-test/
        ├── project-deploy/
        ├── migration/
        └── release/
```

のようにします。

これにより、

```text
汎用Skill
    ↓
~/.claude/skills/

プロジェクト固有Skill
    ↓
project/.claude/skills/
```

と分離できます。

---

# 17. Skillを作る例

例えばPersonal Skillとして `security-review` を作る場合、

```bash
mkdir -p ~/.claude/skills/security-review
```

その中に、

```text
~/.claude/skills/security-review/SKILL.md
```

を作ります。

Project Skillなら、

```bash
mkdir -p .claude/skills/security-review
```

として、

```text
.claude/skills/security-review/SKILL.md
```

を作ります。

Windows（PowerShell）なら、

```powershell
New-Item -ItemType Directory -Force ~/.claude/skills/security-review
```

のようにします。置き場所自体はどのOSでも同じです。

---

# 18. 作成後の確認

基本的に別途登録処理は不要です。

流れは、

```text
Skillフォルダ作成
        ↓
SKILL.md作成
        ↓
保存
        ↓
Claude Codeが検出
```

です。

必要に応じて、

```text
/skills
```

で一覧を確認したり、

```text
/security-review
```

のように直接呼び出して動作確認します。

---

# 19. Skillの動作確認方法

新しいSkillを作ったら、最低限以下を確認します。

## ① 手動実行

```text
/security-review
```

のように直接呼び出す。

---

## ② 自動発火

例えば、

```text
このWebアプリの脆弱性を確認して
```

と依頼し、Claudeが該当Skillを選択するか確認します。

---

## ③ 誤発火確認

例えば、

```text
このUIの余白を調整して
```

という無関係な依頼でsecurity Skillが発火しないか確認します。

---

## ④ 副作用Skill

deployなどについて、

```text
disable-model-invocation: true
```

が効いていて、Claudeが勝手に実行しないことを確認します。

---

# 20. Skillを増やすときの設計原則

Skillを増やす場合は、

```text
1 Skill = 1明確な責務
```

を推奨します。

良い例:

```text
security-review
ui-ux-review
testing
debug
release
architecture-review
```

悪い例:

```text
super-developer-skill
```

の中に、

```text
security
UI
testing
deploy
AWS
database
debug
documentation
```

を全部入れる構成です。

Skillの責務が重複すると、

- 誤発火
- Skill選択の迷い
- 指示競合
- context肥大化

が起きやすくなります。

---

# 21. 最低限これだけ覚えればよい

全プロジェクト共通なら、

```text
~/.claude/skills/
└── my-skill/
    └── SKILL.md
```

プロジェクト限定なら、

```text
project/
└── .claude/
    └── skills/
        └── my-skill/
            └── SKILL.md
```

中身は最低限、

```markdown
---
description: このSkillが何をするか、いつ使うか
---

# Instructions

Claudeに実行させたい手順を書く。
```

です。

---

# 22. まとめ

Claude CodeのSkill登録は、

```text
① Skill用フォルダを作る
↓
② その中にSKILL.mdを置く
↓
③ description等を書く
↓
④ 実行手順を書く
↓
⑤ Claude Codeが自動認識
```

という非常に単純な仕組みです。

配置先は、

```text
全プロジェクト共通
↓
~/.claude/skills/<skill-name>/SKILL.md
```

または、

```text
特定プロジェクト限定
↓
<project>/.claude/skills/<skill-name>/SKILL.md
```

です。

実運用では、

```text
CLAUDE.md
= 常時適用する方針・制約

SKILL.md
= 特定作業の具体的手順

Personal Skill
= 全プロジェクト共通

Project Skill
= そのrepo固有
```

という役割分担にすると、管理しやすく安定したClaude Code環境を作れます。
