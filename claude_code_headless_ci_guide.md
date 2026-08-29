# Claude Code：非対話モード（headless / CI）の構成についての解説

> 対象: Claude Code の `claude -p`（非対話モード / Agent SDK CLI）  
> 更新基準: 2026-08-29 時点のAnthropic公式ドキュメント  
> 目的: スクリプトやCIからClaude Codeを呼ぶときに「何を渡し、何を受け取り、何に注意すべきか」を、設計意図も含めて理解する

---

<!-- toc -->
# 目次

- [1. 非対話モードとは何か](#1-非対話モードとは何か)
- [2. 基本形](#2-基本形)
- [3. 終了コード](#3-終了コード)
- [4. --bare ― CIでは基本これ](#4---bare--ciでは基本これ)
- [5. bare modeで使えるもの・渡すもの](#5-bare-modeで使えるもの渡すもの)
- [6. bare modeの認証](#6-bare-modeの認証)
- [7. stdinからのパイプ](#7-stdinからのパイプ)
- [8. 出力形式](#8-出力形式)
- [9. --json-schema ― スキーマに従わせる](#9---json-schema--スキーマに従わせる)
- [10. jq での取り出し](#10-jq-での取り出し)
- [11. ストリーミング](#11-ストリーミング)
- [12. Subagentのメッセージを追う](#12-subagentのメッセージを追う)
- [13. system/init ― セッションのメタデータ](#13-systeminit--セッションのメタデータ)
- [14. CIをPluginやMCPの失敗で落とす](#14-ciをpluginやmcpの失敗で落とす)
- [15. api_retry ― リトライの可視化](#15-api_retry--リトライの可視化)
- [16. Toolの事前承認](#16-toolの事前承認)
- [17. Permission modeで一括指定する](#17-permission-modeで一括指定する)
- [18. 会話の継続](#18-会話の継続)
- [19. system promptのカスタマイズ](#19-system-promptのカスタマイズ)
- [20. バックグラウンドタスクの扱い](#20-バックグラウンドタスクの扱い)
- [21. SIGTERMでの停止](#21-sigtermでの停止)
- [22. SkillとコマンドをCIで使う](#22-skillとコマンドをciで使う)
- [23. CIにおけるセキュリティ](#23-ciにおけるセキュリティ)
- [24. 実例：typo linter](#24-実例typo-linter)
- [25. 実例：PRレビュー](#25-実例prレビュー)
- [26. 実例：構造化データの抽出](#26-実例構造化データの抽出)
- [27. 実例：CIゲート](#27-実例ciゲート)
- [28. アンチパターン](#28-アンチパターン)
- [29. 最終チェックリスト](#29-最終チェックリスト)
- [30. まとめ](#30-まとめ)
- [参考資料](#参考資料)

<!-- /toc -->

---

# 1. 非対話モードとは何か

`-p`（`--print`）を付けると、Claude Codeは**対話UIを起動せず、1回の応答を返して終了**します。

```bash
claude -p "Find and fix the bug in auth.py" --allowedTools "Read,Edit,Bash"
```

```text
対話モード
    ↓
人が確認しながら進める

非対話モード
    ↓
人がいない前提で完結する
    ↓
権限も入力も事前に決めておく必要がある
```

この違いが、後述するほぼすべての注意点の根拠になります。

---

# 2. 基本形

```bash
claude -p "What does the auth module do?"
```

`-p` と組み合わせられないオプションもあります。

```text
--bg          → エラー
--cloud + タスク説明 → エラー
--cloud + セッションID + -p
    → そのcloudセッションへメッセージを積んで終了
```

よく併用するもの:

```text
--continue      … 会話の継続
--allowedTools  … Toolの事前承認
--output-format … 構造化出力
--bare          … 起動を軽くし環境依存を排除
```

---

# 3. 終了コード

```text
0        … 成功
非0      … 失敗
143      … SIGTERMで停止
```

無効なフラグを渡した場合は、実行開始前にstderrへ報告されます。

一方、認証切れのように**実行中に起きた失敗は、stdoutへ結果として出力されます**。

```text
フラグの誤り
    ↓
stderr、開始前

実行中の失敗
    ↓
stdout、結果として
```

スクリプトでは終了コードで分岐しつつ、`--output-format json` の中身も確認するのが確実です。

---

# 4. `--bare` ― CIでは基本これ

`--bare` を付けると、以下の自動探索を**すべてスキップ**します。

```text
hooks
skills
custom commands
subagents
plugins
MCPサーバー
auto memory
CLAUDE.md
```

理由は明確で、

```text
--bare なし
    ↓
チームメイトの ~/.claude のhookが走る
プロジェクトの .mcp.json のサーバーへ繋がる
    ↓
マシンによって結果が変わる
```

からです。

```text
CI / スクリプト
    ↓
どのマシンでも同じ結果が要る
    ↓
--bare
```

公式にも「スクリプトやSDK呼び出しでは `--bare` が推奨で、将来 `-p` の既定になる」と書かれています。

---

# 5. bare modeで使えるもの・渡すもの

bare modeでもBash、ファイル読み取り、ファイル編集は使えます。

必要なものはフラグで明示的に渡します。

| 読み込ませたいもの | フラグ |
|---|---|
| system promptの追加 | `--append-system-prompt`、`--append-system-prompt-file` |
| 設定 | `--settings <file-or-json>` |
| MCPサーバー | `--mcp-config <file-or-json>` |
| カスタムagent | `--agents <json>` |
| Plugin | `--plugin-dir <path>`、`--plugin-url <url>` |

例外として、

```text
--add-dir で指定したディレクトリ
    ↓
.claude/skills/ は読まれる
.claude/commands/ と .claude/agents/ は読まれない
```

---

# 6. bare modeの認証

重要な注意点です。

```text
--bare
    ↓
OAuth資格情報もシステムキーチェーンも読まない
    ↓
サブスクリプションログインは使えない
```

したがって、

```bash
export ANTHROPIC_API_KEY=sk-ant-...
claude --bare -p "Summarize README.md" --allowedTools "Read"
```

のようにAPIキーを渡すか、`--settings` のJSONで `apiKeyHelper` を指定します。

Amazon Bedrock / Google Cloud の Agent Platform / Microsoft Foundry は、それぞれのプロバイダ資格情報を通常どおり読みます。

---

# 7. stdinからのパイプ

非対話モードは標準入力を読みます。

```bash
cat build-error.txt | claude -p 'concisely explain the root cause of this build error' > output.txt
```

制限:

```text
パイプ入力は10MBまで
    ↓
超えるとエラーで非0終了
    ↓
大きい入力はファイルに書いてパスを渡す
```

パイプする利点は権限面にもあります。

```text
git diff main | claude -p "..."
    ↓
Claudeは git を実行しなくてよい
    ↓
Bash権限が要らない
```

---

# 8. 出力形式

```text
text（既定）
    ↓
プレーンテキスト

json
    ↓
result / session_id / メタデータを含む構造化JSON

stream-json
    ↓
改行区切りJSONのリアルタイムストリーム
```

```bash
claude -p "Summarize this project" --output-format json
```

`--output-format json` では `total_cost_usd` とモデル別のコスト内訳も返るため、呼び出しごとのコストを追えます。これらはクライアント側の推定値で、実際の請求とは差が出ることがあります。

---

# 9. `--json-schema` ― スキーマに従わせる

決まった形で受け取りたいときに使います。

```bash
claude -p "Extract the main function names from auth.py" \
  --output-format json \
  --json-schema '{"type":"object","properties":{"functions":{"type":"array","items":{"type":"string"}}},"required":["functions"]}'
```

結果は `structured_output` フィールドに入ります。

```text
result
    ↓
テキストの応答

structured_output
    ↓
スキーマに従った構造化データ
```

注意点:

```text
無効なJSON Schemaを渡すと
    ↓
Error: --json-schema is not a valid JSON Schema
で終了する

"format": "email" などは受け付けるが
注釈として扱われ、検証はされない
```

---

# 10. `jq` での取り出し

```bash
# テキスト結果
claude -p "Summarize this project" --output-format json | jq -r '.result'

# 構造化出力
claude -p "Extract function names from auth.py" \
  --output-format json \
  --json-schema '{"type":"object","properties":{"functions":{"type":"array","items":{"type":"string"}}},"required":["functions"]}' \
  | jq '.structured_output'
```

---

# 11. ストリーミング

トークンを逐次受け取ります。

```bash
claude -p "Explain recursion" --output-format stream-json --verbose --include-partial-messages
```

```text
--output-format stream-json
    +
--verbose
    +
--include-partial-messages
```

の3つが必要です。

最終行は `result` メッセージで、応答テキスト・コスト・セッションメタデータが入ります。

テキストデルタだけを流す例:

```bash
claude -p "Write a poem" --output-format stream-json --verbose --include-partial-messages | \
  jq -rj 'select(.type == "stream_event" and .event.delta.type? == "text_delta") | .event.delta.text'
```

`-r` は生文字列出力、`-j` は改行なしの連結です。

---

# 12. Subagentのメッセージを追う

```text
parent_tool_use_id
    ↓
null            → メイン会話のメッセージ
tool call の ID → そのSubagentのメッセージ
```

既定ではSubagentの `tool_use` と `tool_result` ブロックだけが流れます。テキストとthinkingも見たい場合:

```bash
claude -p "..." --output-format stream-json --verbose --forward-subagent-text
```

環境変数 `CLAUDE_CODE_FORWARD_SUBAGENT_TEXT` でも指定できます。

有効にすると、入れ子のSubagentのメッセージも流れ、`parent_tool_use_id` を辿って階層を再構成できます。

---

# 13. `system/init` ― セッションのメタデータ

ストリームの先頭付近に届く、セッション情報のイベントです。

```text
model
tools
mcp_servers
plugins
capabilities
```

などが入ります。

`capabilities` はこのバージョンが実装しているプロトコル挙動の名前の配列です。バージョン文字列を比較するのではなく、**この配列で機能検出する**のが推奨です。知らない値は無視します。

なお、`system/init` より前に届くことがあるイベントがあります。

```text
plugin_install イベント
    （CLAUDE_CODE_SYNC_PLUGIN_INSTALL 設定時）

hook_started / hook_progress / hook_response
    （SessionStart や Setup のHook実行中）
```

---

# 14. CIをPluginやMCPの失敗で落とす

`system/init` のフィールドを使って検知します。

| フィールド | 内容 |
|---|---|
| `plugins` | 正常にロードされたPlugin（`name`、`path`） |
| `plugin_errors` | ロード時エラー（`plugin`、`type`、`message`）。エラーが無ければキー自体が省略される |
| `mcp_servers` | セッション内のMCPサーバー（`name`、`status`） |
| `mcp_server_errors` | 検証で除外された `--mcp-config` エントリ（`name`、`type`、`message`）。エラーが無ければキー自体が省略される |

```text
設定ミスがあっても
    ↓
runは続行し、正常終了する
    ↓
黙って機能が欠けたまま通る
```

ため、CIゲートではこの2つの配列が空でないことをチェックします。

```bash
claude --bare -p "..." --output-format stream-json --verbose --mcp-config ./mcp.json \
  | tee out.jsonl

if jq -e 'select(.type=="system" and .subtype=="init") | .mcp_server_errors // empty' out.jsonl > /dev/null; then
  echo "MCP server config error" >&2
  exit 1
fi
```

`--mcp-config` を `-p` と併用すると、Claude Codeは最初のターンの前に未接続サーバーを待ちます（既定30秒、`MCP_TIMEOUT` で変更可）。

---

# 15. `api_retry` ― リトライの可視化

APIがリトライ可能なエラーを返すと、リトライ前に `system/api_retry` イベントが流れます。

| フィールド | 内容 |
|---|---|
| `attempt` | 試行回数（1始まり） |
| `max_retries` | 許可された総リトライ数 |
| `retry_delay_ms` | 次の試行までのミリ秒 |
| `error_status` | HTTPステータス、接続エラーなら `null` |
| `error` | `rate_limit` / `overloaded` / `authentication_failed` などのカテゴリ |

長時間のCIジョブで「止まっているのかリトライ中なのか」を区別するのに使えます。

---

# 16. Toolの事前承認

非対話モードでは確認プロンプトを出せないため、事前に許可します。

```bash
claude -p "Run the test suite and fix any failures" \
  --allowedTools "Bash,Read,Edit"
```

`--allowedTools` はpermission rule構文です。

```bash
claude -p "Look at my staged changes and create an appropriate commit" \
  --allowedTools "Bash(git diff *),Bash(git log *),Bash(git status *),Bash(git commit *)"
```

```text
末尾の「空白 + *」で前方一致
    ↓
Bash(git diff *) は git diff で始まるコマンドを許可

空白は重要
    ↓
Bash(git diff*) は git diff-index にもマッチしてしまう
```

---

# 17. Permission modeで一括指定する

個別に列挙するかわりに、セッション全体の既定を決められます。

```text
-p の既定は Manual（default）
    ↓
プランに関係なく、明示しないと何も自動承認されない
```

主な選択肢:

```text
--permission-mode auto
    ↓
分類器がほとんどの操作を確認して自動承認

--permission-mode dontAsk
    ↓
permissions.allow と read-onlyコマンド以外は自動拒否
    ↓
締めたCI向け

--permission-mode acceptEdits
    ↓
ファイル書き込みと mkdir / touch / mv / cp を自動承認
    ↓
それ以外のシェルコマンドやネットワークは別途許可が必要
```

例:

```bash
claude -p "Apply the lint fixes" --permission-mode acceptEdits
```

`dontAsk` では、`AskUserQuestion` や `requiresUserInteraction` 付きのMCP toolは、allowルールがあっても拒否されます。

---

# 18. 会話の継続

```bash
# 1回目
claude -p "Review this codebase for performance issues"

# 直近の会話を継続
claude -p "Now focus on the database queries" --continue
claude -p "Generate a summary of all issues found" --continue
```

複数の会話を並行させるならセッションIDを捕まえます。

```bash
session_id=$(claude -p "Start a review" --output-format json | jq -r '.session_id')
claude -p "Continue that review" --resume "$session_id"
```

```text
--continue
    ↓
直近の会話（バックグラウンドセッションはスキップ）

--resume <id>
    ↓
特定の会話
    ↓
このマシン上のどのプロジェクトからでも見つかる
```

---

# 19. system promptのカスタマイズ

```bash
gh pr diff "$1" | claude -p \
  --append-system-prompt "You are a security engineer. Review for vulnerabilities." \
  --output-format json
```

```text
--append-system-prompt
    ↓
既定の振る舞いを保ったまま追記

--system-prompt
    ↓
既定のプロンプトを完全に置き換える
```

`--append-system-prompt-file` でファイルからも渡せます。

`--bare` でCLAUDE.mdを読まない構成では、必要な前提をここで渡すことになります。

---

# 20. バックグラウンドタスクの扱い

```text
Claudeが起動したバックグラウンドBashタスク
    ↓
最終結果を返しstdinが閉じた約5秒後に終了させられる
```

猶予は、直後に終わるタスクの出力を拾うためのものです。

一方、

```text
バックグラウンドのSubagentとWorkflow
    ↓
結果が最終出力の一部なので待つ
    ↓
既定で最大10分
```

上限は `CLAUDE_CODE_PRINT_BG_WAIT_CEILING_MS` で変更でき、`0` で無制限になります。

```text
CIが謎に長い
    ↓
バックグラウンドSubagentの待ちを疑う
```

---

# 21. SIGTERMでの停止

```text
SIGTERM（kill、プロセス管理からの停止）
    ↓
終了コード 143
    ↓
進行中のターンは未完のまま、結果は記録されない
```

ターンを終わらせたい場合はSIGINT、またはAgent SDKの `interrupt()` を先に呼びます。

SIGTERM受信時の挙動:

```text
実行中のBashコマンドのプロセスツリーを終了
        ↓
SessionEnd Hookを実行
        ↓
終了

終了処理中は
新しいtool呼び出しもモデルリクエストも行わない
SessionEnd以外のHookも実行しない
```

セッションを再開すると、SIGTERMで中断されたターンから継続されます。

---

# 22. SkillとコマンドをCIで使う

```text
/skill-name をプロンプト文字列に含める
        ↓
Claude Codeが展開してから実行する
```

```bash
claude -p "/security-review src/api/"
```

制約:

```text
ターミナル専用の組み込みコマンド（/login 等）は使えない

/model /effort /fast /color /rename は
値を引数として受け付ける（例: /model sonnet）

/mcp は引数なしでサーバー状態のテキスト要約を出す

/config は key=value を受け付ける（例: /config thinking=false）
```

ただし `--bare` ではSkillが読み込まれないため、Skillを使いたいなら `--bare` を外すか、`--plugin-dir` でSkillを含むPluginを明示的に渡します。

---

# 23. CIにおけるセキュリティ

**最重要**の注意点です。

```text
claude -p（--bare なし）
    ↓
workspace trustダイアログが出ない
MCPサーバーの承認プロンプトも出ない
    ↓
一度も信頼していないフォルダでも
  .claude/settings.json のhookが走る
  .mcp.json のサーバーへ接続する
```

つまり、

```text
fork PRのコードをCIでチェックさせる
        ↓
そのPRが .claude/settings.json にhookを追加していたら
        ↓
CIランナー上で任意コマンドが走る
```

という経路が成立します。

対策:

```text
① --bare を使う（推奨）
② 必要な設定は --settings で明示的に渡す
③ 信頼できないPRでは権限を最小にする
   （--permission-mode dontAsk + 必要最小の --allowedTools）
④ シークレットを環境変数として渡さない、
   または渡す範囲を絞る
⑤ 差分はパイプで渡し、Bash権限を与えない
```

---

# 24. 実例：typo linter

`package.json`:

```json
{
  "scripts": {
    "lint:claude": "git diff main | claude -p \"you are a typo linter. for each typo in this diff, report filename:line on one line and the issue on the next. return nothing else.\""
  }
}
```

```text
diffをパイプで渡す
    ↓
ClaudeはBash権限が不要
    ↓
エスケープした二重引用符でWindowsでも動く
```

---

# 25. 実例：PRレビュー

`review.sh`:

```bash
#!/bin/bash
gh pr diff "$1" | claude --bare -p \
  --append-system-prompt "You are a security engineer. Review for vulnerabilities." \
  --output-format json \
  | jq -r '.result'
```

```bash
bash review.sh 123
```

---

# 26. 実例：構造化データの抽出

```bash
claude --bare -p "このリポジトリで外部HTTPリクエストを行っている箇所を列挙して" \
  --allowedTools "Read,Grep,Glob" \
  --output-format json \
  --json-schema '{
    "type": "object",
    "properties": {
      "findings": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "file": {"type": "string"},
            "line": {"type": "integer"},
            "reason": {"type": "string"}
          },
          "required": ["file", "line", "reason"]
        }
      }
    },
    "required": ["findings"]
  }' | jq '.structured_output.findings'
```

```text
--allowedTools を読み取り系だけに絞る
    +
--json-schema で形を固定する
    ↓
CIで機械的に扱える
```

---

# 27. 実例：CIゲート

```bash
#!/bin/bash
set -euo pipefail

export ANTHROPIC_API_KEY="${ANTHROPIC_API_KEY:?}"

OUT=$(git diff origin/main...HEAD | claude --bare -p \
  "この差分に、テストを弱めている変更、秘密情報のハードコード、
   public APIの破壊的変更が含まれるか判定して。" \
  --output-format json \
  --json-schema '{
    "type":"object",
    "properties":{
      "blocking":{"type":"boolean"},
      "reasons":{"type":"array","items":{"type":"string"}}
    },
    "required":["blocking","reasons"]
  }')

if [ "$(echo "$OUT" | jq -r '.structured_output.blocking')" = "true" ]; then
  echo "$OUT" | jq -r '.structured_output.reasons[]' >&2
  exit 1
fi
```

ポイント:

```text
--bare        … ホスト環境に依存しない
パイプ入力    … Toolも権限も不要
--json-schema … boolで判定できる
exit 1        … CIを落とせる
```

---

# 28. アンチパターン

```text
× CIで --bare を付けない
     リポジトリのhooksとMCPが走る

× --allowedTools に "Bash" とだけ書く
     任意コマンド実行を許すのと同じ

× 対話前提の指示を書く
     非対話モードでは聞き返せない
     推測で進む

× 出力形式を決めずにパースする
     text出力を正規表現で解析すると壊れる
     --json-schema を使う

× 巨大な入力をパイプする
     10MB上限。ファイルパスを渡す

× --bare でサブスクリプションログインを期待する
     ANTHROPIC_API_KEY が必要

× 失敗を終了コードだけで判定する
     実行中の失敗はstdoutの結果として出る
     plugin_errors / mcp_server_errors も見る

× タイムアウトを設けない
     バックグラウンドSubagentの待ちで長引く
```

---

# 29. 最終チェックリスト

```text
[ ] CIでは --bare を付けた
[ ] 認証方法を決めた（ANTHROPIC_API_KEY / provider資格情報）
[ ] --allowedTools が必要最小限
[ ] permission mode を明示した
[ ] 出力を --output-format json で受けている
[ ] 機械処理するなら --json-schema を使っている
[ ] 終了コードと結果の両方を確認している
[ ] plugin_errors / mcp_server_errors をチェックしている
[ ] 入力が10MBを超えない
[ ] 対話を前提とした指示になっていない
[ ] 信頼できないPRで権限を絞っている
[ ] シークレットの露出範囲を確認した
[ ] ジョブ全体のタイムアウトを設定した
[ ] コストを --output-format json で記録している
```

---

# 30. まとめ

非対話モードの構成を最も簡潔にまとめると、

```text
claude -p "<prompt>"
│
├─ 起動制御
│    ↓
│  --bare（環境依存を排除）
│  --settings / --mcp-config / --agents / --plugin-dir
│
├─ 入力
│    ↓
│  プロンプト文字列
│  stdin（10MBまで）
│  --append-system-prompt
│
├─ 権限
│    ↓
│  --allowedTools
│  --permission-mode
│
├─ 出力
│    ↓
│  --output-format text | json | stream-json
│  --json-schema
│
└─ セッション
     ↓
   --continue / --resume
```

対話モードとの本質的な違いは、

```text
確認できない
    ↓
権限は事前に決める
指示に曖昧さを残さない
出力形式を固定する

信頼確認のダイアログが出ない
    ↓
リポジトリの設定が黙って有効になる
    ↓
--bare で遮断する
```

の2点です。

対話モードでは「Claudeが聞いてくれる」「ユーザーが止められる」ことが安全弁になっていますが、非対話モードではその両方が無くなります。**その分を、フラグと出力スキーマで先に固めておく**のが設計の要点です。

---

# 参考資料

Anthropic公式 Claude Code Docs:

- Run Claude Code programmatically  
  https://code.claude.com/docs/en/headless

- CLI reference  
  https://code.claude.com/docs/en/cli-reference

- Agent SDK overview  
  https://code.claude.com/docs/en/agent-sdk/overview

- GitHub Actions  
  https://code.claude.com/docs/en/github-actions

- GitLab CI/CD  
  https://code.claude.com/docs/en/gitlab-ci-cd

特に確認するとよい項目:

- Start faster with bare mode
- Get structured output
- Stream responses
- Read session metadata
- Fail CI when a plugin or MCP server doesn't load
- Auto-approve tools
- Continue conversations
- Background tasks at exit
- Stop a run with SIGTERM
- What runs before you trust a folder
