# AIコーディングエージェント AtCoder 性能比較（2026-08）

> 対象: Claude Code / Codex CLI / Cursor Agent の計 12 モデル
> 測定日: 2026-08-30 〜 08-31
> 問題: AtCoder Problems の difficulty 帯ごとに 10 問、計 60 問（720 試行）
> 判定: 公式テストケース（LiveCodeBench 収録）によるローカル判定

## 要約

- **claude-code:opus が 59/60 (98%) で首位。** 唯一の失点は blue 帯の TLE 1 件で、
  最難関の yellow 帯 (2000-2399) を 10/10 で完走したのは opus だけ
- **codex は推論量の指定がそのまま成績に出た。** sol-high 95% > sol 88% > terra 82% > luna 78%。
  同じ gpt-5.6-sol でも `model_reasoning_effort=high` で 4 問ぶん上回る（推論トークンは約 1.8 倍）
- **速度と正解率の両立では cursor:gemini-3.7-flash-high。** 90% (54/60) を平均 166 秒で出しており、
  同等の正解率の sol-high (95%, 304 秒) や sonnet (93%, 367 秒) より大幅に速い
- **失敗の中身を分けると評価が変わる。** composer-2.5 の失敗 27 件のうち 13 件、
  cursor:auto の 21 件のうち 12 件は「Python 指定を無視して C++ で解答した」もの。
  解けなかったのではなく指示に従わなかった結果で、能力の指標としては読めない
- **haiku の失敗 35 件のうち 32 件が計算量不足 (TLE)。** 方針は立つが計算量を詰め切れない、
  という一貫した傾向。Python 指定の影響を最も強く受けている
- 難易度が上がるほど各モデルの差が開く。gray 帯は 12 モデル中 11 が満点で差がつかない

---

run: `20260830-115924_official (最終)`

判定に使ったテストケース: 公式テストケース (LiveCodeBench 収録) 720件

集計から除外した問題: abc337_e（対話型）。固定テストケースでは正しい解答も RE になるため、全参加者ぶん取り除いています。

## 評価対象の内訳

`エイリアス` は CLI に渡した指定、`実モデル` は実際に応答したモデルです。
推論量は CLI 既定値のままのものは「既定」と表記しています。

| 参加者 | エイリアス | 実モデル | 推論量 |
|---|---|---|---|
| claude-code:opus | `opus` | claude-opus-5 | CLI 既定 |
| claude-code:sonnet | `sonnet` | claude-sonnet-5 | CLI 既定 |
| codex:gpt-5.6-luna | `gpt-5.6-luna` | （CLI が出力しないため不明） | medium（config.toml 既定） |
| claude-code:haiku | `haiku` | claude-haiku-4-5-20251001 | CLI 既定 |
| codex:gpt-5.6-terra | `gpt-5.6-terra` | （CLI が出力しないため不明） | medium（config.toml 既定） |
| codex:gpt-5.6-sol | `gpt-5.6-sol` | （CLI が出力しないため不明） | medium（config.toml 既定） |
| codex:gpt-5.6-sol-high | `gpt-5.6-sol` | （CLI が出力しないため不明） | high |
| cursor:auto | `auto` | （CLI が出力しないため不明） | Cursor が自動選択 |
| cursor:composer-2.5 | `composer-2.5` | （CLI が出力しないため不明） | 指定なし |
| cursor:gemini-3.7-flash-high | `gemini-3.7-flash-high` | （CLI が出力しないため不明） | high（モデル ID に含む） |
| cursor:cursor-grok-4.6-high | `cursor-grok-4.6-high` | （CLI が出力しないため不明） | high（モデル ID に含む） |
| cursor:kimi-k3-high | `kimi-k3-high` | （CLI が出力しないため不明） | high（モデル ID に含む） |

## 失敗原因の内訳

`環境エラー` は測定基盤側の失敗（メモリ不足など）で、モデルの評価には数えるべきではありません。
`指定言語外` は指定言語以外で解答したもので、解けなかったこととは区別されます。

