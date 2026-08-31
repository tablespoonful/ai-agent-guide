# Claude Code コマンド集

> 対象: Claude Code CLI の全コマンド・オプション・ショートカット
> 検証環境: v2.1.229 (Windows 11)
> 出典: 実機の `claude --help` 出力と公式ドキュメント (code.claude.com/docs)
> 更新基準: 2026-08-31 時点

記載内容はすべて実機の出力と公式ドキュメントで確認している。記憶や推測で書いた項目はない。

コマンドの一部は環境によって表示されない。プラットフォーム・契約プラン・実行環境で可用性が変わる
（例: `/desktop` は macOS と x64 Windows でサブスク契約時のみ、`/upgrade` は Enterprise プランでは表示されない）。

---

<!-- toc -->
# 目次

- [1. スラッシュコマンド](#1-スラッシュコマンド)
- [2. キーボードショートカット](#2-キーボードショートカット)
- [3. 入力欄の特殊記号](#3-入力欄の特殊記号)
- [4. CLI サブコマンド](#4-cli-サブコマンド)
- [5. CLI オプション](#5-cli-オプション)
- [6. 自分のコマンドを追加する](#6-自分のコマンドを追加する)

<!-- /toc -->

---

# 1. スラッシュコマンド

セッション中に `/` から始めて入力する。全 111 個。
`<arg>` は必須引数、`[arg]` は省略可能な引数。

- **[S]** … バンドル済みスキル。Claude に渡されるプロンプトとして動く
- **[W]** … バンドル済み動的ワークフロー。多数のサブエージェントに仕事を振り、バックグラウンドで走る


## セッションを操る

| コマンド | 説明 |
|---|---|
| `/autocompact [auto&#124;<tokens>]` | 自動コンパクトの発動閾値を設定する（auto、または 100k〜1M トークン） |
| `/background [prompt]` | 現在のセッションをバックグラウンドエージェントに切り替え、ターミナルを解放する |
| `/branch [name]` | この時点で会話を分岐させる。今の流れを失わずに別方針を試せる |
| `/btw [question]` | 会話履歴に残さずに、今のセッションについて質問する |
| `/clear [name]` | コンテキストを空にして新しい会話を始める。名前を渡すとセッション名になる |
| `/compact [instructions]` | ここまでの会話を要約してコンテキストを空ける。要約の観点を引数で指定できる |
| `/context [all]` | コンテキスト使用量を色分けグリッドで可視化する |
| `/copy [N]` | 直前の応答をクリップボードにコピーする。数字を渡すと N 個前 |
| `/exit` | CLI を終了する。バックグラウンドセッションに接続中なら切り離すだけ |
| `/export [filename]` | 会話をプレーンテキストとして書き出す |
| `/focus` | 直近のやり取りだけを表示するフォーカスビューを切り替える |
| `/fork [prompt]` | 会話をコピーしてバックグラウンドセッションとして走らせる |
| `/recap` | 現在のセッションの要約を 1 行で生成する |
| `/rename [name]` | 現在のセッションに名前を付け、プロンプトバーに表示する |
| `/resume [session]` | ID か名前で会話を再開する。引数なしならピッカーを開く |
| `/rewind` | 会話とコードを過去の時点まで巻き戻す |
| `/stop` | バックグラウンドセッションを停止する（接続中のみ） |
| `/subtask <task>` | 会話を引き継いだバックグラウンドのサブエージェントを起動する |
| `/tasks` | このセッションのバックグラウンド作業を確認・管理する |


## コードを書く・直す

| コマンド | 説明 |
|---|---|
| `/autofix-pr [prompt]` | CI の失敗を監視して自動修正するクラウドセッションを立てる |
| `/batch <instruction>` | **[S]** コードベース全体にまたがる大規模変更を並列で進める |
| `/code-review [low&#124;medium&#124;high&#124;xhigh&#124;max&#124;ultra] [--fix] [--comment] [pr#&#124;branch&#124;path]` | **[S]** 差分・PR・ブランチ・パスをレビューする。--fix で修正まで、--comment で PR にコメント |
| `/debug [description]` | **[S]** デバッグログを有効にし、問題の切り分けを進める |
| `/diff` | 未コミットの変更をインタラクティブな差分ビューアで開く |
| `/goal [condition&#124;clear]` | ゴールを設定し、条件を満たすまで Claude にターンをまたいで作業させる |
| `/plan [description]` | プランモードに入る。説明を引数で渡せる |
| `/review [low&#124;medium&#124;high&#124;xhigh&#124;max&#124;ultra] [--fix] [--comment] [pr#&#124;branch&#124;path]` | `/code-review` のエイリアス |
| `/run` | **[S]** プロジェクトのアプリを起動し、変更が実際に動くところまで確認する |
| `/security-review` | 現在のブランチの変更にセキュリティ脆弱性がないか解析する |
| `/simplify [target]` | **[S]** 変更されたコードを整理の観点でレビューし、修正まで適用する |
| `/ultrareview [PR or branch]` | クラウドサンドボックスでマルチエージェントの深いコードレビューを走らせる |
| `/verify` | **[S]** 変更が意図どおり動くかビルド・実行して確認する |


## モデルと推論量

| コマンド | 説明 |
|---|---|
| `/advisor [model&#124;off]` | アドバイザーツールを切り替える。要所で別モデルに助言を求める |
| `/effort [level&#124;auto&#124;status]` | 推論量を設定する（low〜xhigh、max、ultracode、auto） |
| `/fast [on&#124;off]` | ファストモードを切り替える。応答中に実行しても効く |
| `/model [model]` | モデルを切り替え、新規セッションの既定として保存する |


## 設定と環境

| コマンド | 説明 |
|---|---|
| `/add-dir <path>` | ファイルアクセスを許可する作業ディレクトリを追加する |
| `/auto-mode-setup` | プロジェクトと最近のセッションから `autoMode.environment` の下書きを作る |
| `/cd <path>` | 会話を保ったまま作業ディレクトリを移動する |
| `/color [color&#124;default]` | このセッションのプロンプトバーの色を変える |
| `/config [key=value ...]` | 設定画面を開く（テーマ、モデル、出力スタイルなど） |
| `/doctor` | **[S]** セットアップを点検し、問題があれば修復する |
| `/hooks` | ツールイベントに対するフック設定を確認する |
| `/import [codex&#124;gemini] [--dry-run] [--yes]` | 他のコーディングエージェント（codex / gemini）から設定を取り込む |
| `/init` | `CLAUDE.md` を生成してプロジェクトを初期化する |
| `/keybindings` | キーバインド定義ファイルを開く |
| `/memory` | `CLAUDE.md` を編集し、オートメモリの有効・無効を切り替える |
| `/permissions` | ツール権限の allow / ask / deny ルールを管理する |
| `/sandbox` | サンドボックスモードを切り替える（対応プラットフォームのみ） |
| `/scroll-speed` | マウスホイールのスクロール速度を目盛り付きで調整する |
| `/setup-bedrock` | Amazon Bedrock の認証・リージョン・モデルを設定する |
| `/setup-vertex` | Google Cloud の認証・プロジェクトを設定する |
| `/status` | バージョンやアカウント情報を含むステータスタブを開く |
| `/statusline` | ステータスラインを設定する。欲しい表示を説明すると組んでくれる |
| `/terminal-setup` | VS Code などで Shift+Enter を改行にするキーバインドを入れる |
| `/theme` | カラーテーマを変更する。端末に追従する auto もある |
| `/tui [default&#124;fullscreen]` | ターミナル UI のレンダラを切り替えて再起動する（default / fullscreen） |
| `/voice [hold&#124;tap&#124;off]` | 音声入力を切り替える（hold / tap / off） |


## 拡張機能

| コマンド | 説明 |
|---|---|
| `/agents` | サブエージェントの作り方の案内を表示する（v2.1.198 以降） |
| `/fewer-permission-prompts` | **[S]** 履歴を走査し、権限プロンプトを減らす allowlist を提案する |
| `/list-agents` | メッセージを送れるサブエージェント・チームメイトを一覧する |
| `/loop [interval] [prompt]` | **[S]** プロンプトを一定間隔で繰り返し実行する |
| `/mcp [reconnect <server>&#124;enable&#124;disable [<server>&#124;all]]` | MCP サーバー接続と OAuth 認証を管理する |
| `/plugin [subcommand]` | プラグインを管理する。引数なしでブラウザを開く |
| `/reload-plugins [--force]` | 再起動せずにプラグインを再読み込みする |
| `/reload-skills` | スキル・コマンドのディレクトリを再スキャンする |
| `/run-skill-generator` | **[S]** `/run` と `/verify` にプロジェクト固有の起動方法を覚えさせる |
| `/schedule [description]` | cron スケジュールで動くルーティン（クラウドエージェント）を管理する |
| `/skills` | 利用可能なスキルを一覧する。入力で絞り込める |
| `/workflow-authoring` | **[S]** 動的ワークフローのスクリプト API リファレンスを読み込む |
| `/workflows` | ワークフローの進捗ビューを開き、監視・一時停止・再開する |


## コストと使用量

| コマンド | 説明 |
|---|---|
| `/cost` | `/usage` のエイリアス |
| `/insights` | 最近のセッションを分析した HTML レポートを生成する |
| `/rate-limit-options` | 利用上限に達したときの選択肢を表示する |
| `/stats` | `/usage` のエイリアス。Stats タブで開く |
| `/upgrade` | 上位プランへの切り替えページをブラウザで開く |
| `/usage` | セッションのコスト、プラン上限、利用状況を表示する |
| `/usage-credits` | 使用クレジットを設定する、または管理者に申請する |


## 外部連携

| コマンド | 説明 |
|---|---|
| `/artifacts` | 自分のアーティファクトや共有されたものを一覧し、セッションに添付する |
| `/chrome` | Claude in Chrome の設定を行う |
| `/desktop` | 現在のセッションを Claude Code デスクトップアプリで継続する |
| `/ide` | IDE 連携を管理し、状態を表示する |
| `/install-github-app` | リポジトリに Claude GitHub App を導入する |
| `/install-slack-app` | Claude Slack app を導入する |
| `/mobile` | モバイルアプリのダウンロード用 QR コードを表示する |
| `/remote-control` | claude.ai からのリモート操作を受け付ける状態にする |
| `/remote-env` | クラウドエージェントの既定環境を選ぶ |
| `/teleport` | Claude Code on the web のセッションをこの端末に引き込む |
| `/web-setup` | ローカルの `gh` CLI 認証を使って GitHub アカウントを接続する |


## 調査・設計支援

| コマンド | 説明 |
|---|---|
| `/claude-api [migrate&#124;upgrade&#124;managed-agents-onboard&#124;prompt-audit&#124;cost-optimize]` | **[S]** Claude API と Managed Agents のリファレンスを読み込む |
| `/dataviz [request]` | **[S]** チャート・グラフ・ダッシュボードの設計指針を読み込む |
| `/deep-research <question>` | **[W]** ウェブ検索を並列展開し、読み込んでレポートにまとめる |
| `/design [brief]` | **[S]** UI モックアップ、画面フロー、ランディングページを設計する |
| `/design-login` | `/design-sync` のためにデザインシステムへのアクセスを認可する |
| `/design-sync [hint]` | **[S]** リポジトリの React デザインシステムを変換してアップロードする |
| `/team-onboarding` | 利用状況からチーム向けオンボーディングガイドを生成する |


## ヘルプとその他

| コマンド | 説明 |
|---|---|
| `/bug [report]` | バグを報告する、または会話を共有する。共有範囲は選べる |
| `/feedback [report]` | Claude Code への製品フィードバックを送る |
| `/heapdump` | ヒープスナップショットとメモリ内訳を書き出す（隠しコマンド） |
| `/help` | ヘルプと利用可能なコマンドを表示する |
| `/login` | Anthropic アカウントにサインインする |
| `/logout` | Anthropic アカウントからサインアウトする |
| `/passes` | Claude Code の 1 週間無料パスを友人に共有する |
| `/powerup` | 短いインタラクティブなレッスンで機能を学ぶ |
| `/privacy-settings` | プライバシー設定を確認・更新する |
| `/radio` | Claude FM の lo-fi ラジオをブラウザで開く |
| `/release-notes` | 変更履歴をバージョンピッカーで閲覧する |
| `/stickers` | Claude Code のステッカーを注文する |


## 廃止済み

| コマンド | 説明 |
|---|---|
| `/pr-comments [PR]` | v2.1.91 で削除。PR のコメントは Claude に直接頼む |
| `/ultraplan <prompt>` | 削除済み。プランモードを使う |
| `/vim` | v2.1.92 で削除。Vim / Normal の切り替えは別の方法で行う |
---

# 2. キーボードショートカット

端末やプラットフォームによって効かないものがある。フルスクリーン描画中はトランスクリプトビューアで `?` を押すと、その場で使えるショートカットが出る。

macOS で `Alt+B` `Alt+F` `Alt+D` `Alt+Y` `Alt+P` を使うには、端末側で Option を Meta キーとして扱う設定が必要。

## 全般

| ショートカット | 動作 |
|---|---|
| `Ctrl+C` | 中断する、または入力をクリアする |
| `Ctrl+D` | セッションを終了する |
| `Esc` | Claude を中断する、またはダイアログを閉じる |
| `Esc` `Esc` | 入力の下書きをクリアする、または巻き戻す |
| `Ctrl+O` | トランスクリプトビューアを開閉する |
| `Ctrl+R` | コマンド履歴を逆順検索する |
| `Ctrl+L` | 画面を再描画する |
| `Ctrl+G` または `Ctrl+X` `Ctrl+E` | 既定のテキストエディタで開く |
| `Ctrl+V` / `Cmd+V` (iTerm2) / `Alt+V` (Windows・WSL) | クリップボードの画像を貼る |
| `Ctrl+B` | 実行中のタスクをバックグラウンドに送る |
| `Ctrl+X` `Ctrl+K` | このセッションのバックグラウンドサブエージェントを全部止める |
| `Ctrl+T` | Claude のタスクチェックリストを開閉する |
| `Ctrl+S` | プロンプトを退避する・戻す |
| `Ctrl+Z` | Claude Code をサスペンドする |
| `Tab` | 補完候補を確定する、またはコメントを追加する |
| `Shift+Tab` | 権限モードを切り替える |
| `Option+P` / `Alt+P` | モデルを切り替える |
| `Option+T` / `Alt+T` | 拡張思考を切り替える |
| `Option+O` / `Alt+O` | ファストモードを切り替える |
| `↑` `↓` または `Ctrl+P` `Ctrl+N` | カーソル移動、またはコマンド履歴を辿る |
| `←` `→` | ダイアログのタブを切り替える |

## テキスト編集

| ショートカット | 動作 |
|---|---|
| `Ctrl+A` / `Ctrl+E` | 行頭 / 行末へ移動する |
| `Ctrl+K` | 行末まで削除する（貼り付け用に保持） |
| `Ctrl+U` | カーソルから行頭まで削除する |
| `Ctrl+W` | 直前の単語を削除する |
| `Ctrl+Y` | 削除したテキストを貼り付ける |
| `Alt+Y`（`Ctrl+Y` の直後） | 貼り付け履歴を巡回する |
| `Alt+B` / `Alt+F` | 単語単位で後ろ / 前へ移動する |
| `Alt+D` | 次の単語を削除する |
| `Ctrl+_` または `Ctrl+Shift+-` | 直前の入力編集を取り消す |

`~/.claude/settings.json` に `"keybindingFlavor": "readline"` を入れると、編集キーが GNU readline（Bash と同じ）の流儀になる。既定は `"classic"`。readline では `Ctrl+W` が直前の空白まで削除し、`_` `.` `/` が単語区切りとして扱われる。

## 複数行入力

| 方法 | ショートカット | 条件 |
|---|---|---|
| バックスラッシュ | `\` → `Enter` | どの端末でも使える |
| Control シーケンス | `Ctrl+J` | どの端末でも設定不要 |
| Shift+Enter | `Shift+Enter` | iTerm2、WezTerm、Ghostty、Kitty、Warp、Apple Terminal、Windows Terminal では標準で使える |
| Option キー | `Option+Enter` | macOS で Option を Meta に設定した場合 |

## トランスクリプトビューア（`Ctrl+O` で開く）

| ショートカット | 動作 |
|---|---|
| `?` | ショートカットのヘルプパネルを開閉する（フルスクリーン描画時） |
| `{` / `}` | 前 / 次のユーザープロンプトへ飛ぶ（フルスクリーン描画時） |
| `v` | 会話を一時ファイルに書き出し `$EDITOR` で開く |
| `Ctrl+E` | 全内容の表示を切り替える（classic レンダラのみ） |
| `q` / `Ctrl+C` / `Esc` | ビューアを閉じる |

---

# 3. 入力欄の特殊記号

| 記号 | 位置 | 動作 |
|---|---|---|
| `/` | 行頭 | コマンドまたはスキルを呼ぶ |
| `!` | 行頭 | シェルモード。コマンドを直接実行し、出力をセッションに取り込む |
| `@` | どこでも | ファイルパス補完を出す |
| `:` | どこでも | 絵文字ショートコード。`:name:` を打ち切ると絵文字が入る |
| `?` | 空の入力欄 | ショートカットのヘルプパネルを開閉する |

`!` は「自分でコマンドを打って、その結果を Claude に見せたい」ときに使う。ログイン処理など Claude に任せられない対話的なコマンドもこれで実行できる。

---

# 4. CLI サブコマンド

`claude <subcommand>` の形で、セッションに入らずに実行する。

| サブコマンド | 説明 |
|---|---|
| `claude agents` | バックグラウンドエージェントを管理する |
| `claude auth` | 認証を管理する |
| `claude auto-mode` | auto モードの分類器設定を確認・リセットする |
| `claude doctor` | インストール状態を点検する。信頼プロンプトなしで設定ファイルを読む。修復まで行うなら session 内で `/doctor` |
| `claude gateway` | エンタープライズ向けの認証・テレメトリゲートウェイを起動する |
| `claude import [source]` | 他の AI コーディングエージェントから設定を取り込む |
| `claude install [target]` | ネイティブビルドを導入する（`stable` / `latest` / 特定バージョン） |
| `claude mcp` | MCP サーバーを設定・管理する |
| `claude plugin` | プラグインを管理する |
| `claude project` | プロジェクトの状態を管理する |
| `claude setup-token` | 長期間有効な認証トークンを作る（サブスク契約が必要） |
| `claude ultrareview [target]` | 現在のブランチ（または PR 番号・ベースブランチ）に対してクラウドのマルチエージェントレビューを走らせる |
| `claude update` | 更新を確認して適用する |

---

# 5. CLI オプション

`claude --help` の全項目。用途別に並べ替えている。

## セッションの起動と再開

| オプション | 説明 |
|---|---|
| `-c, --continue` | 同じディレクトリで直近の会話を続ける |
| `-r, --resume [value]` | セッション ID を指定して再開、または検索付きピッカーを開く |
| `--fork-session` | 再開時に元の ID を再利用せず新しい ID を振る |
| `--from-pr [value]` | PR に紐づくセッションを再開する |
| `--session-id <uuid>` | 会話に特定のセッション ID を使う |
| `-n, --name <name>` | セッションに表示名を付ける |
| `--no-session-persistence` | セッションをディスクに保存しない（`--print` 専用） |
| `-w, --worktree [name]` | このセッション用に git worktree を作る |
| `--tmux` | worktree 用に tmux セッションを作る（`--worktree` が必要） |

## 非対話実行（スクリプト・CI 向け）

| オプション | 説明 |
|---|---|
| `-p, --print` | 応答を出力して終了する。パイプ処理向け |
| `--output-format <format>` | `text`（既定） / `json` / `stream-json` |
| `--input-format <format>` | `text`（既定） / `stream-json` |
| `--json-schema <schema>` | 構造化出力を JSON Schema で検証する |
| `--include-partial-messages` | 部分メッセージを逐次出力する |
| `--include-hook-events` | フックのライフサイクルイベントを出力に含める |
| `--replay-user-messages` | stdin のユーザーメッセージを stdout に再出力する |
| `--forward-subagent-text` | サブエージェントのテキストと思考を転送する |
| `--max-budget-usd <amount>` | API 呼び出しの上限金額を設定する |
| `--fallback-model <model>` | 既定モデルが過負荷のときの代替を指定する（カンマ区切りで複数可） |

> **注意**: `--print` を含む非対話モードでは、ワークスペースの信頼ダイアログがスキップされる。信頼できるディレクトリでのみ使うこと。検証に失敗した設定ファイルはエラーを出さず黙って無視される。

## モデルと推論量

| オプション | 説明 |
|---|---|
| `--model <model>` | 使用するモデル。エイリアス（`opus` / `sonnet` / `fable`）か完全名 |
| `--effort <level>` | 推論量（`low` / `medium` / `high` / `xhigh` / `max`） |
| `--betas <betas...>` | API リクエストに含めるベータヘッダ（API キー利用者のみ） |
| `--autocompact <auto\|tokens>` | 自動コンパクトの閾値（auto、または 100k〜1M） |

## 権限とツール

| オプション | 説明 |
|---|---|
| `--permission-mode <mode>` | `acceptEdits` / `auto` / `bypassPermissions` / `manual` / `dontAsk` / `plan` |
| `--allowedTools <tools...>` | 許可するツール名（例: `"Bash(git *) Edit"`） |
| `--disallowedTools <tools...>` | 拒否するツール名 |
| `--tools <tools...>` | 使用可能な組み込みツールを限定する。`""` で全無効、`default` で全有効 |
| `--dangerously-skip-permissions` | 全権限チェックを回避する。ネットワーク遮断されたサンドボックス専用 |
| `--allow-dangerously-skip-permissions` | 上記を選択肢として有効化するが、既定にはしない |
| `--add-dir <directories...>` | ツールがアクセスできるディレクトリを追加する |

## プロンプトと設定の差し込み

| オプション | 説明 |
|---|---|
| `--system-prompt <prompt>` | システムプロンプトを差し替える |
| `--append-system-prompt <prompt>` | 既定のシステムプロンプトに追記する |
| `--settings <file-or-json>` | 追加の設定を JSON ファイルか文字列で読む |
| `--setting-sources <sources>` | 読み込む設定ソースを限定する（`user` / `project` / `local`） |
| `--agent <agent>` | このセッションで使うエージェント |
| `--agents <json>` | カスタムエージェントを JSON で定義する |
| `--mcp-config <configs...>` | MCP サーバーを JSON ファイル／文字列から読む |
| `--strict-mcp-config` | `--mcp-config` の MCP サーバーだけを使う |
| `--plugin-dir <path>` | ディレクトリまたは .zip からプラグインを読む（繰り返し可） |
| `--plugin-url <url>` | URL からプラグイン .zip を取得する（繰り返し可） |
| `--disable-slash-commands` | 全スキルを無効にする |
| `--exclude-dynamic-system-prompt-sections` | 環境依存の情報をシステムプロンプトから最初のユーザーメッセージへ移し、プロンプトキャッシュの再利用率を上げる |

## トラブルシューティング

| オプション | 説明 |
|---|---|
| `--safe-mode` | CLAUDE.md、スキル、プラグイン、フック、MCP など全カスタマイズを無効にして起動する。設定が壊れたときの切り分け用 |
| `--bare` | 最小構成で起動する。フック、LSP、プラグイン同期、CLAUDE.md 自動探索などを飛ばす |
| `-d, --debug [filter]` | デバッグモード（例: `"api,hooks"` や `"!1p,!file"`） |
| `--debug-file <path>` | デバッグログの出力先を指定する |
| `--verbose` | 設定の verbose を上書きする |

`--safe-mode` と `--bare` は目的が違う。**設定を壊した疑いがあるときは `--safe-mode`**（カスタマイズを全部切るが認証・ツール・権限は通常どおり）、**再現性のある最小環境が欲しいときは `--bare`**（認証も `ANTHROPIC_API_KEY` に限定される）。

## 連携・その他

| オプション | 説明 |
|---|---|
| `--ide` | 起動時に IDE へ自動接続する |
| `--chrome` / `--no-chrome` | Claude in Chrome 連携を有効／無効にする |
| `--cloud [description\|id\|url]` | クラウドセッションを作る、または既存のものに接続する |
| `--environment <environment_id>` | 指定した自己ホスト環境でクラウドセッションを作る |
| `--teleport [session]` | teleport セッションを再開する |
| `--remote-control [name]` | Remote Control を有効にして起動する |
| `--bg, --background` | バックグラウンドエージェントとして起動し、即座に戻る |
| `--brief` | エージェントからユーザーへの通知ツールを有効にする |
| `--file <specs...>` | 起動時にダウンロードするファイル（`file_id:相対パス`） |
| `--prompt-suggestions` | プロンプト候補を有効にする |
| `--ax-screen-reader` | スクリーンリーダー向けに装飾なしで描画する |

---

# 6. 自分のコマンドを追加する

カスタムコマンドはスキルに統合された。`.claude/commands/deploy.md` と `.claude/skills/deploy/SKILL.md` はどちらも `/deploy` になり、動作も同じ。既存の `.claude/commands/` はそのまま動く。

スキル形式にすると、補助ファイルを置くディレクトリが持てる、frontmatter で呼び出し主体（自分だけか、Claude も自動で使えるか）を制御できる、といった利点がある。

置き場所は 2 つ。

| 場所 | 適用範囲 |
|---|---|
| `~/.claude/skills/<name>/SKILL.md` | 自分の全プロジェクト |
| `.claude/skills/<name>/SKILL.md` | そのプロジェクトのみ（リポジトリにコミットすればチームで共有できる） |

CLAUDE.md との使い分けは、**読み込まれるタイミング**で決まる。CLAUDE.md は毎セッション読み込まれるので常に必要な事実を書く。スキルの本文は使うときだけ読み込まれるので、長い手順書やリファレンスを置いても普段のコンテキストを圧迫しない。CLAUDE.md の一節が「事実」ではなく「手順」に育ってきたら、スキルに切り出すサインになる。

詳しくは [SKILL.mdの構成](claude_code_skill_md_structure_guide.md) と [Skillの登録方法](claude_code_skill_registration.md) を参照。
