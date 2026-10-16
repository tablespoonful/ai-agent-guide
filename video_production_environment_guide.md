# 動画制作の環境を他の端末で再現する

2D / 3D の解説動画を作る 3 つのプロジェクトを、別の端末でも同じように作れるようにするための仕様。
バージョンは厳密に固定しない (「これ以上」の目安だけ)。例外は wrangler で、理由は下に書く。

| プロジェクト | 中身 | 公開先 |
|---|---|---|
| `~/ai` | AI の内部処理を見せる動画 (2D + Blender の 3D) | https://ai-factory-tour.pages.dev |
| `~/network_security` | ネットワークとセキュリティ (2D) | https://network-security-tour.pages.dev |
| `~/cloud_tour` | AWS (SAA-C03) のしくみ (2D) | https://cloud-tour.pages.dev |

`network_security` と `cloud_tour` の 2D の仕組みは、`~/ai/beginner_video_basics/framework.py` を写して育てたもの。3 つとも同じ考え方で動く。

---

## 1. しくみの全体

### 2D

```
videos/<slug>.py (場面の draw 関数とナレーション)
   │
   ├─ ナレーション → 読み替え (scripts/yomi/yomi.py) → VOICEVOX (HTTP) → video2d/audio_cache/*.wav
   │
   └─ draw(im, d, t) を 24fps で呼ぶ (Pillow で 1280x720 を描く)
         → ffmpeg (imageio-ffmpeg 同梱) に生のフレームを流して mp4 に
         → 場面ごとの mp4 を結合 + 章 + 字幕 (.srt) + storyboard.json
   │
   └─ scripts/build_site.py → site/ (静的 HTML) → wrangler で Cloudflare Pages へ
```

- 解像度 1280x720、24fps (`video2d/framework.py` の `W, H, FPS`)。
- 場面の長さは音声に合わせて自動で伸びる。動きの時間割も同じ割合で伸びる (`_scale`)。
- 音声のキャッシュは「読み替えた後の文」を鍵にする。文と読みの規則が同じなら、2 回目からは VOICEVOX を呼ばない。

### 3D (`~/ai` だけ)

```
Python (uv, PyTorch など) → トレース JSON / npz (traces/)
   → Blender を --background で起動 → blender/build_factory.py がシーンを組んでレンダ (EEVEE)
   → ナレーション版は 2D と同じ VOICEVOX と ffmpeg で合成
```

- Blender 側は Blender 内蔵の Python で動き、torch には依存しない。
- 重いレンダ (Cycles・4K) は `aws-spot` スキルで AWS の GPU インスタンスに出す。

---

## 2. 必要なソフト

| ソフト | 目安 | 用途 | 入れ方 |
|---|---|---|---|
| Git | 何でも | ソースの取得 | 公式の配布物 |
| uv | 0.5 以上 | Python と依存の管理 (`uv sync`) | https://docs.astral.sh/uv/ |
| Python | 3.12 以上 | 本体 (uv が自動で用意する) | uv に任せる |
| Node.js | 18 以上 | `npx wrangler` を動かす | 公式の配布物 |
| wrangler | **4.40.0 に固定** | Cloudflare Pages への公開 | `npx --yes wrangler@4.40.0` (入れる必要なし) |
| VOICEVOX エンジン | 何でも (最新でよい) | ナレーションの音声 | 下の「VOICEVOX」 |
| 日本語フォント | — | 画面の文字 | 下の「フォント」 |
| Blender | 5.x | 3D (`~/ai` だけ) | 公式の配布物 |
| bash | — | `scripts/*.sh` | Windows は Git Bash、Mac / Linux は標準 |
| Docker | 任意 | VOICEVOX をコンテナで動かすとき | 任意 |

- ffmpeg は入れなくてよい。Python の `imageio-ffmpeg` が実行ファイルを同梱している。
- Python の依存 (pillow・numpy・imageio-ffmpeg・cryptography、`~/ai` は torch など) は各プロジェクトの `pyproject.toml` と `uv.lock` にあり、`uv sync` で入る。
- **wrangler だけは固定する**: 新しい版 (4.14x 以降) では `pages` の命令が Workers 側に回され、既存の Pages のプロジェクトを扱えない。`scripts/deploy_pages_slow.sh` は `npx --yes --prefer-offline wrangler@4.40.0` で呼んでいる。