| 参加者 | アルゴリズム誤り | 計算量不足 | 実行時エラー | 指定言語外 | エージェント打ち切り | 解答を書かなかった |
|---|---|---|---|---|---|---|
| claude-code:opus | - | 1 | - | - | - | - |
| claude-code:sonnet | 1 | 2 | - | - | 1 | - |
| codex:gpt-5.6-luna | 3 | 9 | 1 | - | - | - |
| claude-code:haiku | 2 | 32 | 1 | - | - | - |
| codex:gpt-5.6-terra | 2 | 9 | - | - | - | - |
| codex:gpt-5.6-sol | 1 | 6 | - | - | - | - |
| codex:gpt-5.6-sol-high | - | 3 | - | - | - | - |
| cursor:auto | - | 6 | - | 12 | - | 3 |
| cursor:composer-2.5 | 3 | 10 | - | 13 | 1 | - |
| cursor:gemini-3.7-flash-high | - | 6 | - | - | - | - |
| cursor:cursor-grok-4.6-high | - | 4 | - | 1 | - | - |
| cursor:kimi-k3-high | 4 | 6 | 2 | 1 | - | - |

## 難易度帯ごとの正解率 (AC 数 / 試行数)

| 難易度帯 | claude-code:opus | claude-code:sonnet | codex:gpt-5.6-luna | claude-code:haiku | codex:gpt-5.6-terra | codex:gpt-5.6-sol | codex:gpt-5.6-sol-high | cursor:auto | cursor:composer-2.5 | cursor:gemini-3.7-flash-high | cursor:cursor-grok-4.6-high | cursor:kimi-k3-high |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| gray 0-399 | 10/10 (100%) | 10/10 (100%) | 10/10 (100%) | 10/10 (100%) | 10/10 (100%) | 10/10 (100%) | 10/10 (100%) | 9/10 (90%) | 10/10 (100%) | 10/10 (100%) | 10/10 (100%) | 10/10 (100%) |
| brown 400-799 | 10/10 (100%) | 10/10 (100%) | 9/10 (90%) | 6/10 (60%) | 10/10 (100%) | 10/10 (100%) | 10/10 (100%) | 8/10 (80%) | 6/10 (60%) | 10/10 (100%) | 9/10 (90%) | 8/10 (80%) |
| green 800-1199 | 10/10 (100%) | 10/10 (100%) | 9/10 (90%) | 4/10 (40%) | 9/10 (90%) | 10/10 (100%) | 10/10 (100%) | 9/10 (90%) | 6/10 (60%) | 10/10 (100%) | 9/10 (90%) | 9/10 (90%) |
| cyan 1200-1599 | 10/10 (100%) | 10/10 (100%) | 7/10 (70%) | 4/10 (40%) | 8/10 (80%) | 9/10 (90%) | 10/10 (100%) | 7/10 (70%) | 5/10 (50%) | 10/10 (100%) | 10/10 (100%) | 9/10 (90%) |
| blue 1600-1999 | 9/10 (90%) | 8/10 (80%) | 6/10 (60%) | 1/10 (10%) | 7/10 (70%) | 6/10 (60%) | 8/10 (80%) | 4/10 (40%) | 4/10 (40%) | 8/10 (80%) | 9/10 (90%) | 6/10 (60%) |
| yellow 2000-2399 | 10/10 (100%) | 8/10 (80%) | 6/10 (60%) | 0/10 (0%) | 5/10 (50%) | 8/10 (80%) | 9/10 (90%) | 2/10 (20%) | 2/10 (20%) | 6/10 (60%) | 8/10 (80%) | 5/10 (50%) |
| **合計** | **59/60 (98%)** | **56/60 (93%)** | **47/60 (78%)** | **25/60 (42%)** | **49/60 (82%)** | **53/60 (88%)** | **57/60 (95%)** | **39/60 (65%)** | **33/60 (55%)** | **54/60 (90%)** | **55/60 (92%)** | **47/60 (78%)** |

## 難易度帯ごとの平均所要時間 (秒/問)

