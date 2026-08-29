# Claude Code：settings.jsonとpermissionsの構成（中身）についての解説

> 対象: Claude Code の `settings.json` と `permissions`  
> 更新基準: 2026-08-28 時点のAnthropic公式ドキュメント  
> 目的: 設定をどのスコープに置き、権限ルールをどう書けばよいかを、設計意図も含めて理解する

---

<!-- toc -->
# 目次

- [1. settings.jsonとは何か](#1-settingsjsonとは何か)
- [2. 設定ファイルの4スコープ](#2-設定ファイルの4スコープ)
- [3. 優先順位](#3-優先順位)
- [4. どのスコープに何を置くか](#4-どのスコープに何を置くか)
- [5. ファイル形式](#5-ファイル形式)
- [6. 設定の確認方法](#6-設定の確認方法)
- [7. 一時的に設定を変える](#7-一時的に設定を変える)
- [8. 変更が反映されるタイミング](#8-変更が反映されるタイミング)
- [9. permissions ブロックの構造](#9-permissions-ブロックの構造)
- [10. 評価順序](#10-評価順序)
- [11. Tool名だけのルールと、スコープ付きルール](#11-tool名だけのルールとスコープ付きルール)
- [12. Specifierによる絞り込み](#12-specifierによる絞り込み)
- [13. パラメータでマッチする](#13-パラメータでマッチする)
- [14. ワイルドカードの挙動](#14-ワイルドカードの挙動)
- [15. * はサブコマンドの後に置く](#15--はサブコマンドの後に置く)
- [16. Bashルールの詳細な挙動](#16-bashルールの詳細な挙動)
- [17. 環境ランナーとexecラッパーの注意](#17-環境ランナーとexecラッパーの注意)
- [18. リダイレクトの扱い](#18-リダイレクトの扱い)
- [19. PowerShellルール](#19-powershellルール)
- [20. Read と Edit のルール](#20-read-と-edit-のルール)
- [21. パスパターンの4つのアンカー](#21-パスパターンの4つのアンカー)
- [22. Read / Edit denyの限界](#22-read--edit-denyの限界)
- [23. WebFetch のルール](#23-webfetch-のルール)
- [24. MCPのルール](#24-mcpのルール)
- [25. Agent のルール](#25-agent-のルール)
- [26. Cd のルール](#26-cd-のルール)
- [27. Permission modes](#27-permission-modes)
- [28. 作業ディレクトリ](#28-作業ディレクトリ)
- [29. 追加ディレクトリは「設定」を読み込まない](#29-追加ディレクトリは設定を読み込まない)
- [30. sandboxとの関係](#30-sandboxとの関係)
- [31. Hooksとの関係](#31-hooksとの関係)
- [32. Workspace trustと非対話モード](#32-workspace-trustと非対話モード)
- [33. 設定例：個人](#33-設定例個人)
- [34. 設定例：チーム](#34-設定例チーム)
- [35. 設定例：組織](#35-設定例組織)
- [36. よくある落とし穴](#36-よくある落とし穴)
- [37. 権限設計の順序](#37-権限設計の順序)
- [38. permissionsとCLAUDE.mdの併記](#38-permissionsとclaudemdの併記)
- [39. 最終チェックリスト](#39-最終チェックリスト)
- [40. まとめ](#40-まとめ)
- [参考資料](#参考資料)

<!-- /toc -->

---

# 1. settings.jsonとは何か

`settings.json` は、Claude Codeの**クライアント側の挙動を決める設定ファイル**です。

`CLAUDE.md` との違いが最も重要です。

```text
CLAUDE.md
    ↓
Claudeのcontextへ入る
    ↓
Claudeが読んで判断する
    ↓
守られないこともある

settings.json
    ↓
Claude Code本体が読む
    ↓
Claudeの判断とは無関係に適用される
```

したがって、

```markdown
production DBを触らないでください
```

をCLAUDE.mdへ書くのは「お願い」ですが、

```json
{ "permissions": { "deny": ["Bash(psql *)"] } }
```

は「遮断」です。

---

# 2. 設定ファイルの4スコープ

設定は4つのファイルから読まれます。

| スコープ | ファイル | 影響範囲 | 主な用途 |
|---|---|---|---|
| User | `~/.claude/settings.json` | このマシンの自分の全プロジェクト | テーマ、既定モデル、個人のpermission |
| Shared project | `.claude/settings.json` | そのフォルダで作業する全員（コミットして共有） | チームのpermission、hooks、plugin、env |
| Project local | `.claude/settings.local.json` | そのプロジェクトの自分だけ | 個人的な上書き、共有前の試験 |
| Managed | `managed-settings.json` ほか | 組織が配布した全員 | セキュリティポリシー、コンプライアンス |

`~/.claude` はホームディレクトリの `.claude`、単なる `.claude` はプロジェクト内の `.claude` を指します。

---

# 3. 優先順位

同じキーが複数の場所にある場合、次の順で上位が勝ちます。

```text
① Managed settings（最優先）
        ↓
② Command line（claude --settings）
        ↓
③ Project local（.claude/settings.local.json）
        ↓
④ Shared project（.claude/settings.json）
        ↓
⑤ User（~/.claude/settings.json）
```

重要な点として、

```text
Managed settings
    ↓
自分の設定では上書きできない
```

です（ごく一部のセキュリティ例外を除く）。

`CLAUDE.md` が「連結される」のに対し、`settings.json` は「上位が下位をoverrideする」という違いがあります。

---

# 4. どのスコープに何を置くか

判断基準はシンプルです。

```text
全員に必要
    ↓
.claude/settings.json（コミット）

自分だけ・このプロジェクトだけ
    ↓
.claude/settings.local.json

自分だけ・全プロジェクト
    ↓
~/.claude/settings.json

組織として強制
    ↓
managed settings
```

例:

```text
Bash(npm run test *) を許可
    ↓
チーム共通なので .claude/settings.json

自分の好きなテーマ
    ↓
~/.claude/settings.json

自分だけ有効にしたいdebug用hook
    ↓
.claude/settings.local.json
```

`.claude/settings.local.json` はClaude Codeが作成するときはgitから除外されますが、**手動で作った場合は自分で `.gitignore` に追加**します。

---

# 5. ファイル形式

`settings.json` は**strict JSON**です。

```text
// コメント     → 構文エラー
末尾のカンマ    → 構文エラー
```

壊れていると次回起動時に Settings Error として報告されます。

エディタ補完を効かせるには `$schema` を書きます。

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "permissions": {
    "allow": [
      "Bash(npm run lint)",
      "Bash(npm run test *)"
    ],
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)"
    ]
  }
}
```

schemaはCLIの最新リリースに遅れることがあるため、新しいキーに警告が出ても設定が無効とは限りません。

---

# 6. 設定の確認方法

```text
/status
    ↓
どのsettingsソースが読まれたか

/config
    ↓
個人向けオプションのメニュー

/context
    ↓
CLAUDE.md等のメモリファイルのロード状況
```

`/config` は全キーを扱うメニューではなく、テーマやエディタモードなど短いリストです。

単発で変えるなら `key=value` を渡せます。

```text
/config verbose=true
```

---

# 7. 一時的に設定を変える

保存せずにその場だけ変える方法が3つあります。

```text
--settings
    ↓
JSONを直接またはファイルで渡す
user / project / local より上、managed より下に適用

専用フラグ
    ↓
--model、--effort など

環境変数
    ↓
ANTHROPIC_MODEL など
```

例:

```bash
claude --settings '{"model": "claude-opus-4-8"}'
```

CI等で「ホスト側の設定に影響されたくない」なら、後述の `--bare` と組み合わせます。

---

# 8. 変更が反映されるタイミング

Claude Codeは設定ファイルを監視していて、多くの編集は**再起動なしで反映**されます。

```text
即時反映
    ↓
permissions
hooks
apiKeyHelper 等

セッション開始時のみ読む
    ↓
model
effortLevel
outputStyle
```

`model` は `/model`、`effortLevel` は `/effort` でセッション中に変えられます。`outputStyle` は `/clear` か再起動が必要です。

設定ファイルの変更を検知すると `ConfigChange` Hookが走ります。

---

# 9. `permissions` ブロックの構造

権限設定は `permissions` キーの下に書きます。

```json
{
  "permissions": {
    "allow": [],
    "ask": [],
    "deny": [],
    "additionalDirectories": [],
    "defaultMode": "default"
  }
}
```

役割:

```text
allow  … 確認なしで実行を許可
ask    … 必ず確認する
deny   … 常に拒否
```

---

# 10. 評価順序

ここが最重要です。

```text
deny
  ↓
ask
  ↓
allow
```

**最初にマッチしたものが勝ちます。ルールの具体性は順序を変えません。**

したがって、

```json
{
  "permissions": {
    "deny": ["Bash(aws *)"],
    "allow": ["Bash(aws s3 ls)"]
  }
}
```

は `aws s3 ls` も**拒否されます**。denyが先に評価されるためです。

```text
denyに例外は書けない
```

と覚えます。同様に、ask にマッチすれば、より具体的な allow があっても確認が入ります。

---

# 11. Tool名だけのルールと、スコープ付きルール

同じdenyでも挙動が違います。

```text
"Bash"（Tool名だけ）
    ↓
Toolごとcontextから取り除かれる
    ↓
Claudeはそもそも存在を知らない

"Bash(rm *)"（スコープ付き）
    ↓
Toolは使える
    ↓
マッチする呼び出しだけブロック
```

`Bash(*)` は `Bash` と等価で、すべてのBashコマンドにマッチします。denyとして書いた場合はどちらもToolを取り除きます。

例外として `EndConversation` は、他のToolが残っている限りdenyで取り除けません。

---

# 12. Specifierによる絞り込み

Toolごとに固有の指定子があります。

| ルール | 意味 |
|---|---|
| `Bash(npm run build)` | コマンド `npm run build` に完全一致 |
| `Read(./.env)` | カレントディレクトリの `.env` の読み取り |
| `WebFetch(domain:example.com)` | example.com へのfetch |

---

# 13. パラメータでマッチする

スカラーのパラメータ値でも絞れます。

```text
Bash(run_in_background:true)
    ↓
バックグラウンド実行のBash呼び出し
```

ただし、**各Toolの主要な内容フィールドはこの方法で書けません**。

```text
Bash / PowerShell の command
Read / Edit / Write の file_path
Grep / Glob の path
NotebookEdit の notebook_path
WebFetch の url
```

例えば `Bash(command:rm *)` は複合コマンドで迂回できてしまうため無視され、起動時に警告が出ます。

```text
× Bash(command:rm *)
○ Bash(rm *)

× Read(file_path:./secret)
○ Read(./secret)
```

---

# 14. ワイルドカードの挙動

`*` の扱いには独特の規則があります。

| ルール | マッチする | マッチしない |
|---|---|---|
| `Bash(npm run build)` | `npm run build` | `npm run build --watch` |
| `Bash(npm run *)` | `npm run build`、`npm run test --watch`、`npm run` | `npm install` |
| `Bash(git log * main)` | `git log --oneline main`、`git log -5 main` | `git log main` |
| `Bash(git * main)` | `git merge main`、`git push origin main` | `git log` |
| `Bash(* --version)` | `node --version` | `node -v` |
| `Bash(ls *)` | `ls -la`、`ls` | `lsof` |
| `Bash(ls*)` | `ls -la`、`lsof` | |

押さえるべき3点:

```text
① * はその位置の任意のテキストに置き換わる
     Bash(git * main) は -c も含むので危険

② 末尾の「空白 + *」は素のコマンドにもマッチ
     Bash(ls *) は ls にもマッチする
     ただし末尾の * が唯一のワイルドカードのときだけ

③ 末尾 * の前の空白はルールの一部
     Bash(ls *) は lsof にマッチしない
     Bash(ls*) は lsof にマッチする
```

`:*` は末尾ワイルドカードの別表記で、`Bash(ls:*)` は `Bash(ls *)` と同じです。ただし末尾でのみ有効で、`Bash(git:* push)` のコロンはリテラル扱いになります。

---

# 15. `*` はサブコマンドの後に置く

```text
git log --oneline main
 │   │
 │   └─ subcommand（何をするかを決める語）
 └───── program
```

Claude Codeは**最初の `*` より前を書かれた通りに照合**します。したがって、

```text
Bash(git log *)   → git log 系だけ許可
Bash(git *)       → 全gitコマンドを許可
Bash(git * main)  → サブコマンド不問（-c 経由で任意実行が可能）
```

サブコマンドより前に `*` があるallowルールは起動時に警告されます。

---

# 16. Bashルールの詳細な挙動

シェル演算子を理解して評価されます。

```text
認識される区切り
    ↓
&&  ||  ;  |  |&  &  改行
```

`Bash(safe-cmd *)` があっても `safe-cmd && other-cmd` は許可されません。**すべてのサブコマンドにマッチする必要があります**。

また、一部のラッパーは剥がされてから照合されます。

```text
剥がされる
    ↓
timeout / time / nice / nohup / stdbuf
シェル組み込みの command / builtin
zshの noglob
フラグなしの xargs

剥がされない
    ↓
command -v
zshの nocorrect
```

つまり `Bash(npm test *)` は `timeout 30 npm test` にもマッチします。

環境変数の代入についても差があります。

```text
allowルール
    ↓
既知の安全な変数の代入だけ剥がす
    ↓
Bash(npm test *) は NODE_ENV=test npm test にマッチ
それ以外の変数代入があるとマッチしない

deny / askルール
    ↓
任意の代入を越えてマッチする
    ↓
Bash(rm *) は FOO=bar rm -rf tmp/ にマッチ
```

---

# 17. 環境ランナーとexecラッパーの注意

次のツールは**ラッパーとして剥がされません**。

```text
direnv exec
devbox run
mise exec
npx
docker exec
```

したがって、

```json
"allow": ["Bash(devbox run *)"]
```

は `devbox run rm -rf .` も許可してしまいます。

安全に書くなら、ランナーと内側のコマンドを両方含めます。

```json
"allow": ["Bash(devbox run npm test)"]
```

また、

```text
watch / setsid / ionice / flock
find の -exec / -delete
```

は prefix ルールでは自動承認されず、Manualモードでは常に確認が入ります。承認したいなら完全一致のルールを書きます。

---

# 18. リダイレクトの扱い

出力リダイレクトの**書き込み先はファイル書き込みとして判定されます**。

```text
>  >>  2>
    ↓
Edit の allow / deny
protected paths
working directories
に照らして確認される
```

`Bash(git commit *)` はコマンドを許可するだけで、リダイレクト先までは許可しません。

```text
/dev/null  → 判定されない
~ で始まる → 承認が必要
glob文字を含む → 承認が必要
```

---

# 19. PowerShellルール

Bashと同じ形です。

```json
{
  "permissions": {
    "allow": [
      "PowerShell(Get-ChildItem *)",
      "PowerShell(git commit *)"
    ],
    "deny": [
      "PowerShell(Remove-Item *)"
    ]
  }
}
```

特徴:

```text
エイリアスは正規化される
    ↓
PowerShell(Get-ChildItem *) は gci / ls / dir にもマッチ

マッチは大文字小文字を区別しない

ASTを解析して複合コマンドを分割
    ↓
|  ;  （PowerShell 7+では && ||）
    ↓
全サブコマンドがマッチする必要がある
```

---

# 20. `Read` と `Edit` のルール

ファイルアクセスの制御は、**`Read` と `Edit` の2つだけ**で行います。

```text
Read(path)
    ↓
読み取り系

Edit(path)
    ↓
書き込み系すべて（Write / Edit を含む）
```

重要な制約があります。

```text
Write(...)      → 受け付けるが参照されない
NotebookEdit(...) → 同上
Glob(...)       → 同上
MultiEdit(...)  → 同上
```

つまり、

```text
× Write(docs/**)
○ Edit(docs/**)

× Glob(docs/**)
○ Read(docs/**)
```

と書きます。誤ったルールは起動時に警告されます。

なお `Read` のdenyルールは同じパスのEdit / Writeもブロックしますが、**NotebookEditは対象外**なので、どのToolにも変更させたくないパスには `Edit` のdenyも併記します。

---

# 21. パスパターンの4つのアンカー

`Read` / `Edit` は gitignore 構文を使い、先頭の書き方で基準が変わります。

| パターン | 意味 | 例 | 展開先 |
|---|---|---|---|
| `//path` | ファイルシステムのルートからの絶対パス | `Read(//Users/alice/secrets/**)` | `/Users/alice/secrets/**` |
| `~/path` | ホームディレクトリ基準 | `Read(~/Documents/*.pdf)` | `/Users/alice/Documents/*.pdf` |
| `/path` | **settingsファイルの場所**が基準 | `Edit(/src/**/*.ts)` | `<主作業ディレクトリ>/src/**/*.ts` |
| `path` / `./path` | カレントディレクトリ基準 | `Read(*.env)` | `<cwd>/*.env` |

最も間違えやすいのがスラッシュ1つです。

```text
/Users/alice/file
    ↓
絶対パスではない
    ↓
settingsソース基準

//Users/alice/file
    ↓
これが絶対パス
```

---

# 22. Read / Edit denyの限界

```text
効く
    ↓
Claudeの組み込みファイルTool
Claude Codeが認識するBashのファイルコマンド
（cat / head / tail / sed など）

効かない
    ↓
任意のサブプロセスが間接的に読む場合
（Python や Node のスクリプトが自分で open する等）
```

OSレベルで全プロセスを止めたいなら sandbox を有効にします。

```text
permissions.deny
    ↓
Claude Codeが認識する経路をブロック

sandbox
    ↓
OSレベルで遮断
```

---

# 23. `WebFetch` のルール

2つの書き方で効果が変わります。

| ルール | allowにしたとき | denyにしたとき |
|---|---|---|
| `WebFetch`（bare） | 確認なしでfetch。sandboxの到達先は変えない | Toolごと削除。sandboxの到達先は変えない |
| `WebFetch(domain:*)` | 確認なしでfetch。**sandboxからも任意ホストへ到達可** | Toolは残り各fetchを拒否。**sandboxからどこへも到達不可** |

`domain:` 形式だけが sandbox の allowed / denied ドメインリストにも反映されます。

```text
Claudeには自由にfetchさせたいが
sandboxのallowlistは変えたくない
    ↓
bare形式を使う
```

---

# 24. MCPのルール

サーバー名を使います。

```text
mcp__puppeteer
    ↓
puppeteerサーバーの全tool

mcp__puppeteer__*
    ↓
同上（ワイルドカード形式）

mcp__puppeteer__puppeteer_navigate
    ↓
特定のtoolだけ
```

allowルールでのグロブには制約があります。

```text
mcp__<server>__ のリテラル接頭辞が必要
サーバー部分にグロブは使えない

○ mcp__puppeteer__*
○ mcp__github__get_*
× "*"
× "B*"
× "mcp__*"
```

アンカーのないallowグロブは警告付きでスキップされ、何も自動承認しません。

---

# 25. `Agent` のルール

Subagentの使用可否を制御します。

```json
{
  "permissions": {
    "deny": ["Agent(Explore)", "Agent(my-custom-agent)"]
  }
}
```

CLIからも指定できます。

```bash
claude --disallowedTools "Agent(Explore)"
```

---

# 26. `Cd` のルール

`/cd` コマンドの移動先を制御します。

```text
Cd はモデルが呼べるToolではない
    ↓
ユーザーが /cd を実行したときだけ適用される
```

挙動:

```text
bare の Cd を deny
    ↓
/cd を完全に無効化

Cd(<pattern>) を deny
    ↓
マッチする移動先をブロック

Cd の allow を1つでも書く
    ↓
allowlistモードになり、マッチしない移動先は拒否
```

パターンは `//`、`~/`、`/` のアンカーを共有しますが、gitignore式ではなくディレクトリパス全体へのアンカー方式です。

| ルール | マッチする | マッチしない |
|---|---|---|
| `Cd(~/code/*)` | `~/code/app` | `~/code/app/src`、`~/code` |
| `Cd(~/code/**)` | `~/code` とその配下すべて | `~/code` 外 |
| `Cd(**/node_modules)` | 任意の深さの `node_modules` | `node_modules/pkg` |

denyルールはシンボリックリンクの各ホップも含めてすべての綴りを検査します。

---

# 27. Permission modes

セッション全体の既定挙動を決めます。

| モード | 挙動 |
|---|---|
| `default`（別名 `manual`） | 各Toolの初回使用時に確認する。UI上は Manual と表示 |
| `acceptEdits` | 作業ディレクトリ内のファイル編集と `mkdir` / `touch` / `mv` / `cp` などを自動承認 |
| `plan` | 読み取りと read-only コマンドのみ。ソースは編集しない |
| `auto` | バックグラウンドの分類器が安全性を確認して自動承認 |
| `dontAsk` | `permissions.allow` と read-only コマンド以外は自動拒否 |
| `bypassPermissions` | 確認をスキップ（ごく一部の例外を除く） |

設定では、

```json
{
  "permissions": {
    "defaultMode": "acceptEdits"
  }
}
```

CLIでは、

```bash
claude --permission-mode auto
```

です。

`dontAsk` は「確認できない環境で、許可したものだけ動かす」CI向けの選択肢です。

---

# 28. 作業ディレクトリ

Claudeが触れるのは、既定では**起動したディレクトリ**だけです。

拡張する方法が3つあります。

```text
起動時
    ↓
claude --add-dir <path>

セッション中
    ↓
/add-dir

恒久設定
    ↓
permissions.additionalDirectories
```

追加ディレクトリのファイルは、元の作業ディレクトリと同じpermissionルールに従います。

---

# 29. 追加ディレクトリは「設定」を読み込まない

ここは誤解しやすい点です。

```text
--add-dir / /add-dir
    ↓
ファイルアクセスを与える
    ↓
ただし設定ルートにはならない
```

`--add-dir` から読み込まれるものと読み込まれないものは次の通りです。

| 種類 | `--add-dir` から読むか |
|---|---|
| `.claude/skills/` | ○（live reloadあり） |
| `.claude/commands/` | ○（live reloadなし。同名なら自プロジェクト優先） |
| `.claude/agents/` | ○（live reloadなし） |
| `.claude/settings.json` | `enabledPlugins` と `extraKnownMarketplaces` のみ |
| `CLAUDE.md` / `.claude/rules/` | `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1` のときだけ |

さらに重要な差として、

```text
permissions.additionalDirectories に書いた場合
    ↓
ファイルアクセスだけ
    ↓
skills も commands も agents も読まれない
```

設定を共有したいなら、user-levelに置くかPluginにします。

---

# 30. sandboxとの関係

permissionsとsandboxは補完的な層です。

```text
permissions
    ↓
Claude Codeが認識する操作を制御

sandbox
    ↓
OSレベルでプロセスを隔離
```

組み合わせ方:

```text
filesystem
    ↓
sandbox.filesystem の設定
    +
Read / Edit の deny ルール
    ↓
両方をマージして境界を決める

network
    ↓
WebFetch(domain:...) のルール
    +
sandbox の allowedDomains / deniedDomains
```

sandboxを有効にし `autoAllowBashIfSandboxed` を既定の `true` のままにすると、bare の `Bash` askルールがあってもsandbox内のBashは確認なしで動きます。sandbox境界がその確認の代わりになるためです。

ただしplanモードではこの置き換えは行われません。

---

# 31. Hooksとの関係

評価の流れは次の通りです。

```text
Claudeが Tool を呼ぶ
        ↓
   PreToolUse Hook
        ↓
   permission rules
   （deny → ask → allow）
        ↓
      実行
```

重要な原則:

```text
Hookが "allow" を返しても
deny / ask ルールは無効化されない

Hookが exit 2 を返すと
permission rules より前にブロックされる
```

つまり、

```text
Hookは「厳しくする」方向には確実に効く
「緩める」方向には permission rule が優先される
```

これを利用した定番パターンがあります。

```text
allow に "Bash" を入れて全許可
        +
PreToolUse Hook で特定コマンドだけ拒否
        ↓
「原則許可・例外拒否」を実現
```

denyルールに例外が書けないため、この形が必要になります。

---

# 32. Workspace trustと非対話モード

信頼していないフォルダでは、project settingsのallowルールはそのままは適用されません。

一方で、非対話モード（`claude -p`）には注意が必要です。

```text
claude -p
    ↓
workspace trustダイアログは出ない
MCPサーバーの承認プロンプトも出ない
    ↓
プロジェクトの .claude/settings.json のHookが走る
.mcp.json のサーバーへ接続する
```

CIで他人のリポジトリを扱うなら、

```bash
claude --bare -p "..." --allowedTools "Read"
```

のように `--bare` で読み込ませないのが安全です。

---

# 33. 設定例：個人

`~/.claude/settings.json`:

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "permissions": {
    "allow": [
      "Bash(git status *)",
      "Bash(git diff *)",
      "Bash(git log *)",
      "Bash(rg *)"
    ],
    "deny": [
      "Read(~/.ssh/**)",
      "Read(~/.aws/**)",
      "Edit(~/.ssh/**)"
    ]
  }
}
```

「どのプロジェクトでも常に安全側にしたいもの」だけを置きます。

---

# 34. 設定例：チーム

`.claude/settings.json`（コミットする）:

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "permissions": {
    "allow": [
      "Bash(npm ci)",
      "Bash(npm run lint)",
      "Bash(npm run test *)",
      "Bash(npm run typecheck)",
      "Bash(npm run build)"
    ],
    "ask": [
      "Bash(git push *)"
    ],
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Edit(./.env)",
      "Bash(npm publish *)",
      "Bash(terraform apply *)"
    ]
  },
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write"
          }
        ]
      }
    ]
  }
}
```

方針:

```text
allow
    ↓
毎回聞かれると邪魔な、安全な定型コマンド

ask
    ↓
実行してよいが必ず確認したいもの

deny
    ↓
理由を問わず実行させないもの
```

---

# 35. 設定例：組織

managed settingsでは、開発者が上書きできない項目を置きます。

```json
{
  "permissions": {
    "deny": [
      "Read(**/.env)",
      "Read(**/id_rsa)",
      "Bash(aws *)",
      "Bash(kubectl *)"
    ]
  },
  "allowManagedHooksOnly": true,
  "claudeMd": "Never commit secrets.\nTreat production data as read-only."
}
```

`claudeMd` キーを使うと、別ファイルを配布せずにmanaged CLAUDE.mdの内容を設定へ直接書けます。

役割分担:

```text
Managed settings
    ↓
技術的な強制

Managed CLAUDE.md
    ↓
行動指針
```

---

# 36. よくある落とし穴

実務で踏みやすいものを挙げます。

```text
① denyに例外を書こうとする
     → 先に評価されるので例外は成立しない
     → allow + PreToolUse Hook で書く

② Write(...) や Glob(...) でパスを制御しようとする
     → 参照されない
     → Edit(...) / Read(...) を使う

③ /path が絶対パスだと思う
     → settingsソース基準
     → 絶対パスは //path

④ Bash(git * main) のような書き方
     → git -c 経由で任意プログラム実行が可能

⑤ Bash(devbox run *) のようなランナー許可
     → 内側の任意コマンドを許可してしまう

⑥ Bash(ls*) と Bash(ls *) を混同
     → 前者は lsof にもマッチ

⑦ CLAUDE.mdだけで危険操作を防ごうとする
     → 強制力がない

⑧ CIで --bare を付けない
     → リポジトリのhooksやMCPが動く
```

---

# 37. 権限設計の順序

新しいプロジェクトで組むときの順序です。

```text
① 絶対に触らせないものを deny に書く
        ↓
② 毎回聞かれると邪魔なものを allow に書く
        ↓
③ 実行はさせたいが確認したいものを ask に書く
        ↓
④ 個人的な緩和を settings.local.json に置く
        ↓
⑤ /status でロードを確認
        ↓
⑥ 実タスクで確認回数を見て調整
```

最初から広く allow せず、**確認が煩わしくなった順に足していく**ほうが安全です。

---

# 38. permissionsとCLAUDE.mdの併記

強制と説明はセットにすると効果的です。

```json
{
  "permissions": {
    "deny": ["Bash(git push *)"]
  }
}
```

に加えて、CLAUDE.mdへ:

```markdown
# Git

- Pushはユーザーが手動で行う。Claudeはpushしない。
- Push相当の操作が必要な場合は、コマンドを提示して確認を求める。
```

理由は、

```text
permissionsだけ
    ↓
Claudeは「なぜ失敗したか」を知らない
    ↓
別の手段で回避しようとする

CLAUDE.mdも書く
    ↓
最初から試みない
    ↓
無駄なやり取りが減る
```

からです。

---

# 39. 最終チェックリスト

```text
[ ] スコープの選択が正しい（共有すべきものがlocalに入っていない）
[ ] .claude/settings.local.json が .gitignore に入っている
[ ] strict JSONとして壊れていない
[ ] denyに例外を書こうとしていない
[ ] ファイルパスは Read / Edit で書いている
[ ] 絶対パスは // で始めている
[ ] Bashルールの * がサブコマンドの後にある
[ ] 環境ランナーを丸ごと許可していない
[ ] MCPのallowグロブがサーバー名までリテラルになっている
[ ] additionalDirectories に設定読み込みを期待していない
[ ] CIでは --bare を検討した
[ ] /status でロードを確認した
[ ] 強制したいものをCLAUDE.mdだけに頼っていない
```

---

# 40. まとめ

`settings.json` と `permissions` の構成を最も簡潔にまとめると、

```text
settings.json
│
├─ スコープ
│    ↓
│  Managed > CLI > local > project > user
│
├─ permissions
│    │
│    ├─ deny   … 常に拒否（最優先・例外不可）
│    ├─ ask    … 必ず確認
│    ├─ allow  … 確認なしで許可
│    ├─ additionalDirectories … ファイルアクセスの拡張
│    └─ defaultMode … セッション既定モード
│
├─ hooks
│    ↓
│  実行時の決定論的処理
│
└─ その他
     ↓
   model / env / plugin / telemetry 等
```

そしてClaude Code全体での位置づけは、

```text
CLAUDE.md / rules
=
Claudeに判断させる

settings.json の permissions
=
静的に許可・拒否する

Hooks
=
実行時に判断して介入する

sandbox
=
OSレベルで隔離する
```

です。

上の層ほど柔軟で、下の層ほど確実です。

```text
柔軟だが確実でないもの
    ↓
CLAUDE.md

確実だが柔軟でないもの
    ↓
sandbox
```

危険度に応じて層を選び、重要なものは複数の層で重ねるのが実務的な設計です。

---

# 参考資料

Anthropic公式 Claude Code Docs:

- Claude Code settings  
  https://code.claude.com/docs/en/settings

- Configure permissions  
  https://code.claude.com/docs/en/permissions

- Settings reference  
  https://code.claude.com/docs/en/settings-reference

- Permission modes  
  https://code.claude.com/docs/en/permission-modes

- Sandboxing  
  https://code.claude.com/docs/en/sandboxing

特に確認するとよい項目:

- Settings files and who they affect
- Settings precedence
- Permission rule syntax
- Wildcard patterns
- Tool-specific permission rules
- Working directories
- Additional directories grant file access, not configuration
- How permissions interact with sandboxing
- Project allow rules and workspace trust