### VOICEVOX

- 話者は **No.7 のアナウンス (speaker 30)**。クレジット「VOICEVOX:No.7」を動画の右下とサイトに出す (個人・非商用のみ可)。
- エンジンは HTTP (`http://127.0.0.1:50021`) で呼ぶ。動いていれば、どの入れ方でもよい。
  - Windows / Mac: 公式サイトのエンジン (または製品版) を入れて起動する。
  - Linux / Docker: 公式のイメージ `voicevox/voicevox_engine` の CPU 版を `-p 50021:50021` で動かす。
- GPU は要らない (この PC も CPU 版)。
- 版が変わると音声がわずかに変わることがある。キャッシュの鍵は文だけなので、版を上げても古いキャッシュはそのまま使われる。版の違いをそろえたいときは、キャッシュを消して作り直す。

### フォント

- 本文は日本語のゴシック、16 進やアドレスは等幅を使う。
- いまは Windows のフォントを直接読んでいる (`meiryo.ttc` / `meiryob.ttc` / `consola.ttf` / `consolab.ttf`)。Mac・Linux には無い。
- 他の端末でも同じ見た目にするには、配布が自由なフォント (例: Noto Sans JP と等幅の Noto Sans Mono や JetBrains Mono) をリポジトリに入れ、それを読むようにする (「5. 移せるようにするための変更」)。
  文字の幅が変わるので、置き換えたら全動画の `--probe` で、はみ出しを見直す。

---

## 3. 端末に固定の設定 (いまの値)

| 何 | いまの場所 | 他の端末では |
|---|---|---|
| VOICEVOX のエンジンの実行ファイル | `scripts/tts/voicevox.py` の `ENGINE` (`C:\Users\table\tools\voicevox\windows-cpu\run.exe`)。止まっていれば自動で起動する | その端末のパス、またはエンジンを先に起動しておく |
| VOICEVOX の URL | 同じファイルの `HOST` (`http://127.0.0.1:50021`) | Docker なら同じ。別の端末で動かすならその URL |
| フォント | `video2d/framework.py` の `FONT` / `FONTB`、`video2d/netparts.py` の `MONO` / `MONOB` | 上の「フォント」 |
| 重い処理の鍵 | `scripts/render_all.sh` の `LOCK` (`/c/Users/table/ai/.heavy_lock`) と各 CLAUDE.md | その端末で 1 か所に決める (同じ端末で複数の作業を並べないなら不要) |
| Blender | `~/ai` は環境変数 `BLENDER` で上書きできる (既定は `C:\Program Files\Blender Foundation\Blender 5.2\blender.exe`) | `BLENDER` を設定する |

- Windows では、日本語を print するコマンドに `PYTHONIOENCODING=utf-8` を付ける (付けないと cp932 で化ける)。Mac / Linux では不要。

---

## 4. 移すもの・移さないもの

| もの | git に入っているか | 他の端末では |
|---|---|---|
| ソース (動画の台本・部品・クイズ・用語集・サイトの生成) | 入っている | `git clone` |
| 読みの規則 `scripts/yomi/yomi.py` | 入っている | そのまま |
| 音声のキャッシュ `video2d/audio_cache/` | 入っていない (cloud_tour で約 2GB) | 作り直す (VOICEVOX を CPU で、全動画なら数時間)。急ぐときは外付けの記憶装置などで写す |
| 書き出した動画 `video2d/out/` | 入っていない (約 1GB) | 作り直す。公開版は Cloudflare にある |
| サイト `site/`・`site_build/` | 入っていない | `build_site.py` で作り直す |
| 3D のトレース・レンダ (`~/ai/traces/`・`renders/`) | プロジェクトの .gitignore による | 作り直す。大きいものは写す |
| Cloudflare の認証 | 入っていない | その端末で `npx wrangler@4.40.0 login` |
| API キー (`~/ai` の OpenRouter など) | 入っていない (`.env`) | その端末の `.env` に書く。git に入れない |