| 難易度帯 | claude-code:opus | claude-code:sonnet | codex:gpt-5.6-luna | claude-code:haiku | codex:gpt-5.6-terra | codex:gpt-5.6-sol | codex:gpt-5.6-sol-high | cursor:auto | cursor:composer-2.5 | cursor:gemini-3.7-flash-high | cursor:cursor-grok-4.6-high | cursor:kimi-k3-high |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| gray 0-399 | 59 | 74 | 119 | 121 | 102 | 174 | 175 | 70 | 72 | 107 | 90 | 80 |
| brown 400-799 | 173 | 137 | 112 | 131 | 101 | 179 | 247 | 78 | 78 | 220 | 118 | 105 |
| green 800-1199 | 92 | 106 | 110 | 162 | 89 | 200 | 234 | 124 | 71 | 145 | 127 | 136 |
| cyan 1200-1599 | 223 | 364 | 135 | 208 | 107 | 237 | 256 | 80 | 64 | 108 | 303 | 157 |
| blue 1600-1999 | 494 | 514 | 174 | 298 | 119 | 321 | 450 | 138 | 268 | 192 | 265 | 526 |
| yellow 2000-2399 | 512 | 1010 | 160 | 290 | 188 | 366 | 461 | 142 | 205 | 223 | 654 | 374 |
| **全体平均** | **259** | **367** | **135** | **202** | **118** | **246** | **304** | **105** | **126** | **166** | **259** | **230** |

## エージェント・モデル別サマリ

| 参加者 | 試行 | AC | 正解率 | 提出 | 合計時間 | 平均/問 | 最長/問 | 総コスト(USD) | 出力トークン |
|---|---|---|---|---|---|---|---|---|---|
| claude-code:opus | 60 | 59 | 98% | 0 | 259.0分 | 259s | 1492s | 32.149 | 472,365 |
| claude-code:sonnet | 60 | 56 | 93% | 0 | 367.4分 | 367s | 1500s | 25.288 | 557,778 |
| codex:gpt-5.6-luna | 60 | 47 | 78% | 0 | 135.0分 | 135s | 245s | 0.000 | 207,838 |
| claude-code:haiku | 60 | 25 | 42% | 0 | 201.6分 | 202s | 1496s | 8.640 | 681,817 |
| codex:gpt-5.6-terra | 60 | 49 | 82% | 0 | 117.6分 | 118s | 262s | 0.000 | 177,604 |
| codex:gpt-5.6-sol | 60 | 53 | 88% | 0 | 246.0分 | 246s | 780s | 0.000 | 336,104 |
| codex:gpt-5.6-sol-high | 60 | 57 | 95% | 0 | 303.9分 | 304s | 1230s | 0.000 | 465,984 |
| cursor:auto | 60 | 39 | 65% | 0 | 105.3分 | 105s | 796s | 0.000 | 0 |
| cursor:composer-2.5 | 60 | 33 | 55% | 0 | 126.4分 | 126s | 1200s | 0.000 | 0 |
| cursor:gemini-3.7-flash-high | 60 | 54 | 90% | 0 | 165.9分 | 166s | 720s | 0.000 | 0 |
| cursor:cursor-grok-4.6-high | 60 | 55 | 92% | 0 | 259.4分 | 259s | 1502s | 0.000 | 0 |
| cursor:kimi-k3-high | 60 | 47 | 78% | 0 | 229.7分 | 230s | 1200s | 0.000 | 0 |

## 判定内訳

