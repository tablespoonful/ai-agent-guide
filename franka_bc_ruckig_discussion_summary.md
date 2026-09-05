# Franka 20Hz BC + Ruckig ガタつき問題：検討まとめ

作成日：2026-09-05
対象：Franka FR3 / ROS2 Jazzy / franka_ros2 v3.0.0 / BC推論 20Hz / Cartesian pose 渡し / Ruckig 使用
関連ファイル：`franka_bc_ruckig_jitter_prompt_v4.md`（Claude Code 向け調査・修正プロンプト）

---

## 目次

1. [元プロンプトのレビューと見落とし](#sec1)
2. [前提の修正：MoveIt不使用・Cartesian pose 渡し](#sec2)
3. [MoveIt Servo を使う案の評価](#sec3)
4. [実機で20Hz推論を滑らかに動かしている事例と、シムが滑らかな理由](#sec4)
5. [ガタつきの原因（噛み砕き版）](#sec5)
6. [各原因への対策と確かめ方（噛み砕き版）](#sec6)
7. [IsaacLab IK-Rel タスクの action 定義（ソース確認済み）](#sec7)
8. [Ruckig を自前実装に取り入れるべきか](#sec8)
9. [他のロボットアームで使われている滑らか・繊細な制御方法](#sec9)
10. [Cartesian impedance とは](#sec10)
11. [分岐点：速度管理化 vs インピーダンス化](#sec11)
12. [トルク制御の「安全責任」と残る危険](#sec12)
13. [franka_ros2 リポジトリで提供されているもの（実調査）](#sec13)
14. [elbow 指令とは](#sec14)
15. [結論と次の行動](#sec15)
16. [cartesian_impedance_example_controller には何を入力するか](#sec16)
17. [Isaac Sim はインピーダンス制御か](#sec17)
18. [Cartesian impedance と関節空間インピーダンス（PD）の違い](#sec18)

---

<a id="sec1"></a>
## 1. 元プロンプトのレビューと見落とし

元プロンプトは「Ruckigの使い方・current state の連続性・IK・command 経路」を順に確認させる骨格で、方向性は正しい。ただし **Ruckig が正しく動いていてもガタつく経路** が数か所抜けていた。影響が大きい順：

1. **`target_velocity = 0` による stop-and-go（最重要）**
   Ruckig の target_velocity はデフォルト 0。50ms ごとに target を更新しても「その target で止まる軌道」を毎回生成するため、pass_to_input が完璧でも 50ms 周期で加減速する。元プロンプトは current 側のゼロ初期化しか見ていなかった。
2. **command 経路の特定を最初に行う**
   Ruckig 出力が JTC（二重補間）や topic 経由になっていれば Ruckig を直しても効かない。
3. **delta action の基準が measured になっていないか**
   `target = measured + delta` だと追従遅れとノイズが閉ループで戻り発振する。
4. **quaternion の二重被覆（q / −q）** と IK 初期値の出所。
5. **1kHz スレッドのブロッキング**：gripper API（ブロッキング）、ログ I/O、Python なら torch との GIL 競合。
6. **切り分け実験の欠如**：オフライン再生・合成正弦波 target・HDF5 トレース再生で「BC 由来かパイプライン由来か」を実験で決める。
7. **完了条件が定性的**：velocity PSD の 20Hz ピーク、jerk RMS、周期 jitter p99 を修正前後で数値報告させる。

補足：「絶対に解決できるプロンプト」は原理的に存在しない。Phase 分け（読み取り専用調査 → 計測 → 修正 → 承認後に実機）と切り分け実験を強制することで、「推定で直して症状を隠す」失敗を防ぐのが狙い。

---

<a id="sec2"></a>
## 2. 前提の修正：MoveIt不使用・Cartesian pose 渡し

大輔さんの構成は **MoveIt を使わず、Cartesian pose をそのまま Franka に渡す（IK は Franka 内部）**。これにより：

- MoveIt・JTC・自前 IK 関連の確認項目を削除し、command 経路の分類を libfranka 直／franka_ros2 controller／topic 経由／Python ラッパーに絞った。
- 「自分のコードに存在しない IK を探させる」項目の代わりに、**Franka 内部 IK の挙動**（4.8）を置いた。`O_T_EE_c`（Franka が受理した Cartesian）が滑らかなのに `q_d`（内部 IK 出力）が折れていれば、原因は elbow・特異点・joint 制限張り付き。
- Cartesian 構成特有の新規チェック：Ruckig の姿勢 DoF の扱い（quaternion 4 成分を独立に入れると非正規化と符号反転で大回り、rotation vector なら wrap-around）。
- libfranka CartesianPose 固有の要件：初期値は `O_T_EE`（measured）ではなく `O_T_EE_c`（commanded）、毎周期必ず書く、`O_T_EE_d` vs `O_T_EE_c` でクリップの有無を検出。
- Cartesian 制限値がそのまま Ruckig の制限値になる。

---

<a id="sec3"></a>
## 3. MoveIt Servo を使う案の評価

**結論：今のガタつきの「修正」としては勧めない。原因未特定の段階でパイプラインを差し替えると、原因が消えたのか隠れたのか分からなくなる。**

MoveIt Servo の実態（Jazzy 2.14 系）：

- ライセンスは BSD-3-Clause（商用可）。
- 入力は twist / joint jog / pose tracking、出力は joint command。特異点・衝突に近づくと速度をスケールダウン。衝突チェックとスムージングは無効化可能。
- スムージングはプラグイン式（Butterworth 等）。2.11.0（2024-09）で Ruckig jerk 制限スムージングが追加されたが、直後に不具合報告あり。UR5e の pose tracking で加速度制限が読まれず荒い動きになる報告もある。

構造的な問題点：

1. Servo は独立ノードで topic 経由で ros2_control の controller に渡す＝「避けるべき」とした topic 経由＋controller 側補間の経路。
2. Servo は自前 IK（Jacobian ベース）を行うので、Cartesian pose 渡しから joint command への方式変更になる。
3. 特異点付近の速度スケーリングは「BC の target にわざと追従しない」挙動で、BC の閉ループから見ると未知の外乱。
4. 主要因（target_velocity=0、delta の measured 基準、BC target の反転）は Servo に替えても残る。pose tracking は P 制御で twist を出すので stop-and-go は起きにくい、という点だけは利点。

提案：まず Phase 1 で原因を確定。Franka 内部 IK / 特異点 / elbow が原因なら joint 空間へ移行するが、その際は Servo ではなく **自前 ros2_control controller 内で Ruckig を velocity interface で回す** 方が遅延・依存少なく同等効果。Servo が本当に効くのは「衝突チェック・特異点保護を運用上必須にしたい」場合。

---

<a id="sec4"></a>
## 4. 実機で20Hz推論を滑らかに動かしている事例と、シムが滑らかな理由

### 4.1 事例

「20Hz で単発の target をそのまま Ruckig に渡して滑らか」という事例は稀。滑らかに動いている系はほぼ **action chunk＋高周期の低レベル制御＋境界処理** の構造。

- ACT（ALOHA）：50Hz の joint chunk を temporal ensembling で重ね合わせ。
- Diffusion Policy：推論は約 10Hz でも 8〜16 ステップの chunk を出し、内部を低レベル制御器（1kHz impedance / PD）が補間して追従。
- π0 / π0.5 / SmolVLA：50 ステップ（1 秒分）の chunk。chunk 境界の不連続を Real-Time Chunking（RTC）で処理。次の chunk を現在の chunk 実行中に非同期生成し、確実に実行される部分を凍結して残りを inpaint。300ms 超の推論遅延でも滑らか。chunking だけでは遅延問題は解決せず、chunk 境界で停止や分布外のガタつきが起きる。
- 逆の例：OpenVLA 系（5〜10Hz 単発）は実機でガタつくことで知られ、後続研究が chunking を入れた主因の一つ。

共通点：「モデルが出すのは密な軌道片、ロボットが追うのは 1kHz 制御器」で、**20Hz のステップ境界そのものを制御器に見せない**。

### 4.2 Isaac Sim / Isaac Lab が滑らかな理由

1. **遅延ゼロ・jitter ゼロ**：ポリシー周期は物理ステップの整数倍（decimation）に固定され、観測した状態のまさにその瞬間に action が適用される。実機はカメラ遅延＋推論＋通信で 50〜100ms 前の状態に対して行動するので、`target = 現在状態 + delta` が閉ループの発振源になる。
2. **理想的な関節ドライブ**：action は decimation の間 ZOH で保持され、implicit PD（stiffness / damping）が追従する。この PD が事実上の低域フィルタで、jerk 制限も反射停止もない。実機 Franka は加速度・jerk 違反で reflex が出る硬い制御器。
3. **delta 基準の整合**：シムでは「観測状態＝制御器が実際に到達した状態」なので delta の基準がずれない。実機では commanded と measured が常にずれる。
4. **ノイズ・摩擦・バックラッシュなし**、描画は物理とは別に補間されるので 20Hz で 12.5mm 刻みでも見た目に段差が出ない。

### 4.3 含意

- 現行モデルが単発 20Hz 出力なら、制御側でできる最善は「target を位置ではなく速度 feedforward 付きの移動目標として扱う」（Ruckig velocity interface）。
- 根本的にはモデル側を chunk 出力にし、chunk 内補間＋境界処理を制御器に置くのが事例と同じ構造。
- どちらにせよ現行構成の原因切り分けは必要。target_velocity=0 や delta 基準が原因なら chunk 化しても同じ症状が出る。

---

<a id="sec5"></a>
## 5. ガタつきの原因（噛み砕き版）

**1. 「止まる前提の道案内」問題（最有力）**
Ruckig に target を渡すとき「着いたときの速度」を指定しないと「着いたら止まる」軌道を作る。タクシーに 50ms ごとに「次はあの電柱まで」と言うと、運転手は電柱ごとにブレーキを踏む。次の target は 12mm 先くらいなので、加速する暇もなく減速に入る。

**2. 「遅れた地図を見て修正」問題**
BC が「今の位置から +3mm 右へ」と出すとき、「今の位置」をセンサ実測から取ると、実測は指令より遅れているので目標が手前に戻る。次の周期でもまた手前に…。目標が前後にふらつき、ロボットもふらつく。正しくは「前回出した指令位置から +3mm」（または速度として解釈）。

**3. 「毎回、今の位置からやり直し」問題**
Ruckig は「今どこにいて、どの速度で、どの加速度か」を覚えて次の一歩を計算する。これを 50ms ごとに実測で上書きすると、遅れとノイズで「思ってた位置と違う」と軌道を引き直し、引き直すたびに折れができる。

**4. 「1 秒に 20 回しか指示を送っていない」問題**
Ruckig が 1kHz で滑らかな軌道を作っても、Franka に送るのが 20Hz なら 50ms ごとの段差指令になる。topic 経由の構成だと起きやすい。

**5. 「向きの表し方で遠回り」問題**
quaternion は同じ向きに q と −q の 2 表現がある。符号が入れ替わると Ruckig は「ものすごく違う向きに回れ」と解釈して大きく振る。

**まとめ**：1・2・3 は全部「50ms ごとに何かをリセットしている」という同じ性質で、区間の境目で折れる。だからガタつきが約 20Hz 周期で出るはず。それを確かめるのが「速度グラフに 50ms ごとの縦線を引いて折れが一致するか見る」というログの狙い。シムで滑らかだったのは「実測＝指令」が成り立つので 2 と 3 が起きようがないから。

---

<a id="sec6"></a>
## 6. 各原因への対策と確かめ方（噛み砕き版）

**1. 止まる前提の道案内 → 通過速度も一緒に伝える**
対策：target の位置だけでなく「前回 target との差÷50ms」を target_velocity として渡す。さらに良いのは位置指令をやめて速度指令にすること（目標との距離に応じた速度＋target の動きから計算した速度）。
確かめ方：Ruckig が作った各軌道の所要時間を出力。50ms より短い軌道ばかりなら「毎回止まっていた」証拠。対策後に速度グラフの 50ms ごとの谷が消える。

**2. 遅れた地図 → 自分が出した指令を基準にする**
対策：「今の位置＋delta」の基準を実測ではなく前回の指令位置にする。ただし学習データの定義（実測基準）との整合を確認する（→ [第 7 節](#sec7)）。
確かめ方：BC 生 target を時系列で並べて 50ms ごとに前後に振れていないか見る。決定的なのは、滑らかな正弦波を「実測＋delta」方式で流す実験。ガタつけば基準の問題、滑らかなら BC 出力自体の問題。

**3. 毎回やり直し → Ruckig の記憶を上書きしない**
対策：current state は前周期に Ruckig 自身が出した値を引き継ぐ（pass_to_input）。実測は監視専用（指令と実測が離れすぎたら止める安全判定）。最初の 1 回だけは Franka が「最後に受理した指令位置」（`O_T_EE_c`）から始める。
確かめ方：Ruckig に入る「今の位置」と前周期の「次の位置」を並べてプロットし、常に一致するか。

**4. 20 回しか送っていない → Ruckig を 1kHz 側に置く**
対策：Ruckig の計算を Franka に指令を書く 1kHz ループの中に置く。推論スレッドは「最新の target」を共有メモリに置くだけ。topic 経由ではなくロックの少ない方法でつなぐ。推論の待ち時間が 1kHz ループに入らないことが重要。
確かめ方：Franka に書いた指令位置と Franka が受理した指令位置（`O_T_EE_c`）を 1ms ごとに並べる。50ms ごとにしか変わっていなければ ZOH。

**5. 向きの遠回り → 符号を揃えてから渡す**
対策：新しい quaternion と前回の内積が負なら符号を反転。Ruckig 出力も毎回正規化。rotation vector なら ±180 度付近で飛ぶので前回姿勢からの相対回転で扱う。
確かめ方：姿勢固定で位置だけ動かす実験と、位置固定で姿勢だけ動かす実験を分ける。

**共通の確かめ方**：対策の前後で同じ 3 つの数字（速度の周波数分析での 20Hz 成分、jerk の大きさ、制御ループ周期のぶれ）を出す。目視ではなく数字で判断。

**順番**：まず 4（送信経路）と 3（引き継ぎ）を確認。次に 1（一番効く可能性が高い）。その後 2 と 5 を切り分け実験で確認。

---

<a id="sec7"></a>
## 7. IsaacLab IK-Rel タスクの action 定義（ソース確認済み）

IsaacLab main ブランチのソースを直接取得して確認（大輔さんの環境のバージョンとは細部が違う可能性があるため、手元でも同じ箇所を確認すること）。

**タスク設定**（`stack_ik_rel_env_cfg.py` / `stack_ik_rel_visuomotor_env_cfg.py`）
`DifferentialInverseKinematicsActionCfg(body_name="panda_hand", command_type="pose", use_relative_mode=True, ik_method="dls", scale=0.5, body_offset=[0,0,0.107])`。ロボットは `FRANKA_PANDA_HIGH_PD_CFG`（IK 追従のため硬めの PD）。

**周期**（`stack_env_cfg.py`）
`sim.dt = 0.01`、`decimation = 5` → ポリシー周期 20Hz。実機推論周期と一致。

**delta の基準は実測**（`task_space_actions.py` の `process_actions`）
ポリシーステップごとに `processed = raw × 0.5` → `_compute_frame_pose()` でシムの `body_pos_w / body_quat_w`（実測）を base 座標系に変換しオフセットを足す → `set_command` → `apply_delta_pose(ee_curr, delta)`。

**delta の意味**（`math.py` の `apply_delta_pose`）
位置：`target_pos = ee_curr + delta[0:3]`（base 座標系、m）
姿勢：`delta[3:6]` は axis-angle、`target_rot = quat_mul(delta_quat, ee_quat)`。**左から掛けている＝base 座標系での回転**（EE 座標系ではない）。ソースに「順序が正しいか要確認」という TODO コメントあり。

**内側のループ**（`apply_actions`）
物理ステップ（100Hz）ごとに実測姿勢を取り直し、固定 target との誤差を計算し、`joint_pos_des = 実測 joint_pos + J⁺(DLS)·誤差` を 1 回だけ線形化して関節 PD に渡す。50ms の間、実測から目標へ引き寄せる P 制御が 5 回走る構造。

**テレオペ記録**
`Se3RelRetargeter`（Vision Pro 手追跡）が `delta_pos_scale_factor=10, delta_rot_scale_factor=10` で delta を生成し、`env.step` の action として記録され、環境内で ×0.5 される。HDF5 の `actions` は ×0.5 前の値のはず（要確認）。

**実機への解釈（1 行）**

```text
50ms あたりの変位 = 0.5 × action（位置[m] は base 座標系、姿勢は axis-angle[rad] を base 座標系で左掛け）
速度に直すと v = 0.5 × action / 0.05 = 10 × action [m/s, rad/s]
TCP = panda_hand 原点から +0.107 m（z 方向）
```

**含意**

- 学習データの定義は「実測＋delta」なので、実機で位置目標として忠実に再現すると [第 5 節](#sec5) の 2 番（遅れた実測へのフィードバック）を必ず抱える。シムでは PD が引き寄せるだけなので問題化しない。
- 速度指令として解釈すれば基準問題が消え、値の意味も学習時と同じ。Franka の `CartesianVelocities` インターフェース、または Ruckig velocity interface → `CartesianPose` の構成に自然にはまる。
- 姿勢 delta が base 座標系である点は、実機で EE 座標系で回している場合に見落としやすい。
- 実機の TCP 定義が `0.107m` と一致しているか確認。

**Claude Code への調査指示（当初案。上記で回答済みだが fork 側の差分確認用に残す）**

```text
Isaac Lab の Isaac-Stack-Cube-Franka-IK-Rel-Visuomotor を継承した本タスクについて、
学習データの action（delta pose）の定義を確定したい。コードは一切変更せず、
以下を該当ファイル・行番号付きで報告してください。推測は「推測」と明記すること。

1. 本タスクの環境cfg（継承元含む）で、actions に設定されている
   DifferentialInverseKinematicsActionCfg の全パラメータ
2. Isaac Lab 側の DifferentialInverseKinematicsAction の実装を読み、
   relative mode のときの目標姿勢の計算式を書き出す（現在姿勢の取得元、delta のフレーム、回転の合成順）
3. sim.dt と decimation から policy 周期を計算する
4. 学習データ（HDF5）の actions が何から生成されたか（テレオペ delta か状態差分か）、clip / scale / 正規化の適用箇所
5. actions の姿勢成分の表現と変換方法
6. 以上をもとに、実機で同じ意味になる解釈を1行で書く
```

---

<a id="sec8"></a>
## 8. Ruckig を自前実装に取り入れるべきか

**入れる価値はある。ただし使い方が変わる。**

今の使い方は「点から点への軌道計画器」で、これが「止まる前提」を生んでいる。20Hz で目標がどんどん動く用途では、Ruckig は **速度インターフェースの「なめらか化＋制限保証」部品** として使うのが正解。入力は「今欲しい速度」、出力は「加速度・jerk 上限を守りながらその速度に近づける 1ms ごとの位置・速度」。

代替案：同じことは「速度に対する加速度制限付きスルーレート制限＋一次 LPF」でも実現でき、コードは数十行。違いは Ruckig が jerk 上限も数学的に保証する点と、Franka の反射停止（加速度・jerk 違反）を確実に避けられる点。community 版は MIT で商用の障害なし。

Ruckig が不要なケース：Franka の `CartesianVelocities` に `limitRate` 込みで速度を渡し、前段の簡易フィルタで十分な場合。簡易版で動かして jerk 違反で止まるなら Ruckig に置き換える順序でも可。

---

<a id="sec9"></a>
## 9. 他のロボットアームで使われている滑らか・繊細な制御方法

「滑らか」と「繊細（接触に強い）」は別の仕組みで実現されている。

**1. 低レベル制御器：位置制御ではなくインピーダンス／OSC**
研究系マニピュレーションで最も多いのはトルク制御ベースの Cartesian impedance control または Operational Space Control（Khatib）。目標姿勢との誤差にバネ・ダンパで力を返し、目標が多少ガタついても物理的なコンプライアンスが高周波成分を吸収する。Diffusion Policy の実機は Polymetis（1kHz Cartesian impedance）、Deoxys（500Hz OSC）は robomimic 系で標準的、ALOHA は関節 PD。いずれも「モデルが 10〜50Hz で粗い目標を出し、1kHz 級の柔らかい制御器が追う」。
Franka の CartesianPose インターフェースは内部の硬い位置制御器なので、目標の段差がそのまま出やすく接触にも弱い。

**2. 軌道層：jerk 制限付き OTG またはスプライン**
Ruckig / Reflexxes、5 次スプライン補間。産業用ロボットの内部はこれで、位置制御でも滑らかなのはコントローラ内でこの層が高周期で走っているから。学習ポリシーの実機では action chunk＋補間＋境界処理がこの層に相当。

**3. 接触を伴う繊細作業：力制御**
挿入・研磨・組立では hybrid force/position control か admittance control。「押し込み量」ではなく「押し付け力」を制御するので位置誤差が数 mm あっても壊れない。

**4. RL 系ヒューマノイド・脚ロボット：PD＋action 平滑化**
ポリシーは 50Hz 程度で関節目標を出し、1kHz 級の PD が追う。滑らかさは PD のダンピングと、学習時の action rate penalty、推論時の action low-pass filter で作る。BC にも転用可能。

**5. 全身 QP IK（テレオペ・リターゲット）**
GR00T のテレオペや IsaacLab の新しい経路は Pink（Pinocchio ベースの重み付き QP IK）。滑らかさよりは「安全な解の選択」の層。

**含意**：現状は「硬い位置制御＋点間計画の Ruckig」で、事例とは逆の組み合わせ。段階的に近づけるなら、まず速度インターフェース化（軌道層を正す）、次に低レベル制御を Cartesian impedance へ（繊細さ・接触耐性）。impedance 化は Franka 内部 IK の問題も同時に消える。ただし安全設計の責任が自分側に移る。

---

<a id="sec10"></a>
## 10. Cartesian impedance とは

「手先を、目標位置とバネとダンパでつないだように振る舞わせる」制御。目標と実際の手先の差に比例した力（バネ）と、速度に比例した抵抗（ダンパ）を手先に発生させる。押されればたわみ、離せば戻る。壁に当たっても力はバネの伸び分しか出ない。

```text
F = K·(x_target − x) − D·ẋ          手先に出したい力
τ = Jᵀ·F + 重力・コリオリ補償        関節トルクに変換
```

`J` はヤコビアン。転置を掛けると「手先にこの力を出すには各関節にどのトルクが要るか」が一発で出る。**IK を解いていない**のがポイント。

**構造**

```text
推論          → 手先の目標位置 x_target（20Hz）
自前制御器    → ロボットの現在状態（q, dq, J）を毎 1ms 読み、上の式でトルク τ を計算
ロボットへ    → 「関節角度」ではなく「関節トルク」を 1kHz で渡す
```

「次の関節角度を計算して渡す」のは位置制御。インピーダンスは「角度」を渡さず「トルク」を渡すので、どの角度に落ち着くかは物理と力の釣り合いで決まる。Franka の場合、重力補償は Franka 側が自動で足すので、自前で計算するのはコリオリ補償と上の式だけ。libfranka の `cartesian_impedance_control` サンプルがほぼそのまま。

**王道かどうか**

- 研究系の学習ポリシー実機では王道。
- 産業用では王道ではない。位置制御＋コントローラ内補間が主流で、トルクインターフェースを開放しているロボット自体が少ない（Franka、KUKA iiwa、一部の協働ロボット）。
- 「自前 IK で関節角度を渡す」はその中間で、ROS2＋ros2_control では一般的。追従が硬いので Ruckig 等の軌道層が必須。

**注意点**

- バネが柔らかいほど滑らかで安全だが目標に届かない（定常誤差）。硬くすると位置制御に近づき振動しやすい。ゲイン調整が必ず要る。
- 冗長自由度（7 軸目）は上の式では決まらないので null-space 項を足す。
- トルク制御は安全の責任が自分側に来る（→ [第 12 節](#sec12)）。

---

<a id="sec11"></a>
## 11. 分岐点：速度管理化 vs インピーダンス化

二択に見えるが、**層が違うので排他ではない**。

```text
軌道層（目標の解釈）    ：位置目標のまま ／ 速度指令として解釈（Ruckig velocity）
低レベル制御器          ：Franka 内部の位置制御（CartesianPose）／ 自前 Cartesian impedance（トルク）
```

インピーダンスにしても、20Hz で目標が「止まる前提」で飛んでくる構造や delta 基準の問題は残る。バネが吸収する分マシに見えるが原因は消えていない。

### A. 速度管理化のみ（位置制御は維持）

メリット
- 変更が小さい。制御インターフェースを変えないので安全設計・franka_ros2 構成・グリッパ連携がそのまま
- 学習定義（50ms あたり変位＝速度）と一致し、基準問題と蓄積問題が構造的に消える
- Ruckig で jerk 保証が残り、Franka の反射停止を避けやすい
- 数日で実機検証まで行ける規模

デメリット
- 硬い位置制御のままなので、接触作業では力が逃げず、物体を弾いたり反射停止したりしやすい
- Franka 内部 IK の挙動（肘・特異点）は制御できない
- 滑らかさの上限は Ruckig の加速度・jerk 制限で決まり、追従遅れとのトレードオフ

### B. Cartesian impedance 化（＋速度管理も入れる）

メリット
- 目標のガタつきをバネ・ダンパが物理的に吸収し、動きが柔らかい
- 接触に強く繊細な作業に向く。研究系の実機事例と同じ構成
- 自前でトルクを出すので Franka 内部 IK・肘問題が消え、null-space で 7 軸目を制御できる
- 力制御や hybrid 制御へ拡張しやすい

デメリット
- 変更が大きい。トルク制御ノード、ゲイン調整、null-space、安全制限を自前で持つ
- 定常追従誤差があり、精密な位置決めで剛性を上げる必要 → 振動リスクとのせめぎ合い
- 商用納品では安全検証項目が増える
- 軌道層の修正（速度解釈）は別途必要。B だけでは原因 1・2 は残る

### 判断

**まず A。** B をやる場合も A は必要なので無駄にならない。今のガタつきの原因はほぼ軌道層にあり A で解決する可能性が高い。A で残った問題が B の導入理由になるか判断できる。

B へ進む基準：A の実機ログで「動作中は滑らかだが接触の瞬間に反射停止や弾きが起きる」「Franka 内部 IK 出力（q_d）に折れが残る」のどちらかが確認されたとき。

---

<a id="sec12"></a>
## 12. トルク制御の「安全責任」と残る危険

**「責任が自分側に来る」の意味**

位置制御では、渡すのは「次の 1ms にいてほしい位置」だけで、Franka は制限を超える指令を拒否して止める（反射停止）。**コードがおかしくてもロボットが出せる運動エネルギーは Franka が決めた枠に収まる**。トルク制御では渡すのは「各関節に今かける力」そのもので、Franka は「その力を出すとどうなるか」を判断しない。バネの向きを逆に書けば目標から離れる方向に加速し続ける。**制御器のバグが直接エネルギーになる経路が開く**。

**具体的な危険**

- 符号ミス・単位ミス：正帰還になり鞭のように振る
- 発振：剛性上げすぎ、ダンピング不足、制御周期の jitter で腕全体が振動
- 目標の飛び：目標が 1m 先に飛ぶと剛性 × 1m の巨大な力。位置制御なら Franka が拒否する場面
- 特異点付近：小さな手先力が大きな関節トルクに化ける
- 重力・コリオリ補償の誤り：静止しているはずの腕が沈む・浮く
- NaN・未初期化値：1 周期でもそのままトルクになる
- 接触時の想定外の力：速度が乗った状態でぶつかればダンパ力＋慣性で人に当たる力は出る

Franka の安全機構（関節トルク上限、トルク変化率制限、衝突検知の反射停止、関節・速度リミット、非常停止）は残るので無制限に暴走はしない。ただしこれらは「異常が起きた後に止める」網。

**対策をしても危険は残るか**

残る。正しく設計すれば「通常運転中の危険度」は位置制御と同等以下にできるが、「バグが物理的な力に直結する経路」自体は消えない。

多層の対策：
- 目標のクランプ：現在位置から一定距離以上の目標は受け付けない
- トルク飽和：Franka の上限よりさらに低い自前上限
- 誤差・速度の監視：閾値超過で即停止
- ウォッチドッグ：目標更新が途切れたら安全に減速停止
- ゲインの上限管理：低剛性・低速から段階的に上げる
- 検証：シム / dry-run で符号・単位・座標系を数値検証してから実機
- 運用：初期評価は人が届かない範囲・低速モード・非常停止を持った人が付く

**商用納品の観点**：「なぜこの動きが安全と言えるか」を説明する責任が Franka の仕様書ではなく自分の設計文書になる。協働運用ならリスクアセスメント（ISO 10218 / ISO/TS 15066 の考え方）で上記対策を根拠として示す必要がある。

---

<a id="sec13"></a>
## 13. franka_ros2 リポジトリで提供されているもの（実調査）

リポジトリを直接読んだ結果（main は v3.5.3、2026-09-01。v3.0.0 は 2025-09-18）。

### 13.1 速度管理化に必要なもの：v3.0.0 で揃っている

- `FrankaCartesianVelocityInterface`（franka_semantic_components）が v3.0.0 に存在。`setCommand(linear, angular)` で 6 次元速度を渡す。elbow 指令の有無を選べる。`cartesian_velocity_example_controller` がサンプル。
- 中身は libfranka の `CartesianVelocities`（`O_dP_EE`）へそのまま渡す。
- **重要な発見**：`franka_hardware/robot.cpp` で、速度・位置・Cartesian pose すべての **rate limiter と low-pass filter がハードコードで無効**（`{false}`、cutoff 100Hz は定義だけ）。v3.0.0 も main も同じで、パラメータで有効化できない。**CartesianPose に書いた値はフィルタも制限もなく libfranka へ直行**。連続性の担保は 100% 自分側の責任。有効化したければソース書き換え（source build なので可能）。
- 軌道生成層はリポジトリにない。`motion_generator.cpp` は関節空間の点間移動（move_to_start 用）だけ。**インターフェースは用意されているが、なめらか化は自前で書く**。

### 13.2 インピーダンス制御化：v3.0.0 には Cartesian impedance がない

v3.0.0 にあるもの：
- `joint_impedance_example_controller`：関節空間の PD トルク制御
- `joint_impedance_with_ik_example_controller`：MoveIt の IK service を呼ぶ構成（1kHz 用ではない）
- `gravity_compensation_example_controller`
- `FrankaRobotModel`：ヤコビアン・コリオリ・質量行列を取れる semantic component。**Cartesian impedance を自作する材料はここにある**

Cartesian impedance のサンプルは **v3.2.2（2026-03-03）で追加**。main 版の中身：`τ = Jᵀ(−K·error − D·J·dq) + null-space 項 + coriolis`、目標は `PoseStamped` topic → RealtimeBuffer、剛性はサービスで変更可能、quaternion の半球チェックと目標への slerp フィルタ入り。ただし「例」であって製品用ではなく、目標のクランプ・ウォッチドッグ・誤差超過停止はない。

使うには v3.2.2 以降へ上げる必要があり、main は libfranka ≥ 0.20.4 を要求。v3.0.0 → v3.5.x はコントローラ API や franka_description の変更を伴う。代替はそのファイル 1 本を v3.0.0 に移植（依存は `FrankaRobotModel` と `realtime_tools` だけ）。

### 13.3 第 3 の選択肢

`franka_param_service_server` に `~/set_cartesian_stiffness` サービスがあり、libfranka の `setCartesianImpedance` を呼ぶ。**CartesianPose インターフェースのまま Franka 内部制御器の Cartesian 剛性を下げる**機能。トルク制御に移行せずにある程度のコンプライアンスが得られ、安全責任は Franka 側に残る。ただし内部制御器の挙動は公開情報が薄いので実機で数値確認が必要。

### 13.4 まとめ

```text
速度管理化           v3.0.0 で可能。Ruckig velocity は自前実装。rate limiter は自分で有効化
内部剛性を下げる     v3.0.0 で可能（サービス 1 つ）。効果は実機で要確認
Cartesian impedance  v3.2.2+ が必要、または例を 1 本移植。安全層は自作
```

v4 プロンプトの 4.8 と 6 の「libfranka の rate limiter との整合」は「franka_ros2 では無効。有効化するか Ruckig で担保するかを決める」に読み替える。

---

<a id="sec14"></a>
## 14. elbow 指令とは

Franka は 7 関節なので、手先の位置と向きを決めても「肘の位置」は決まらない。肩と手首を結ぶ軸を中心に肘を振っても手先は同じ場所にいられる。この余った 1 自由度を Franka は「elbow」と呼ぶ。

```text
elbow[0] = 関節 3（q3）の角度        肘をどこに振っているか
elbow[1] = 関節 4（q4）の符号（±1）   肘が曲がる向き
```

Cartesian pose / velocity インターフェースでは elbow を「一緒に指令する」か「Franka に任せる」かを選べる。franka_ros2 の `FrankaCartesianVelocityInterface(command_elbow_activate)` の引数がそれで、`cartesian_elbow_example_controller` がサンプル。

**Franka に任せると**：内部 IK が「今の肘姿勢をなるべく保つ」ように解くが、アルゴリズムは非公開。特異点や可動域端に近づいたとき肘の解が動いて関節側だけ振れる可能性がある（v4 プロンプト 4.8 で `O_T_EE_c` は滑らかなのに `q_d` が折れていないかを見させている理由）。

**指令すると**：冗長自由度の暴れを消せる。ただし elbow にも速度・加速度・jerk 制限があり、不連続に指令すれば反射停止。BC モデルは肘の情報を出さないので、指令するなら「初期姿勢の elbow を保持」か「ゆっくり好みの角度へ戻す」程度が現実的。

**位置づけ**：学習側（IsaacLab）の DLS-IK は前回の関節角度から最小変化で解くので肘は勝手に大きく動かない。実機の Franka 内部 IK も近い挙動のはずだが未確認。まず 4.8 のログで `elbow_c` が動いているか見て、動いていれば elbow 指令を有効にして初期値で固定する順序で十分。今のガタつきの主原因である可能性は低い。

---

<a id="sec15"></a>
## 15. 結論と次の行動

**原因の見立て（実験で裏付けるまでは「見立て」）**

1. Ruckig の target_velocity=0 による stop-and-go
2. delta action を実測基準の位置目標として扱っていることによる閉ループ発振
3. Ruckig current state の実測上書き、または 20Hz ZOH での Franka 送信
4. 姿勢表現の不連続、Franka 内部 IK / elbow（可能性は低い）

**方針**

- 軌道層：delta を **Cartesian 速度指令**（`10 × action`、base 座標系、axis-angle 左掛け、TCP +0.107m）として解釈し、Ruckig velocity interface（1kHz、controller 内）でつなぐ。学習定義と一致し、基準問題・蓄積問題が構造的に消える。
- 低レベル制御：当面は CartesianPose / CartesianVelocities のまま。`set_cartesian_stiffness` で内部剛性を下げる実験は低コストなので併用可。
- Cartesian impedance 化は、A の実機ログで接触時の弾き・反射停止や `q_d` の折れが残った場合に、安全設計の工数込みで判断。

**次の行動**

1. `franka_bc_ruckig_jitter_prompt_v4.md` の Phase 1（読み取り専用調査）を Claude Code で実行。特に項目 3・5・13・32（delta の解釈と学習定義の差分、command 経路、target_velocity、fork の override）。
2. 切り分け実験（オフライン再生、合成正弦波、HDF5 トレース再生）で原因を数値で確定。
3. 速度解釈＋Ruckig velocity interface へ修正。rate limiter は自前有効化か Ruckig で担保かを決める。
4. 修正前後を同一指標（速度 PSD の 20Hz ピーク、jerk RMS、周期 jitter p99）で比較。dry-run → 承認後に実機。
5. 残った問題に応じて内部剛性調整 → Cartesian impedance の順に検討。

---

<a id="sec16"></a>
## 16. cartesian_impedance_example_controller には何を入力するか

franka_ros2 main / v3.2.2+ の `cartesian_impedance_example_controller` のソースを確認した結果。

**入力は3つ**

1. **目標姿勢（平衡点）**：`~/equilibrium_pose` topic（`geometry_msgs/PoseStamped`）
   - 座標系は base（`fr3_link0`）。位置は m、姿勢は単位 quaternion。norm ≈ 0 の quaternion は破棄（NaN 対策）。
   - 周期は任意。非RTスレッドで受けて `RealtimeBuffer` に入れ、1kHz の `update()` が読む。
   - 内部で `position_d_` へ一次フィルタ、`orientation_d_` へ半球チェック付き slerp。`filter_params_ = 0.005` × 1kHz で時定数 ≈ 0.2 s（かなり重い平滑化）。
2. **ゲイン**：パラメータまたは `~/set_cartesian_stiffness` サービス
   - `translational_stiffness`（既定 150 N/m）、`rotational_stiffness`（10 Nm/rad）、`nullspace_stiffness`（20）
   - ダンピングは自動で `2√K`（臨界減衰）。実行中の変更も同じフィルタで滑らかに切り替わる。
3. **null-space の目標関節角**：activate 時の `q` に固定。外部入力なし。

**そのままでは使えない点**

- `update()` が毎周期 `updateMotionTarget()` を呼び、**位置目標を円運動のデモ軌道で上書きしている**（姿勢だけ topic の値を使う）。topic から位置を与えるにはこの呼び出しを削除する必要がある。
- 目標のクランプ・ウォッチドッグ・誤差超過停止はない。安全層は自作。

**BC パイプラインから何を入れるか**

生の 20Hz BC target を直接入れるのは勧めない。内部フィルタ（τ ≈ 0.2 s）で追従が遅れて BC の閉ループが崩れ、delta の積み上げ先が測定姿勢になる問題もこの controller では解決されない。

推奨は、前段の速度解釈＋Ruckig velocity interface で作った **1kHz の連続な姿勢を equilibrium pose として与える** 構成：

```text
BC（20Hz）→ delta を速度解釈 → Ruckig velocity（1kHz）→ 積分した pose
    → equilibrium_pose（controller 内の RealtimeBuffer に直接）
    → Jᵀ(−K·error − D·ẋ) + null-space + coriolis → トルク
```

内部フィルタは不要なので `filter_params_` を 1.0 近くにするか、Ruckig を controller 内に取り込んで topic を経由しない。1kHz を topic で送るのは DDS jitter の問題があるので後者が本筋。

「入れるべきもの」は Ruckig で滑らかにした後の目標姿勢であって BC の出力そのものではない。インピーダンスは「目標が多少荒れても壊れない」保険であって、目標を作る側の責任を肩代わりしない。

---

<a id="sec17"></a>
## 17. Isaac Sim はインピーダンス制御か

**正確には関節空間のインピーダンス（PD）制御。Cartesian impedance ではない。** IsaacLab の `franka.py` で確認。

**仕組み**：`ImplicitActuatorCfg` で定義され、PhysX の articulation drive が関節ごとに

```text
τ = K·(q_target − q) + D·(q̇_target − q̇)     （effort_limit でクランプ）
```

を物理ステップごとに暗黙的に解く。IK-Rel action は `q_target` を作るだけで、あとは各関節がバネ・ダンパで引っ張られる。

**ゲイン**

```text
FRANKA_PANDA_CFG         K = 80,  D = 4      （デフォルト）
FRANKA_PANDA_HIGH_PD_CFG K = 400, D = 80     ← Stack Cube IK-Rel が使うもの
                         disable_gravity = True
```

単位は Nm/rad。重要な事実が2つ：

1. **実機 Franka より大幅に柔らかい。** 実機の内部関節インピーダンス（`setJointImpedance` 既定値）は 3000〜2000 Nm/rad 程度で、シムの HIGH_PD でも 5〜7 倍硬い。Cartesian pose モードの内部剛性は既定で並進 3000 N/m。シムで粗い 20Hz 目標が平気なのは、関節バネが柔らかくて段差を吸収しているから（第4節の「PD が低域フィルタとして働く」の実体）。
2. **HIGH_PD は重力を切っている**（`disable_gravity=True`）。K=400 だと重力で腕が垂れるので、重力自体を無効化して硬さを補っている。学習データは「重力のない柔らかい腕」で集められている。実機は重力補償が入るので落下はしないが、追従の挙動はシムと同じにはならない。

**含意**

- シムの挙動に実機を近づける方向は2つ。(a) 実機の内部剛性を下げる（`set_cartesian_stiffness` / `set_joint_stiffness` サービス。トルク制御不要）。(b) 自前 Cartesian impedance。シムが joint impedance なので素直に対応するのは (a) の関節剛性側だが、Cartesian pose モードでは Cartesian 剛性の方が効く。
- 剛性を下げると定常誤差が増え、積み上げの最終位置精度が落ちる。シムは重力なしでその誤差が出ていないので、実機でどこまで下げられるかは実測で決める。
- 目標の作り方（速度解釈＋Ruckig）を先に直す順序は変わらない。剛性を下げるのは保険。
- fork で `robot` cfg（K・D・disable_gravity）を上書きしていないかを Phase 1 の確認項目に追加する。

---

<a id="sec18"></a>
## 18. Cartesian impedance と関節空間インピーダンス（PD）の違い

**バネをどこに付けるか、の違い。**

**関節空間のインピーダンス（PD）**
7つの関節それぞれにバネとダンパを付ける。「関節1は 30 度が目標、今 28 度だから 2 度分の力で戻す」を関節ごとに独立にやる。

```text
各関節：τ_i = K·(目標角_i − 今の角_i) − D·角速度_i
```

手先の目標位置は直接見ていない。IK で目標角に変換してから、あとは関節が引っ張られる。IsaacLab の PhysX ドライブ、Franka の JointPositions モードの内部制御器、ALOHA の関節 PD がこれ。

**Cartesian impedance**
バネを手先に付ける。「手先は目標より 5mm 左、だから右向きに 5mm × K の力を出す」と手先で考え、その力を出すための各関節の負担をヤコビアンで配分する。

```text
手先：F = K·(目標位置 − 今の位置) − D·手先速度
関節：τ = Jᵀ·F
```

IK は使わない。目標角という概念がなく「手先がここに来るように力をかけ続ける」だけ。

**たとえ**：腕を肩・肘・手首の3関節と思って、手先を壁に押し付けたとき、

- 関節バネ：肩・肘・手首のバネがそれぞれ勝手にたわむ。手先がどの方向にどれだけ柔らかいかは腕の姿勢で変わる。「横は硬く、押し込み方向は柔らかく」のような指定はできない。
- 手先バネ：「押し込み方向は柔らかく、横はズレないよう硬く」と手先座標で直接指定できる。姿勢が変わっても手先の手応えは同じ。挿入や組立で欲しいのはこちら。

**余った関節**：Franka は 7 関節なので手先を固定しても肘は動ける。関節バネなら 7 個全部に目標角があるので肘も決まる。手先バネは手先しか見ないので肘がフラつく。だから Cartesian impedance には必ず null-space 項（肘を好みの姿勢に弱く引くバネ）を足す。

```text
                  関節バネ（joint impedance / PD）    手先バネ（Cartesian impedance）
バネの場所        各関節                              手先
必要な変換        IK（位置→角度）                     ヤコビアン転置（力→トルク）
柔らかさの指定    関節ごと                            手先の方向ごと
姿勢による変化    手先の手応えが変わる                手先の手応えは一定
肘（冗長性）      自動的に決まる                      null-space 項が必要
特異点            IK が困る                           ヤコビアンが小さくなり力が出にくい
実装の難しさ      低い                                中（座標系・quaternion・null-space）
代表例            IsaacLab、Franka JointPositions     libfranka サンプル、Polymetis、Deoxys
```

**大輔さんの構成で見ると**：学習側（IsaacLab）は関節バネ、実機の Cartesian pose モードは内部が「手先バネ寄り」の Franka 内部インピーダンス。つまり今すでにシムと実機で「バネの場所」が違う。ただしこの違いが問題になるのは接触したときで、空中を動いている間は目標の作り方（速度解釈）の方がはるかに効く。「バネの場所を揃える」のは、接触時の挙動の差が納品要件に効いてきたときに考える話。