- **いまは 3 つのリポジトリともリモートが無い** (この PC にしかない)。まず非公開のリモート (GitHub など) を作って push する。

---

## 5. 移せるようにするための変更 (未対応)

1. リモートを作って 3 つのリポジトリを push する。
2. 端末に固定の値を環境変数で上書きできるようにする。値が無ければ、いまの値を使う。
   - `VOICEVOX_URL` (既定 `http://127.0.0.1:50021`)
   - `VOICEVOX_ENGINE` (自動で起動するエンジンの実行ファイル。無ければ自動起動しない)
   - `HEAVY_LOCK` (鍵のフォルダ)
   - `FONT_DIR` か、リポジトリに入れたフォントを既定にする
   - `.env.example` に一覧を書く。`.env` は git に入れない (いまの .gitignore のまま)。
3. フォントをリポジトリに入れる (配布が自由なもの)。全動画の `--probe` で見直す。
4. 準備スクリプト (`scripts/setup.sh` など) を 1 つ用意する。
   `uv sync` → VOICEVOX の応答の確認 → フォントの確認 → `wrangler whoami` の確認、を順に行い、足りないものを表示する。
5. (任意) Python・uv・フォント・VOICEVOX をまとめた devcontainer を作る。VS Code や Codespaces でそのまま開ける。

---

## 6. 新しい端末での手順 (5 の変更の後)

```bash
git clone <リモート>/cloud_tour && cd cloud_tour
uv sync
cp .env.example .env            # 必要なら VOICEVOX_URL などを書き換える
# VOICEVOX を起動 (アプリ、または docker run -p 50021:50021 voicevox/voicevox_engine:cpu-latest)
bash scripts/setup.sh           # 足りないものを表示
```

動くかの確かめ方 (どれも短時間で終わる):

```bash
uv run python video2d/make.py c_vpc --probe      # 静止画だけ → video2d/out/c_vpc/build/_sheet.png を見る
uv run python scripts/cues.py c_vpc 1            # VOICEVOX が答えるか (音声を 1 場面ぶん作る)
uv run python scripts/check_quiz.py              # クイズの検査
uv run python scripts/site_glossary.py           # 用語集の検査
```

本番の書き出しと公開:

```bash
bash scripts/render_all.sh site_build/render.log c_vpc
bash scripts/deploy_pages_slow.sh                # 初回は npx wrangler@4.40.0 login が要る
```

- 同じ端末で重い処理を並べないなら、鍵 (`HEAVY_LOCK`) は気にしなくてよい。
- メモリは 16GB で足りる (2D の書き出しは 1 本ずつ)。GPU は要らない。

---

## 7. 同じ出力になるか

- 画面: 同じフォント・同じ Pillow なら、描画は同じになる。フォントが違うと文字の幅が変わり、はみ出すことがある。
- 音声: VOICEVOX の版と話者が同じなら、ほぼ同じになる。キャッシュを写せば完全に同じになる。
- 値: 動画の中の値はすべてコードで計算しているので、端末によらず同じ。乱数は種を固定している。
- 公開: wrangler 4.40.0 と Cloudflare のアカウントが同じなら、同じ URL に上がる。

---

## 8. Claude Code で同じように作業するには

- 各リポジトリの `CLAUDE.md` に、作り方・規約・コマンドが書いてある (リポジトリと一緒に移る)。
- ユーザー全体の設定は別に写す:
  - `~/.claude/CLAUDE.md` (日本語で応答するなどの共通の指示)
  - `~/.claude/skills/` (wrangler・aws-spot・knowledge など)
  - `~/knowledge/` (プロジェクトをまたぐ知見)
  - これらも非公開のリポジトリにしておくと、端末を替えても同じ働き方になる。
- 動画づくりの分担 (台本 → `cues.py` で時刻合わせ → `cuesheet.py` と `--probe` で確認 → 書き出し → 字幕の時刻ごとの画面で確認 → 公開) は、cloud_tour の `CLAUDE.md` に書いてある。