| 参加者 | AC | WA | TLE | RE | CE | 未提出 | エラー | 時間切れ |
|---|---|---|---|---|---|---|---|---|
| claude-code:opus | 59 | 0 | 1 | 0 | 0 | 0 | 0 | 4 |
| claude-code:sonnet | 56 | 1 | 2 | 0 | 0 | 1 | 0 | 10 |
| codex:gpt-5.6-luna | 47 | 3 | 9 | 1 | 0 | 0 | 0 | 0 |
| claude-code:haiku | 25 | 2 | 32 | 1 | 0 | 0 | 0 | 1 |
| codex:gpt-5.6-terra | 49 | 2 | 9 | 0 | 0 | 0 | 0 | 0 |
| codex:gpt-5.6-sol | 53 | 1 | 6 | 0 | 0 | 0 | 0 | 0 |
| codex:gpt-5.6-sol-high | 57 | 0 | 3 | 0 | 0 | 0 | 0 | 1 |
| cursor:auto | 39 | 0 | 6 | 0 | 0 | 15 | 0 | 1 |
| cursor:composer-2.5 | 33 | 3 | 10 | 0 | 0 | 14 | 0 | 1 |
| cursor:gemini-3.7-flash-high | 54 | 0 | 6 | 0 | 0 | 0 | 0 | 1 |
| cursor:cursor-grok-4.6-high | 55 | 0 | 4 | 0 | 0 | 1 | 0 | 2 |
| cursor:kimi-k3-high | 47 | 4 | 6 | 2 | 0 | 1 | 0 | 2 |

## 問題ごとの結果

