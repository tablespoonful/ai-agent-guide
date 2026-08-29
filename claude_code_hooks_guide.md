# Claude Code：Hooksの構成（中身）についての解説

> 対象: Claude Code の Hooks  
> 更新基準: 2026-08-28 時点のAnthropic公式ドキュメント  
> 目的: `settings.json` などに書くHook定義を「何を・どのイベントで・どう書けばよいか」を、設計意図も含めて理解する

---

<!-- toc -->
# 目次

- [1. Hooksとは何か](#1-hooksとは何か)
- [2. なぜHooksが必要なのか](#2-なぜhooksが必要なのか)
- [3. Hookの3階層構造](#3-hookの3階層構造)
- [4. Hookを定義できる場所](#4-hookを定義できる場所)
- [5. 最小例](#5-最小例)
- [6. Hookの発火頻度](#6-hookの発火頻度)
- [7. イベント一覧](#7-イベント一覧)
- [8. PreToolUse](#8-pretooluse)
- [9. PostToolUse / PostToolUseFailure](#9-posttooluse--posttoolusefailure)
- [10. UserPromptSubmit](#10-userpromptsubmit)
- [11. SessionStart / SessionEnd](#11-sessionstart--sessionend)
- [12. Stop / SubagentStop](#12-stop--subagentstop)
- [13. PreCompact / PostCompact](#13-precompact--postcompact)
- [14. InstructionsLoaded](#14-instructionsloaded)
- [15. PermissionRequest / PermissionDenied](#15-permissionrequest--permissiondenied)
- [16. Matcherの評価ルール](#16-matcherの評価ルール)
- [17. イベントごとのmatcher対象](#17-イベントごとのmatcher対象)
- [18. MCP toolのmatcher](#18-mcp-toolのmatcher)
- [19. Handler種別：command](#19-handler種別command)
- [20. exec形式とshell形式](#20-exec形式とshell形式)
- [21. Handler種別：http](#21-handler種別http)
- [22. Handler種別：mcp_tool](#22-handler種別mcp_tool)
- [23. Handler種別：prompt](#23-handler種別prompt)
- [24. Handler種別：agent](#24-handler種別agent)
- [25. 共通フィールド](#25-共通フィールド)
- [26. if による絞り込み](#26-if-による絞り込み)
- [27. Hookへの入力](#27-hookへの入力)
- [28. 終了コードの意味](#28-終了コードの意味)
- [29. exit 2 でブロックできるイベント](#29-exit-2-でブロックできるイベント)
- [30. JSON出力形式](#30-json出力形式)
- [31. 例：破壊的コマンドの遮断](#31-例破壊的コマンドの遮断)
- [32. 例：Windows PowerShell版](#32-例windows-powershell版)
- [33. スクリプトのパス指定](#33-スクリプトのパス指定)
- [34. 非同期実行](#34-非同期実行)
- [35. SkillやSubagentのfrontmatterに書くHook](#35-skillやsubagentのfrontmatterに書くhook)
- [36. HTTP Hookの制限設定](#36-http-hookの制限設定)
- [37. Hookの一括無効化](#37-hookの一括無効化)
- [38. /hooks メニュー](#38-hooks-メニュー)
- [39. デバッグ](#39-デバッグ)
- [40. 設計原則：決定論とLLM判断の分離](#40-設計原則決定論とllm判断の分離)
- [41. パターン：編集後の自動整形](#41-パターン編集後の自動整形)
- [42. パターン：秘密ファイルの保護](#42-パターン秘密ファイルの保護)
- [43. パターン：セッション開始時のコンテキスト注入](#43-パターンセッション開始時のコンテキスト注入)
- [44. パターン：完了前の検証強制](#44-パターン完了前の検証強制)
- [45. Hookを書くときの注意](#45-hookを書くときの注意)
- [46. セキュリティ上の注意](#46-セキュリティ上の注意)
- [47. permissionsとの関係](#47-permissionsとの関係)
- [48. CLAUDE.md / Skill / Hook の使い分け](#48-claudemd--skill--hook-の使い分け)
- [49. 最終チェックリスト](#49-最終チェックリスト)
- [50. まとめ](#50-まとめ)
- [参考資料](#参考資料)

<!-- /toc -->

---

# 1. Hooksとは何か

Hooksは、Claude Codeのライフサイクル上の特定のタイミングで、

```text
shell command
HTTP endpoint
MCP tool呼び出し
LLM prompt
Subagent
```

のいずれかを**自動実行する仕組み**です。

最も重要な理解は、

```text
CLAUDE.md / SKILL.md
=
Claudeへの「お願い」

Hooks
=
Claudeの判断とは無関係に走る「決定論的な処理」
```

ということです。

Hooksは、ターミナル、IDE拡張、デスクトップアプリ、Web環境のいずれでも一貫して発火します。

---

# 2. なぜHooksが必要なのか

`CLAUDE.md` に、

```markdown
変更後は必ず lint を実行すること
```

と書いても、Claudeが実行しないことはあり得ます。CLAUDE.mdはcontextに入る行動指示であって、強制ではないためです。

一方、

```text
PostToolUse Hook
    ↓
Edit / Write のあとに必ず lint を実行
```

とすれば、Claudeの判断に関係なく実行されます。

整理すると、

```text
やってほしい
    ↓
CLAUDE.md

必ずやらせる
    ↓
Hook

やらせない
    ↓
permissions.deny / PreToolUse Hook
```

です。

---

# 3. Hookの3階層構造

Hook定義は必ず3階層になります。

```text
① Hook event
      ↓
   いつ発火するか（PreToolUse など）

② Matcher group
      ↓
   何に対して発火するか（Bash だけ など）

③ Hook handlers
      ↓
   何を実行するか（command / http / mcp_tool / prompt / agent）
```

JSONにすると次の形です。

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "./scripts/check.sh"
          }
        ]
      }
    ]
  }
}
```

`"hooks"` という語が2回出てきますが、外側がイベント辞書、内側がハンドラ配列です。

---

# 4. Hookを定義できる場所

| 場所 | スコープ | 共有できるか |
|---|---|---|
| `~/.claude/settings.json` | 全プロジェクト | × |
| `.claude/settings.json` | そのプロジェクト | ○（コミットする） |
| `.claude/settings.local.json` | そのプロジェクト（自分だけ） | × |
| Managed policy settings | 組織全体 | ○ |
| Plugin の `hooks/hooks.json` | Plugin有効時 | ○ |
| Skill / Subagent の frontmatter | セッション / Subagentスコープ | ○ |

チーム共有したいHookは、

```text
.claude/settings.json
```

に置いてコミットします。

---

# 5. 最小例

`.claude/settings.json`:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "npm run lint:fix"
          }
        ]
      }
    ]
  }
}
```

これだけで、

```text
Claudeがファイルを編集
        ↓
lint:fix が自動実行
```

になります。

---

# 6. Hookの発火頻度

Hookは大きく3つのcadenceに分かれます。

```text
セッションに1回
    ↓
SessionStart / SessionEnd

ターンに1回
    ↓
UserPromptSubmit / Stop / StopFailure

Tool呼び出しごと
    ↓
PreToolUse / PostToolUse
```

この違いは重要で、

```text
毎ターン走らせたい処理
    ↓
UserPromptSubmit / Stop

Tool単位で止めたい処理
    ↓
PreToolUse
```

と使い分けます。

---

# 7. イベント一覧

主なイベントは以下です。

| イベント | 発火タイミング |
|---|---|
| `SessionStart` | セッション開始・再開時 |
| `Setup` | `-p` モードで `--init-only` / `--init` / `--maintenance` 指定時 |
| `UserPromptSubmit` | ユーザーがプロンプトを送信した直後、処理前 |
| `UserPromptExpansion` | コマンドがプロンプトへ展開されるとき |
| `PreToolUse` | Tool実行前（ブロック可能） |
| `PermissionRequest` | Toolがpermission判断を必要としたとき |
| `PermissionDenied` | autoモードがTool呼び出しを拒否したとき |
| `PostToolUse` | Tool成功後 |
| `PostToolUseFailure` | Tool失敗後 |
| `PostToolBatch` | 並列Tool呼び出しのバッチ完了後 |
| `Notification` | 通知送信時 |
| `MessageDisplay` | assistantメッセージ表示中 |
| `SubagentStart` | Subagent起動時 |
| `SubagentStop` | Subagent終了時 |
| `TaskCreated` | `TaskCreate` でタスク作成時 |
| `TaskCompleted` | タスク完了時 |
| `Stop` | Claudeが応答を終えたとき |
| `StopFailure` | APIエラーでターンが終了したとき |
| `InstructionsLoaded` | CLAUDE.md / rules がロードされたとき |
| `ConfigChange` | 設定ファイル変更時 |
| `CwdChanged` | 作業ディレクトリ変更時 |
| `DirectoryAdded` | セッション中にディレクトリ追加時 |
| `FileChanged` | 監視ファイル変更時 |
| `WorktreeCreate` / `WorktreeRemove` | worktree作成・削除時 |
| `PreCompact` / `PostCompact` | context compaction前後 |
| `Elicitation` / `ElicitationResult` | MCPがユーザー入力を要求したとき／応答後 |
| `SessionEnd` | セッション終了時 |

実運用でよく使うのは、

```text
PreToolUse
PostToolUse
UserPromptSubmit
SessionStart
Stop
```

の5つです。

---

# 8. `PreToolUse`

最も強力なイベントです。

```text
Claudeが Tool を呼ぼうとする
            ↓
      PreToolUse Hook
            ↓
    ┌───────┴───────┐
    │               │
  ブロック        通過
    │               │
    ▼               ▼
 実行されない   permission判定へ
```

用途:

```text
危険コマンドの遮断
保護ファイルへの編集拒否
コマンド内容の書き換え
追加コンテキストの注入
```

`PreToolUse` は exit code 2 でTool呼び出しを**ブロックできます**。

---

# 9. `PostToolUse` / `PostToolUseFailure`

Tool実行後に走ります。

```text
PostToolUse
    ↓
Tool成功後

PostToolUseFailure
    ↓
Tool失敗後
```

用途:

```text
編集後の自動format / lint
編集後のtypecheck
変更ファイルの記録
失敗時の追加情報提示
```

どちらもブロックはできません。ただしstderrはClaudeへ見せられるので、

```text
lint失敗
    ↓
stderrに内容を出す
    ↓
Claudeが読んで直す
```

という誘導は可能です。

---

# 10. `UserPromptSubmit`

ユーザーの入力がClaudeへ渡る前に走ります。

```text
ユーザー入力
      ↓
UserPromptSubmit Hook
      ↓
  ┌───┴───┐
  │       │
ブロック  通過（+ context追加）
```

このイベントは、

```text
exit code 0 のとき
    ↓
plain textのstdoutがcontextへ追加される
```

という特徴があります。`SessionStart` と `UserPromptExpansion` も同様です。

用途:

```text
現在のブランチ名やチケット番号を毎回注入する
禁止ワードを含むプロンプトを弾く
時刻・環境情報を添える
```

---

# 11. `SessionStart` / `SessionEnd`

```text
SessionStart
    ↓
セッション開始時に一度だけ

SessionEnd
    ↓
セッション終了時に一度だけ
```

`SessionStart` のmatcherは、開始モードで絞れます。

```text
startup
resume
clear
compact
fork
```

用途:

```text
起動時に依存関係を確認する
起動時にプロジェクト状態を注入する
終了時にログを保存する
```

`SessionEnd` のmatcherは終了理由（`clear` / `resume` / `logout` / `prompt_input_exit` / `other`）で絞れます。

---

# 12. `Stop` / `SubagentStop`

```text
Claudeが「終わりました」と応答を終える
                ↓
            Stop Hook
                ↓
        ┌───────┴───────┐
        │               │
   exit 2 で継続       終了
```

`Stop` は exit code 2 で**停止を阻止できます**。

用途:

```text
testが通っていないのに完了させない
未commitの変更が残っていたら続行させる
出力形式を満たしていなければやり直させる
```

Agentとして使うなら非常に有効ですが、条件を誤ると無限ループになるため、

```text
必ず「もう十分」と判断できる終了条件
```

を持たせます。

---

# 13. `PreCompact` / `PostCompact`

context compactionの前後です。

```text
PreCompact
    ↓
compaction前（exit 2 でブロック可能）

PostCompact
    ↓
compaction後
```

用途:

```text
compaction前に重要情報を外部保存する
compaction後に必要な情報を再注入する
```

---

# 14. `InstructionsLoaded`

CLAUDE.mdやrulesがロードされたときに発火します。

matcherはロード理由で絞れます。

```text
session_start
nested_traversal
path_glob_match
include
compact
```

用途としては、

```text
どの指示ファイルが、いつ、なぜロードされたか
```

をログに出す用途が中心です。path-specific rulesやlazy loadのデバッグに有効です。

---

# 15. `PermissionRequest` / `PermissionDenied`

```text
PermissionRequest
    ↓
Toolがpermission判断を必要としたとき

PermissionDenied
    ↓
autoモードが拒否したとき
```

どちらも exit 2 では制御できません。

`PermissionRequest` は `decision` オブジェクトで、`PermissionDenied` は `retry: true` で制御します。

---

# 16. Matcherの評価ルール

`matcher` の値は、書き方によって解釈が変わります。

| matcherの値 | 解釈 | 例 |
|---|---|---|
| `"*"` / `""` / 省略 | すべてにマッチ | 毎回発火 |
| 英数字・`_`・`-`・空白・`,`・`\|` のみ | 完全一致またはリスト | `Bash`、`Edit\|Write`、`code-reviewer` |
| それ以外の文字を含む | JavaScript正規表現（アンカーなし） | `^Notebook`、`mcp__memory__.*` |

つまり、

```text
Bash          → 完全一致
Edit|Write    → リスト
mcp__.*       → 正規表現
```

と自動判定されます。

意図せず正規表現扱いになると誤マッチするので、記号を含めるときは正規表現として正しいか確認します。

---

# 17. イベントごとのmatcher対象

matcherが「何を見るか」はイベントによって違います。

| イベント | matcherの対象 | 例 |
|---|---|---|
| `PreToolUse` / `PostToolUse` / `PostToolUseFailure` / `PermissionRequest` / `PermissionDenied` | Tool名 | `Bash`、`Edit\|Write`、`mcp__.*` |
| `SessionStart` | 開始モード | `startup`、`resume`、`clear`、`compact`、`fork` |
| `SessionEnd` | 終了理由 | `clear`、`logout` 等 |
| `Notification` | 通知種別 | `permission_prompt`、`idle_prompt` 等 |
| `SubagentStart` / `SubagentStop` | Agent種別 | `Explore`、`Plan`、`general-purpose`、カスタム名 |
| `PreCompact` / `PostCompact` | 発動契機 | `manual`、`auto` |
| `ConfigChange` | 設定ソース | `user_settings`、`project_settings` 等 |
| `FileChanged` | ファイル名リテラル | `.envrc\|.env` |
| `StopFailure` | エラー種別 | `rate_limit`、`overloaded` 等 |
| `InstructionsLoaded` | ロード理由 | `session_start`、`path_glob_match` 等 |

「Tool名で絞る」のはあくまでTool系イベントだけ、という点に注意します。

---

# 18. MCP toolのmatcher

MCP toolの名前は、

```text
mcp__<server>__<tool>
```

という形式です。

例:

```text
mcp__memory__create_entities
mcp__filesystem__read_file
mcp__github__search_repositories
```

サーバー単位で捕まえるなら、

```json
"matcher": "mcp__memory__.*"
```

書き込み系だけを捕まえるなら、

```json
"matcher": "mcp__.*__write.*"
```

Pluginが同梱するMCPサーバーはスコープ付きの名前になります。

```json
"matcher": "mcp__plugin_my-plugin_db__.*"
```

---

# 19. Handler種別：`command`

最も基本的なハンドラです。

```json
{
  "type": "command",
  "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/check-style.sh",
  "args": [],
  "timeout": 30,
  "async": false,
  "shell": "bash",
  "if": "Bash(git *)",
  "statusMessage": "Checking style..."
}
```

主なフィールド:

```text
command  … 必須。実行するコマンド
args     … 省略可。渡すとexec形式になる
async    … バックグラウンド実行
shell    … "bash" または "powershell"
```

---

# 20. exec形式とshell形式

`args` の有無で挙動が変わります。

```text
args あり（exec形式）
    ↓
直接プロセス起動
シェル解釈なし
パイプ・&&・globは効かない

args なし（shell形式）
    ↓
シェル経由
パイプ・&&・globが使える
```

セキュリティ上は、

```text
外部由来の値を含む場合
    ↓
exec形式
```

が安全です。shell形式は文字列連結によるインジェクションの余地があります。

---

# 21. Handler種別：`http`

HTTPエンドポイントへPOSTします。

```json
{
  "type": "http",
  "url": "http://localhost:8080/hooks/pre-tool-use",
  "headers": {
    "Authorization": "Bearer $MY_TOKEN"
  },
  "allowedEnvVars": ["MY_TOKEN"],
  "timeout": 30
}
```

レスポンスは command hook と同じJSON形式で返します。

```text
2xx  → 成功。JSONとして解釈
それ以外 → エラー（非ブロッキング扱い）
```

社内の集中管理サーバーでポリシーを一元化したい場合に使います。

---

# 22. Handler種別：`mcp_tool`

MCPサーバーのtoolを直接呼びます。

```json
{
  "type": "mcp_tool",
  "server": "my_server",
  "tool": "security_scan",
  "input": { "file_path": "${tool_input.file_path}" },
  "timeout": 30
}
```

`input` の中では `${path}` 記法でHook入力の値を参照できます。

---

# 23. Handler種別：`prompt`

LLMに判定させるハンドラです。

```json
{
  "type": "prompt",
  "prompt": "Is this command safe? Analyze: $ARGUMENTS",
  "timeout": 30
}
```

`$ARGUMENTS` にHook入力のJSONが入ります。

```text
決定論的に書けないが
機械的にチェックしたい
```

という中間的な用途向けです。ただしLLM判定である以上、確実な遮断層としては扱わないほうが安全です。

---

# 24. Handler種別：`agent`

Subagentを起動して判定させます。

```json
{
  "type": "agent",
  "prompt": "Review this command and return a decision: $ARGUMENTS",
  "timeout": 60
}
```

`prompt` hookと違い、Read / Grep / Glob などのToolを使って調査できます。

重い代わりに判断材料を集められるので、

```text
コード全体を見ないと判定できないルール
```

に向いています。

---

# 25. 共通フィールド

全ハンドラ共通で使えるフィールドです。

| フィールド | 必須 | 説明 |
|---|---|---|
| `type` | ○ | `command` / `http` / `mcp_tool` / `prompt` / `agent` |
| `if` | × | permission rule形式の絞り込み（Tool系イベントのみ） |
| `timeout` | × | 秒。既定は command / http / mcp_tool が600、prompt が30、agent が60 |
| `statusMessage` | × | スピナーに出す文言 |
| `once` | × | 初回成功後に解除（Skill frontmatterのみ） |

---

# 26. `if` による絞り込み

`matcher` はTool名までしか絞れませんが、`if` はpermission rule構文で中身まで絞れます。

```json
{
  "type": "command",
  "if": "Bash(rm *)",
  "command": "./scripts/block-rm.sh"
}
```

書ける形:

```text
Bash(git *)
Edit(*.ts)
Bash(rm *)
mcp__memory__.*
```

つまり、

```text
matcher
    ↓
どのToolか

if
    ↓
そのToolのどの呼び出しか
```

という二段構えになります。

Bashパターンの評価では、

```text
先頭の VAR=value 代入は除去される
サブコマンドは個別に判定される
$() やバッククォート内も判定される
$VAR の展開先は判定できないためHookは走る
```

という挙動になります。

---

# 27. Hookへの入力

Hookはstdin（HTTPならbody）にJSONを受け取ります。全イベント共通で入るのは次の項目です。

```json
{
  "session_id": "abc123",
  "prompt_id": "550e8400-e29b-41d4-a716-446655440000",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/current/working/dir",
  "permission_mode": "default",
  "hook_event_name": "PreToolUse",
  "agent_id": "subagent-id（Subagent内のとき）",
  "agent_type": "agent-name（Subagent内のとき）",
  "effort": { "level": "high" }
}
```

これに加えて、イベント固有のフィールド（`tool_name`、`tool_input`、`tool_response` など）が入ります。

shellスクリプトなら `jq` で取り出すのが定番です。

```bash
COMMAND=$(jq -r '.tool_input.command')
```

---

# 28. 終了コードの意味

```text
exit 0
    ↓
成功

exit 2
    ↓
ブロッキングエラー（対応イベントのみ）

その他
    ↓
非ブロッキングエラー
```

`exit 0` のときのstdoutの扱いは次の通りです。

```text
stdoutが { で始まり } で終わる
    ↓
JSONとして解釈

UserPromptSubmit / UserPromptExpansion / SessionStart
    ↓
plain textのstdoutをcontextへ追加

それ以外のイベント
    ↓
stdoutはdebug logのみ
```

---

# 29. exit 2 でブロックできるイベント

ブロックできるのは以下です。

| イベント | exit 2 の効果 |
|---|---|
| `PreToolUse` | Tool呼び出しをブロック |
| `UserPromptSubmit` | プロンプトをブロックし消去 |
| `UserPromptExpansion` | 展開をブロック |
| `Stop` / `SubagentStop` / `TeammateIdle` | 停止・idle化を阻止 |
| `TaskCreated` / `TaskCompleted` | 作成をロールバック／完了を阻止 |
| `ConfigChange` | 設定変更をブロック |
| `PostToolBatch` | agentic loopを停止 |
| `PreCompact` | compactionをブロック |
| `Elicitation` / `ElicitationResult` | elicitationを拒否／応答をブロック |
| `WorktreeCreate` | 作成を失敗させる |

逆に、以下は exit 2 でも効果がありません。

```text
PostToolUse / PostToolUseFailure
PermissionRequest / PermissionDenied
StopFailure / Notification
SubagentStart / SessionStart / Setup / SessionEnd
CwdChanged / DirectoryAdded / FileChanged
PostCompact / MessageDisplay / InstructionsLoaded
```

`PostToolUse` で「ブロックしたい」という設計は成立しないので、
遮断したいなら `PreToolUse` に置き換えます。

---

# 30. JSON出力形式

command / http hookはJSONで細かい制御を返せます。

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "理由",
    "additionalContext": "Claudeに見せたいテキスト",
    "updatedInput": { "modified": "input" },
    "retry": true,
    "systemMessage": "debug log向けメッセージ"
  },
  "terminalSequence": ""
}
```

主なフィールドの役割:

```text
permissionDecision
    ↓
allow / deny / ask

additionalContext
    ↓
Claudeへ追加情報を渡す

updatedInput
    ↓
Tool入力を書き換える

terminalSequence
    ↓
ベル・ウィンドウタイトル等の端末制御
```

---

# 31. 例：破壊的コマンドの遮断

`.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "if": "Bash(rm *)",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/block-rm.sh",
            "args": []
          }
        ]
      }
    ]
  }
}
```

`.claude/hooks/block-rm.sh`:

```bash
#!/bin/bash
COMMAND=$(jq -r '.tool_input.command')

if echo "$COMMAND" | grep -q 'rm -rf'; then
  jq -n '{
    hookSpecificOutput: {
      hookEventName: "PreToolUse",
      permissionDecision: "deny",
      permissionDecisionReason: "Destructive command blocked by hook"
    }
  }'
else
  exit 0
fi
```

`exit 0` で何も出力しなければ「判断しない」となり、通常のpermissionフローへ進みます。

---

# 32. 例：Windows PowerShell版

同じことをPowerShellで書く場合です。

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash|PowerShell",
        "hooks": [
          {
            "type": "command",
            "if": "PowerShell(Remove-Item *)",
            "command": "powershell.exe",
            "args": [
              "-NoProfile",
              "-ExecutionPolicy",
              "Bypass",
              "-File",
              "${CLAUDE_PROJECT_DIR}/.claude/hooks/block-rm.ps1"
            ]
          }
        ]
      }
    ]
  }
}
```

```powershell
$callInput = [Console]::In.ReadToEnd() | ConvertFrom-Json
$command = $callInput.tool_input.command

if ($command -match 'rm -rf|Remove-Item.*-Recurse') {
  @{
    hookSpecificOutput = @{
      hookEventName = "PreToolUse"
      permissionDecision = "deny"
      permissionDecisionReason = "Destructive command blocked by hook"
    }
  } | ConvertTo-Json
} else {
  exit 0
}
```

Windows環境では `shell: "powershell"` を指定するか、上のように `powershell.exe` を明示的に呼びます。

---

# 33. スクリプトのパス指定

Hookの `command` では、作業ディレクトリが変わっても壊れないよう変数を使います。

| 変数 | 指す場所 |
|---|---|
| `${CLAUDE_PROJECT_DIR}` | セッションを開始したプロジェクトルート |
| `${CLAUDE_PLUGIN_ROOT}` | Pluginのインストールディレクトリ |
| `${CLAUDE_PLUGIN_DATA}` | Pluginの永続データディレクトリ |

注意点として、

```text
worktreeを使っても
${CLAUDE_PROJECT_DIR} は元の場所を指し続ける
```

ため、現在のディレクトリが必要ならHook入力の `cwd` を読みます。

---

# 34. 非同期実行

長い処理でClaudeを待たせたくない場合に使います。

```json
{
  "type": "command",
  "command": "/path/to/long-check.sh",
  "async": true
}
```

挙動:

```text
async: true
    ↓
バックグラウンド実行
timeoutは適用されない
Claudeは待たずに続行

asyncRewake: true
    ↓
バックグラウンド実行
exit code 2 でClaudeを起こす
stdout / stderr が system reminder として渡る
```

`asyncRewake` は、

```text
重いテストを裏で回す
        ↓
失敗したときだけClaudeに知らせる
```

という設計に向いています。

---

# 35. SkillやSubagentのfrontmatterに書くHook

`settings.json` 以外に、Skill / Subagentのfrontmatterでも定義できます。

```yaml
---
name: secure-operations
description: セキュリティチェック付きで操作を行う
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/security-check.sh"
---
```

スコープが異なります。

```text
Skillのhooks
    ↓
Skill発火時に登録され
セッションの残り全体で有効

Subagentのhooks
    ↓
そのSubagentの実行中だけ有効
終了時に解除
```

Skill側のHookは「一度登録すると残る」ため、一回だけ効かせたいときは `once: true` を付けます。

---

# 36. HTTP Hookの制限設定

HTTP hookは外部通信を伴うため、送信先を絞れます。

```json
{
  "allowedHttpHookUrls": [
    "http://localhost:*",
    "https://api.example.com/*"
  ]
}
```

ヘッダに使える環境変数も絞れます。

```json
{
  "httpHookAllowedEnvVars": ["MY_TOKEN", "API_KEY"]
}
```

---

# 37. Hookの一括無効化

```json
{
  "disableAllHooks": true
}
```

一時的に切りたいときは起動時に渡せます。

```bash
claude --settings '{"disableAllHooks": true}'
```

組織側で「管理者が配ったHookだけ許可する」なら、managed settingsで、

```json
{
  "allowManagedHooksOnly": true
}
```

を設定します。これによりuser / project / local のHookはブロックされます。

---

# 38. `/hooks` メニュー

セッション内で、

```text
/hooks
```

を実行すると、設定済みHookを一覧できます。

```text
イベントごとの件数
matcherとハンドラの内容
ソースラベル
  User Settings
  Project Settings
  Local Settings
  Plugin Hooks
  Session Hooks
```

読み取り専用なので、変更はJSONを直接編集します。

---

# 39. デバッグ

Hookが動かないときは、

```bash
claude --debug
```

で起動します。

`~/.claude/logs/` に、

```text
Hookコマンドの出力
stderrの内容
パースエラー
timeout警告
```

が記録されます。

確認の順序としては、

```text
① /hooks で登録されているか
        ↓
② --debug でマッチしたか
        ↓
③ 手動でスクリプトを実行して動くか
        ↓
④ 終了コードと出力形式が正しいか
```

が効率的です。

---

# 40. 設計原則：決定論とLLM判断の分離

Hookを設計するときの中心原則です。

LLMに向いている処理:

```text
コード理解
設計判断
分類
原因分析
修正案の提示
```

Hookに向いている処理:

```text
lint / format
schema validation
secret scanning
出力形式の機械検証
危険操作のblock
特定コマンドの強制実行
```

理想の構成:

```text
CLAUDE.md
    ↓
方針を伝える

Claude
    ↓
判断・実装

Hook
    ↓
機械的に検証・強制

Claude
    ↓
結果を分析して修正
```

---

# 41. パターン：編集後の自動整形

```json
{
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

「整形はClaudeに任せない」という分離です。

---

# 42. パターン：秘密ファイルの保護

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write|Read",
        "hooks": [
          {
            "type": "command",
            "if": "Read(./.env)",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/deny.sh",
            "args": []
          }
        ]
      }
    ]
  }
}
```

ただし、単に読ませたくないだけなら `permissions.deny` のほうが簡単です。

```json
{
  "permissions": {
    "deny": ["Read(./.env)", "Read(./secrets/**)"]
  }
}
```

Hookを使うのは、

```text
条件が動的
理由をログに残したい
判定を外部システムに委ねたい
```

という場合です。

---

# 43. パターン：セッション開始時のコンテキスト注入

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup",
        "hooks": [
          {
            "type": "command",
            "command": "git branch --show-current && git status --short"
          }
        ]
      }
    ]
  }
}
```

`SessionStart` は stdout がそのままcontextへ入るため、

```text
現在のブランチ
未commitの変更
稼働環境
```

などを毎回渡せます。

CLAUDE.mdに書くと古くなる情報を、実行時に取得できるのが利点です。

---

# 44. パターン：完了前の検証強制

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/verify.sh",
            "args": []
          }
        ]
      }
    ]
  }
}
```

`verify.sh` がtest失敗時に exit 2 を返せば、Claudeは停止できません。

```text
実装
  ↓
Stop Hook
  ↓
test失敗 → exit 2
  ↓
Claudeが続行して修正
  ↓
再度Stop Hook
  ↓
通過 → 完了
```

強力ですが、

```text
通らない条件を書く
    ↓
永久に終わらない
```

というリスクがあるので、リトライ上限や明示的なエスケープを用意します。

---

# 45. Hookを書くときの注意

実運用上、次を意識すると壊れにくくなります。

```text
冪等にする
    ↓
何度走っても壊れない

速くする
    ↓
PreToolUse / PostToolUse は毎回走る

失敗を明確にする
    ↓
終了コードと出力形式を守る

出力を絞る
    ↓
contextへ入るイベントでは特に重要

依存を減らす
    ↓
jq等が無い環境でも壊れないようにする
```

特に `PostToolUse` は編集のたびに走るため、数百ms単位の差が体感に効きます。

---

# 46. セキュリティ上の注意

Hookは**あなたの権限でそのまま実行されます**。

```text
Hook設定をrepositoryから取得
        ↓
中身を確認しない
        ↓
任意コマンドが実行される
```

というリスクがあります。

したがって、

```text
他人のrepositoryの .claude/settings.json は必ず読む
外部由来の値をshell形式で連結しない
HTTP hookの送信先をallowlistで絞る
機密値はheadersへ直書きせず環境変数経由にする
```

を守ります。

なお、非対話モード（`claude -p`）では信頼していないフォルダでもproject settingsのHookが走ります。CIでは `--bare` を使って読み込ませないのが安全です。

---

# 47. permissionsとの関係

両方とも「強制層」ですが役割が違います。

```text
permissions
    ↓
静的なルールで許可・拒否

Hooks
    ↓
実行時に判断して許可・拒否・書き換え
```

評価順は、

```text
PreToolUse Hook
      ↓
permission rules（deny → ask → allow）
```

ですが、重要な原則があります。

```text
Hookが "allow" を返しても
deny / ask ルールは無効化されない

Hookが exit 2 を返すと
allowルールより先にブロックされる
```

つまり、

```text
Hookは「さらに厳しくする」方向には確実に効く
「緩める」方向には permission ruleが優先される
```

と理解しておくと安全です。

---

# 48. CLAUDE.md / Skill / Hook の使い分け

```text
CLAUDE.md
    ↓
常に持っていてほしい方針

.claude/rules/
    ↓
特定pathで効かせたい方針

SKILL.md
    ↓
特定タスクの手順

Hooks
    ↓
判断に依存させたくない処理
```

例:

```text
「変更後はtestを実行する」
    ↓
CLAUDE.md（誘導）
    +
Stop Hook（強制）

「productionへdeployしない」
    ↓
permissions.deny（遮断）
    +
CLAUDE.md（説明）
```

「誘導」と「強制」は排他ではなく、重ねて使うのが実務的です。

---

# 49. 最終チェックリスト

新しいHookを追加したら確認します。

```text
[ ] イベントの選択が正しい（PostToolUseでブロックしようとしていない）
[ ] matcherが意図通りに解釈される（正規表現化していないか）
[ ] ifで対象を十分に絞っている
[ ] 終了コードの意味を正しく使っている
[ ] JSON出力のスキーマが正しい
[ ] スクリプトを単体実行して動作確認した
[ ] --debug でマッチとexit codeを確認した
[ ] 冪等である
[ ] 実行時間が許容範囲
[ ] 無限ループの危険がない（特にStop）
[ ] 外部入力をshell形式で連結していない
[ ] 機密値をログや出力に出していない
[ ] チーム共有すべきものが settings.json に入っている
[ ] 個人用のものが settings.local.json に入っている
```

---

# 50. まとめ

Hooksの構成を最も簡潔にまとめると、

```text
Hooks
│
├─ Event
│    ↓
│  いつ発火するか
│
├─ Matcher
│    ↓
│  何に対して発火するか
│
└─ Handler
     │
     ├─ command    … shellコマンド
     ├─ http       … HTTPエンドポイント
     ├─ mcp_tool   … MCP tool呼び出し
     ├─ prompt     … LLM判定
     └─ agent      … Subagent判定
          ↓
        何を実行するか
```

そして制御は、

```text
入力
    ↓
stdinのJSON

出力
    ↓
exit code（0 / 2 / その他）
    +
stdoutのJSON（permissionDecision等）
```

で行います。

Claude Code全体の中での位置づけは、

```text
CLAUDE.md / rules / SKILL.md
=
Claudeに判断させる層

permissions / sandbox
=
静的に許可・拒否する層

Hooks
=
実行時に決定論的な処理を差し込む層
```

です。

この3層を分けて設計すると、

```text
指示は短く保てる
安全性はClaudeの判断に依存しない
検証は自動化される
```

という状態を作れます。

---

# 参考資料

Anthropic公式 Claude Code Docs:

- Hooks reference  
  https://code.claude.com/docs/en/hooks

- Get started with hooks  
  https://code.claude.com/docs/en/hooks-guide

- Configure permissions  
  https://code.claude.com/docs/en/permissions

特に確認するとよい項目:

- Hook events
- Matcher patterns
- Hook handler types
- Common input fields
- Exit code behavior
- JSON output format
- Hooks in skills and agents
- Run hooks in the background
- Debug hooks