| 問題 | 難易度 | 帯 | claude-code:opus | claude-code:sonnet | codex:gpt-5.6-luna | claude-code:haiku | codex:gpt-5.6-terra | codex:gpt-5.6-sol | codex:gpt-5.6-sol-high | cursor:auto | cursor:composer-2.5 | cursor:gemini-3.7-flash-high | cursor:cursor-grok-4.6-high | cursor:kimi-k3-high |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| [abc346_a](https://atcoder.jp/contests/abc346/tasks/abc346_a) | 10 | gray 0-399 | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC |
| [abc352_a](https://atcoder.jp/contests/abc352/tasks/abc352_a) | 14 | gray 0-399 | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC |
| [abc361_a](https://atcoder.jp/contests/abc361/tasks/abc361_a) | 15 | gray 0-399 | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC |
| [abc305_b](https://atcoder.jp/contests/abc305/tasks/abc305_b) | 23 | gray 0-399 | AC | AC | AC | AC | AC | AC | AC | NO_SUBMISSION | AC | AC | AC | AC |
| [abc327_a](https://atcoder.jp/contests/abc327/tasks/abc327_a) | 23 | gray 0-399 | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC |
| [abc326_a](https://atcoder.jp/contests/abc326/tasks/abc326_a) | 28 | gray 0-399 | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC |
| [abc355_b](https://atcoder.jp/contests/abc355/tasks/abc355_b) | 105 | gray 0-399 | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC |
| [abc339_c](https://atcoder.jp/contests/abc339/tasks/abc339_c) | 174 | gray 0-399 | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC |
| [abc358_c](https://atcoder.jp/contests/abc358/tasks/abc358_c) | 273 | gray 0-399 | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC |
| [abc357_c](https://atcoder.jp/contests/abc357/tasks/abc357_c) | 371 | gray 0-399 | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC |
| [abc371_d](https://atcoder.jp/contests/abc371/tasks/abc371_d) | 408 | brown 400-799 | AC | AC | AC | TLE | AC | AC | AC | AC | NO_SUBMISSION | AC | AC | TLE |
| [abc303_c](https://atcoder.jp/contests/abc303/tasks/abc303_c) | 417 | brown 400-799 | AC | AC | AC | WA | AC | AC | AC | AC | AC | AC | AC | AC |
| [abc330_c](https://atcoder.jp/contests/abc330/tasks/abc330_c) | 519 | brown 400-799 | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC |
| [abc356_c](https://atcoder.jp/contests/abc356/tasks/abc356_c) | 568 | brown 400-799 | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC |
| [abc306_d](https://atcoder.jp/contests/abc306/tasks/abc306_d) | 596 | brown 400-799 | AC | AC | AC | AC | AC | AC | AC | AC | NO_SUBMISSION | AC | AC | AC |
| [abc363_c](https://atcoder.jp/contests/abc363/tasks/abc363_c) | 602 | brown 400-799 | AC | AC | TLE | TLE | AC | AC | AC | TLE | NO_SUBMISSION | AC | TLE | TLE |
| [abc308_c](https://atcoder.jp/contests/abc308/tasks/abc308_c) | 605 | brown 400-799 | AC | AC | AC | TLE | AC | AC | AC | TLE | AC | AC | AC | AC |
| [abc319_d](https://atcoder.jp/contests/abc319/tasks/abc319_d) | 631 | brown 400-799 | AC | AC | AC | AC | AC | AC | AC | AC | NO_SUBMISSION | AC | AC | AC |
| [abc302_d](https://atcoder.jp/contests/abc302/tasks/abc302_d) | 682 | brown 400-799 | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC |
| [abc327_d](https://atcoder.jp/contests/abc327/tasks/abc327_d) | 709 | brown 400-799 | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC |
| [abc368_d](https://atcoder.jp/contests/abc368/tasks/abc368_d) | 816 | green 800-1199 | AC | AC | AC | TLE | AC | AC | AC | AC | NO_SUBMISSION | AC | AC | AC |
| [abc341_d](https://atcoder.jp/contests/abc341/tasks/abc341_d) | 829 | green 800-1199 | AC | AC | AC | AC | AC | AC | AC | NO_SUBMISSION | AC | AC | AC | AC |
| [abc347_c](https://atcoder.jp/contests/abc347/tasks/abc347_c) | 926 | green 800-1199 | AC | AC | AC | WA | AC | AC | AC | AC | NO_SUBMISSION | AC | AC | AC |
| [abc342_d](https://atcoder.jp/contests/abc342/tasks/abc342_d) | 944 | green 800-1199 | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | TLE |
| [abc318_e](https://atcoder.jp/contests/abc318/tasks/abc318_e) | 1004 | green 800-1199 | AC | AC | AC | TLE | AC | AC | AC | AC | NO_SUBMISSION | AC | AC | AC |
| [abc304_d](https://atcoder.jp/contests/abc304/tasks/abc304_d) | 1015 | green 800-1199 | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC |
| [abc365_e](https://atcoder.jp/contests/abc365/tasks/abc365_e) | 1102 | green 800-1199 | AC | AC | WA | TLE | WA | AC | AC | AC | AC | AC | AC | AC |
| [abc348_d](https://atcoder.jp/contests/abc348/tasks/abc348_d) | 1108 | green 800-1199 | AC | AC | AC | TLE | AC | AC | AC | AC | AC | AC | TLE | AC |
| [abc328_e](https://atcoder.jp/contests/abc328/tasks/abc328_e) | 1160 | green 800-1199 | AC | AC | AC | TLE | AC | AC | AC | AC | TLE | AC | AC | AC |
| [abc332_d](https://atcoder.jp/contests/abc332/tasks/abc332_d) | 1175 | green 800-1199 | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC |
| [abc354_e](https://atcoder.jp/contests/abc354/tasks/abc354_e) | 1223 | cyan 1200-1599 | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC |
| [abc322_d](https://atcoder.jp/contests/abc322/tasks/abc322_d) | 1310 | cyan 1200-1599 | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC |
| [abc348_e](https://atcoder.jp/contests/abc348/tasks/abc348_e) | 1347 | cyan 1200-1599 | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC | AC |
| [abc325_d](https://atcoder.jp/contests/abc325/tasks/abc325_d) | 1367 | cyan 1200-1599 | AC | AC | AC | TLE | AC | AC | AC | NO_SUBMISSION | AC | AC | AC | AC |
| [abc367_e](https://atcoder.jp/contests/abc367/tasks/abc367_e) | 1370 | cyan 1200-1599 | AC | AC | AC | TLE | AC | TLE | AC | AC | NO_SUBMISSION | AC | AC | AC |
| [abc349_e](https://atcoder.jp/contests/abc349/tasks/abc349_e) | 1464 | cyan 1200-1599 | AC | AC | WA | AC | AC | AC | AC | AC | AC | AC | AC | AC |
| [abc342_e](https://atcoder.jp/contests/abc342/tasks/abc342_e) | 1476 | cyan 1200-1599 | AC | AC | AC | TLE | AC | AC | AC | AC | NO_SUBMISSION | AC | AC | AC |
| [abc356_e](https://atcoder.jp/contests/abc356/tasks/abc356_e) | 1506 | cyan 1200-1599 | AC | AC | TLE | TLE | TLE | AC | AC | TLE | TLE | AC | AC | AC |
| [abc315_d](https://atcoder.jp/contests/abc315/tasks/abc315_d) | 1531 | cyan 1200-1599 | AC | AC | AC | TLE | AC | AC | AC | AC | TLE | AC | AC | WA |
| [abc373_e](https://atcoder.jp/contests/abc373/tasks/abc373_e) | 1592 | cyan 1200-1599 | AC | AC | TLE | TLE | TLE | AC | AC | NO_SUBMISSION | NO_SUBMISSION | AC | AC | AC |
| [abc341_f](https://atcoder.jp/contests/abc341/tasks/abc341_f) | 1604 | blue 1600-1999 | AC | AC | AC | RE | AC | WA | AC | AC | AC | AC | AC | WA |
| [abc321_e](https://atcoder.jp/contests/abc321/tasks/abc321_e) | 1627 | blue 1600-1999 | AC | AC | AC | TLE | AC | AC | AC | AC | NO_SUBMISSION | AC | AC | AC |
| [abc351_e](https://atcoder.jp/contests/abc351/tasks/abc351_e) | 1627 | blue 1600-1999 | AC | AC | AC | TLE | AC | AC | AC | AC | AC | AC | AC | AC |
| [arc181_b](https://atcoder.jp/contests/arc181/tasks/arc181_b) | 1633 | blue 1600-1999 | AC | NO_SUBMISSION | AC | TLE | AC | AC | AC | NO_SUBMISSION | NO_SUBMISSION | AC | AC | WA |
| [abc315_f](https://atcoder.jp/contests/abc315/tasks/abc315_f) | 1674 | blue 1600-1999 | AC | AC | AC | TLE | AC | AC | AC | NO_SUBMISSION | TLE | AC | AC | AC |
| [abc372_f](https://atcoder.jp/contests/abc372/tasks/abc372_f) | 1722 | blue 1600-1999 | TLE | TLE | WA | TLE | TLE | TLE | AC | NO_SUBMISSION | TLE | TLE | AC | AC |
| [abc324_f](https://atcoder.jp/contests/abc324/tasks/abc324_f) | 1815 | blue 1600-1999 | AC | AC | TLE | TLE | TLE | TLE | TLE | TLE | AC | AC | TLE | RE |
| [abc338_f](https://atcoder.jp/contests/abc338/tasks/abc338_f) | 1835 | blue 1600-1999 | AC | AC | TLE | TLE | WA | TLE | TLE | TLE | TLE | TLE | AC | AC |
| [abc364_f](https://atcoder.jp/contests/abc364/tasks/abc364_f) | 1878 | blue 1600-1999 | AC | AC | AC | TLE | AC | AC | AC | TLE | NO_SUBMISSION | AC | AC | WA |
| [abc310_f](https://atcoder.jp/contests/abc310/tasks/abc310_f) | 1938 | blue 1600-1999 | AC | AC | RE | AC | AC | AC | AC | AC | AC | AC | AC | AC |
| [abc375_g](https://atcoder.jp/contests/abc375/tasks/abc375_g) | 2014 | yellow 2000-2399 | AC | AC | AC | TLE | TLE | AC | AC | AC | WA | AC | AC | AC |
| [abc373_f](https://atcoder.jp/contests/abc373/tasks/abc373_f) | 2018 | yellow 2000-2399 | AC | TLE | TLE | TLE | TLE | AC | AC | NO_SUBMISSION | TLE | TLE | AC | NO_SUBMISSION |
| [arc183_c](https://atcoder.jp/contests/arc183/tasks/arc183_c) | 2018 | yellow 2000-2399 | AC | AC | TLE | TLE | TLE | AC | TLE | NO_SUBMISSION | TLE | TLE | AC | AC |
| [abc374_f](https://atcoder.jp/contests/abc374/tasks/abc374_f) | 2026 | yellow 2000-2399 | AC | AC | AC | TLE | AC | AC | AC | NO_SUBMISSION | AC | AC | AC | AC |
| [abc312_e](https://atcoder.jp/contests/abc312/tasks/abc312_e) | 2047 | yellow 2000-2399 | AC | AC | AC | TLE | AC | AC | AC | NO_SUBMISSION | TLE | AC | AC | TLE |
| [abc369_g](https://atcoder.jp/contests/abc369/tasks/abc369_g) | 2055 | yellow 2000-2399 | AC | AC | AC | TLE | AC | AC | AC | NO_SUBMISSION | WA | AC | NO_SUBMISSION | AC |
| [abc376_f](https://atcoder.jp/contests/abc376/tasks/abc376_f) | 2089 | yellow 2000-2399 | AC | WA | TLE | TLE | TLE | TLE | AC | NO_SUBMISSION | NO_SUBMISSION | TLE | TLE | TLE |
| [abc370_f](https://atcoder.jp/contests/abc370/tasks/abc370_f) | 2102 | yellow 2000-2399 | AC | AC | TLE | TLE | TLE | TLE | AC | NO_SUBMISSION | TLE | TLE | AC | TLE |
| [abc368_e](https://atcoder.jp/contests/abc368/tasks/abc368_e) | 2140 | yellow 2000-2399 | AC | AC | AC | TLE | AC | AC | AC | AC | AC | AC | AC | AC |
| [abc371_g](https://atcoder.jp/contests/abc371/tasks/abc371_g) | 2348 | yellow 2000-2399 | AC | AC | AC | TLE | AC | AC | AC | NO_SUBMISSION | WA | AC | AC | RE |

## 測定条件と、数字を読むときの注意

### 測定条件
- 解答言語は **Python 3 (CPython)** に固定。全参加者に同一のプロンプトを渡している
- 1 問につき 1 試行。難易度帯ごとに実行時間の上限を設定（gray 600 秒 〜 yellow 1500 秒）
- 判定は手元で公式テストケースを実行。最初に失敗したケースで打ち切る
- 各試行は使い捨ての作業ディレクトリで実行し、サンプル入出力のみを与える

### 数字の限界
- **Python 固定の影響が大きい。** 失敗の最多要因は計算量不足 (TLE) で、C++ を指定すれば
  通っていた解答が相当数含まれる。これは「Python での競技プログラミング能力」であって、
  アルゴリズム設計能力そのものではない
- **部分点は測れない。** 最初の失敗ケースで判定を打ち切るため、`通過数 / 全体数` は
  実際に走ったケース数に対する値で、惜しい失敗と全く駄目な失敗を区別できない
- **コストは Claude Code しか取得できない。** Codex と Cursor の CLI は USD を出力しない。
  Cursor はトークン数も未取得（テキスト出力で実行したため）
- **入力トークン数はエージェント間で比較できない。** Codex はキャッシュ読み込みを含む累計、
  Claude Code はキャッシュを除いた純粋な入力を報告している。出力トークンは比較可能
- **ターン数も比較できない。** Claude Code の `num_turns` と Codex のターン数は定義が違い、
  Cursor は取得できない
- 1 問 1 試行なので、たまたま失敗した分の揺れが数ポイント分は乗る

### 測定中に見つかった基盤側の問題（対処済み）
- 対話型問題 (abc337_e) は固定テストケースで判定不能。検出して除外し、代替問題に差し替えた
- Cursor はタイムアウト時に孫プロセスが残り、実行全体を止めることがある
  （プロセスツリーごと終了させるよう修正）
- Cursor は並列実行時にメモリ枯渇と設定ファイルの競合を起こす。並列度を上げられない
- 上記で失敗した試行は無効な測定として破棄し、すべて再実行した

