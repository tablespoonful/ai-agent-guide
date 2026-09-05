# Franka 20Hz BC + Ruckig ガタつき調査・改善プロンプト（v4：Cartesian pose 渡し構成／IsaacLab action 定義確定版）

以下のFranka実機制御コードを調査・修正してください。

## 0. 進め方（必ずこの順序で。Phaseを飛ばさない）

```text
Phase 1  読み取り専用調査   コードを一切変更せず「確認項目」に全て回答し報告する
Phase 2  計測・切り分け     ログ追加＋オフライン再生／合成target実験で原因を特定する
Phase 3  修正               特定した原因に対する最小限の修正を行う
Phase 4  検証               dry-run → 実機（実機投入はユーザー承認後）
```

- Phase 1 の報告が終わるまで修正を始めないこと。
- 原因を「推定」で断定しないこと。ログ・実験で裏付けが取れたものだけを原因と呼ぶ。
- 場当たり的にフィルタを強くして症状を隠さないこと。まず原因を切り分けてから修正する。
- 実機を動かすコマンドはユーザーの明示的な承認を得てから実行する。

## 1. 背景・前提

- Franka FR3 を BC（Behavior Cloning）モデルの推論結果で制御している
- BC推論周期は約20Hz（50msごと）
- 最大TCP並進速度は250mm/s程度
- 実機は動作するが、軌道がガタガタする／カクつく
- 軌道生成には Ruckig をすでに使用している
- **制御方式：Cartesian pose をそのまま Franka に渡す（Franka の CartesianPose インターフェース）。IK は Franka 内部で行われ、自前の IK は持たない。MoveIt は使っていない。**
- スタック：ROS2 Jazzy / franka_ros2 v3.0.0（source build）/ PREEMPT_RT ホスト / アーム制御ノードとグリッパ制御ノードは分離
- したがって「補間器がないこと」を前提にせず、Ruckigの使い方・target更新方法・Cartesian command の生成経路・Franka内部IKの挙動を優先的に調査すること

### 1.1 学習側（IsaacLab）の action 定義（ソース確認済み。前提として扱う）

学習タスクは IsaacLab の `Isaac-Stack-Cube-Franka-IK-Rel-Visuomotor` を継承している。IsaacLab main の該当ソースを読んだ結果、action の意味は以下の通り。**ユーザーの fork / IsaacLab バージョンで override されていないか、同じ箇所を必ず再確認し、差分があれば報告すること。**

```text
環境cfg（stack_ik_rel_env_cfg.py / stack_ik_rel_visuomotor_env_cfg.py）
  DifferentialInverseKinematicsActionCfg(
    body_name="panda_hand", command_type="pose", use_relative_mode=True,
    ik_method="dls", scale=0.5, body_offset=pos[0,0,0.107])
  robot = FRANKA_PANDA_HIGH_PD_CFG（IK追従用の硬いPD）

周期（stack_env_cfg.py）
  sim.dt = 0.01（100Hz）、decimation = 5 → ポリシー周期 20Hz

process_actions（ポリシーステップごと、20Hz）
  processed = raw_action × 0.5（clip はデフォルト None）
  ee_curr   = シム実測の body_pos_w / body_quat_w を base 座標系へ変換 ＋ offset
  target    = apply_delta_pose(ee_curr, processed)
              位置: target_pos = ee_curr_pos + delta[0:3]          （base座標系, m）
              姿勢: delta[3:6] は axis-angle, target_rot = quat_mul(delta_quat, ee_quat)
                    ＝ base座標系での回転（左掛け）。EE座標系ではない

apply_actions（物理ステップごと、100Hz、ポリシーステップあたり5回）
  実測姿勢を取り直し → 固定 target との誤差 → joint_pos_des = 実測 joint_pos + J⁺(DLS)·誤差
  → 関節 PD へ。つまり 50ms の間「実測から目標へ引き寄せるP制御」が5回走る

テレオペ記録
  Se3RelRetargeter（delta_pos_scale_factor=10, delta_rot_scale_factor=10）の出力が
  env.step の action として記録される。HDF5 の actions は ×0.5 前の値のはず（要確認）
```

**結論（実機側で守るべき解釈）：**

```text
50ms あたりの変位 = 0.5 × action
  位置 [m]     : base 座標系
  姿勢 [rad]   : axis-angle、base 座標系で左掛け
速度に換算     : v = 0.5 × action / 0.05 = 10 × action  [m/s, rad/s]
TCP           : panda_hand 原点から +0.107 m（z方向）。実機の TCP 定義と一致させる
```

学習データでの delta 基準は「シム実測」であり、シムでは実測≒指令なので破綻しないが、実機で位置目標として忠実に再現すると 4.4 の閉ループ発振を必ず抱える。したがって実機では **delta を Cartesian 速度指令として解釈する**（4.4 参照）のが学習時の意味を保ちつつ基準問題を消す解釈である。

## 2. 目的

以下の各段階のどこで不連続・周期的な揺れ・jerk spikeが生じているかを明確にする。

```text
BC target更新（20Hz）
→ target前処理（delta→絶対pose変換、Pose Filter、姿勢表現の変換）
→ Ruckig target更新
→ Ruckig update（高周期、Cartesian空間）
→ 4x4同次変換（O_T_EE_d）の組み立て
→ ROS2 / ros2_control / libfranka の command経路
→ Franka内部IK → joint commanded state（q_d, dq_d）
→ Franka measured state（q, dq, O_T_EE）
```

## 3. 最初に確定させること：command経路の特定

Phase 1 で最初に、Ruckig出力がFrankaへ届くまでの実経路を以下のどれかに分類してください。

```text
A. libfranka を直接使用（franka::Robot::control の CartesianPose callback 内で command 生成）
B. ros2_control のカスタム controller 内で Ruckig update() →
   FrankaCartesianPoseInterface（franka_semantic_components）に書き込み
C. 別ノードから topic で pose を publish し、controller が受け取って書き込み
D. Python ラッパー（franky / panda-py 等）の Cartesian motion 経由
E. その他（具体的に記述）
```

各分類で必ず確認すること：

- **A / B 共通（libfranka CartesianPose の要件）**
  - command は毎周期（1ms）必ず書く。書かない周期があれば libfranka 側は前回値を保持し、次の更新で段差が出る
  - 初期 pose は **`O_T_EE_c`（最後に commanded された pose）** から開始しているか。`O_T_EE`（measured）から開始すると初回に段差が出る
  - Ruckig の初期 current state も同様に commanded pose 基準か
  - `franka::Robot::control(..., limit_rate, cutoff_frequency)`：rate limiter 有効か、LPF cutoff（デフォルト100Hz）
  - `franka::limitRate` が Cartesian command をクリップしていないか（クリップされていれば Ruckig の Cartesian 制限値が libfranka 制限より緩い＝設定不整合）
  - `elbow` を command しているか（`CartesianPose(pose, elbow)`）。command していない場合、肘（q3 / q4 の符号）の扱いは Franka 任せになる
  - libfranka の Cartesian motion generator 系エラー（velocity / acceleration discontinuity、elbow 関連、joint limit 関連 等）が reflex として出ていないか。エラー履歴（`robot_state.last_motion_errors` / `current_errors`）をログする
- **B（franka_ros2）**
  - controller_manager の `update_rate`（1000Hz想定）と hardware interface 周期の一致
  - controller の `update()` 内で毎回 pose interface に書いているか。`update()` が 1ms 以内で返っているか（`realtime_tools` の統計、または自前計測）
  - 姿勢の入力形式（quaternion / 4x4行列）と内部変換
- **C（topic経由）**
  - DDS 経由で 1kHz command を送ると executor / timer の jitter がそのまま制御に入る。rclpy なら 1kHz は事実上不可能。**controller 内（realtime context）で Ruckig を回す構成に変える対象**
  - controller 側で topic が来ない周期に何をしているか（前回値保持＝Zero-Order Hold → 20Hz でしか pose が更新されない典型パターン）
- **D（Python ラッパー）**
  - franky は内部で Ruckig を持つ。`move(CartesianMotion, asynchronous=True)` を 20Hz で連打すると各 motion の目標速度が 0 のため stop-and-go になる（4.3 と同じ問題）
  - Python 制御ループは torch 推論と GIL を取り合う（4.9）

## 4. 最重要確認事項

### 4.1 Ruckigのupdate周期

BC推論は20Hzのままで構わない。しかしRuckig自体を20Hzでしか更新していない、または Ruckig 出力が 20Hz でしか Franka に反映されていない場合は問題。

理想構成：

```text
BC inference / target update   20 Hz
   ↓
Ruckig target更新             （新target到着時のみ）
   ↓
Ruckig update()               1000 Hz（libfranka / controller周期と一致）
   ↓
Franka CartesianPose command  1000 Hz
```

確認する：

- BC推論周期（設定値ではなく**実測**。推論時間が50msを超えていないか、jitterはどれだけか）
- Ruckig update() の実行周期（実測）
- Franka command 書き込み周期（実測）
- それぞれが同一スレッドか別スレッドか
- Ruckigの `delta_time` が実際の制御周期と一致しているか

```cpp
Ruckig<6> ruckig{0.001}; // 1 kHz の場合（DoF数は 4.5 で確認）
```

- libfranka callback は `period` を引数で渡す（パケット欠落時は2ms以上になる）。`period` が `delta_time` より長い場合の処理（update() を複数回呼ぶ／無視）を確認する。

### 4.2 Ruckig current stateの引き継ぎ（最重要）

各update後に

```cpp
output.pass_to_input(input);
```

またはそれと等価な処理で、`current_position / current_velocity / current_acceleration` が前回Ruckig出力から次周期へ連続しているか確認する。

問題になりやすい実装：

```text
BC target更新（50msごと）
   ↓
measured pose（O_T_EE）を Ruckig current state に再設定
   ↓
trajectory再生成
```

measured pose を current state に強制代入すると、追従誤差とセンサノイズが trajectory generator へ入り、軌道が再計画され続ける。

原則：

```text
Ruckig commanded state → 次のRuckig commanded state
```

measured（`O_T_EE`, `q`）は監視・安全判定・追従誤差評価に使い、毎周期または20Hzごとに無条件でリセットしない。measured を数値微分して `current_velocity / current_acceleration` を作っている場合は、それ自体がノイズ源。

### 4.3 【最重要】Ruckig の target_velocity が 0 になっていないか

Ruckigの `target_velocity` はデフォルト0。BC targetを50msごとに更新し、毎回 `target_velocity = 0` のままだと、Ruckigは**各targetで完全停止する軌道**を生成する。

```text
target到着 → 加速 → 減速 → 停止（または停止しかけ） → 次target到着 → 加速 → ...
```

各trajectoryはjerk-limitedでも、全体としては50ms周期の加減速＝ガタつきになる。**BC targetがきれいでもRuckigが正しく連続していてもこれだけで発生する。**

確認：

- `input.target_velocity` に何を入れているか（0 / 推定値 / 前回値）
- target到着から到達までの所要時間（`trajectory.get_duration()`）が50ms未満なら、この問題が起きている可能性が高い

対策（優先順）：

1. target_velocity に**target系列から推定した速度**を与える（例：`(target_k − target_{k−1}) / Δt` を LPF 後に使用。速度上限内にクリップする）
2. **推奨**：Ruckig を **velocity control interface**（`ControlInterface::Velocity`）で使い、4.4 OK-B の速度指令（10×action）をそのまま target_velocity にする。これは 1.1 の学習定義（50ms あたり変位）と同義であり、4.3 と 4.4 を同時に解決する。Ruckig 出力の position を積分して CartesianPose に書く
3. Ruckig Pro の Tracking interface が使える場合はそれを使う（商用ライセンス要確認。community版はMIT）

### 4.4 BC delta action の基準と解釈（学習定義との整合）

1.1 の通り、学習時の action は「シム実測 EE 姿勢 ＋ 0.5×action」を 50ms 保持する定義（実質は 50ms あたりの変位＝速度）。実機側で現在どう解釈しているかを確定する。

```text
NG-1: target_k = O_T_EE（measured）_k + 0.5·a_k
      → 学習定義には忠実だが、実機では measured が commanded より遅れ＋ノイズを含むため
        閉ループの発振として 50ms 周期の揺れが出る
NG-2: scale / 座標系 / 回転の掛け順が学習定義と違う（×0.5 の二重適用や欠落、EE座標系で回転 等）
OK-A: target_k = commanded_target_{k−1} + 0.5·a_k
      → 基準問題は消えるが、追従誤差が蓄積すると target が実機から離れていく。
        commanded と measured の差を監視し、閾値超過で停止する安全判定が必須
OK-B（推奨）: v_k = 0.5·a_k / 0.05 を Cartesian 速度指令として扱い、
      Ruckig の velocity interface（または Franka CartesianVelocities）で
      加速度・jerk 制限付きに 50ms ごとの速度段差をつなぐ
      → 学習時の意味（50ms あたり変位）と一致し、基準問題も蓄積問題も構造的に消える
```

確認：

- 実機コードで delta をどの基準に足しているか、×0.5 をどこで適用しているか（HDF5 の actions が ×0.5 前なら、学習時の正規化と実機推論出力のスケールを追う）
- 位置 delta の座標系（base か EE か）、姿勢 delta の表現（axis-angle か）と掛け順（base 座標系で左掛けか）
- TCP オフセット（panda_hand +0.107m）が実機の EE フレーム定義と一致しているか
- BC が action chunk を出す場合：chunk 内の各 step をどの周期で消費しているか、新 chunk 到着時に旧 chunk と不連続に切り替えていないか（temporal ensembling の有無）、chunk の実行レートと学習時の 20Hz が一致しているか

### 4.5 【Cartesian構成特有】Ruckig の姿勢表現と DoF 構成

自前IKがない以上、Ruckigは Cartesian 空間で動いている。姿勢をどう扱っているかで不連続が入る。

確認：

- Ruckig の DoF 数と各 DoF の意味（3：位置のみ／6：位置＋rotation vector／7：位置＋quaternion成分 など）
- **quaternion の4成分をそれぞれ独立に Ruckig に入れている場合**：出力は単位quaternionにならない（要再正規化）、かつ `q` と `−q` の符号反転が入ると「遠回りの回転」を生成して大きく振れる。内積<0なら符号を揃える処理があるか確認する
- **rotation vector / Euler の場合**：±π付近の wrap-around で不連続になる。姿勢変化が小さい前提で使っているか、angle-axis の差分（前回commanded姿勢からの相対回転）で扱っているか
- 姿勢に Ruckig を使わず SLERP のみで補間している場合：位置と姿勢の時間同期が取れているか（位置は jerk-limited、姿勢は線形 → 姿勢側で角加速度 spike）
- Ruckig 出力から `O_T_EE_d`（4x4、column-major）を組み立てる際に回転行列が正規直交になっているか（非正規直交だと libfranka がエラーにする、または Franka 側で補正されて段差が出る）
- 位置 DoF と姿勢 DoF の制限値の単位（m/s と rad/s）が混在していないか。`synchronization` 設定（Time 同期なら姿勢制限が位置を遅くする可能性）

### 4.6 BC targetが50msごとに反転・振動していないか

Ruckigは制約を守るtrajectoryを生成できるが、target自体が

```text
t=0ms    x=400mm
t=50ms   x=412mm
t=100ms  x=398mm
t=150ms  x=414mm
```

のように毎回反転していれば、ロボット全体としてはガタついて見える。

ログ：`BC raw target pose / filtered target pose / target差分 / target velocity相当値 / 方向反転回数`。50ms周期で方向反転が発生していないか確認する。

### 4.7 Ruckig target更新時の挙動

新BC target到着時に、

```text
現在のRuckig commanded position / velocity / acceleration → 新targetへ再計画
```

となっているか。**`velocity = 0, acceleration = 0` へ初期化していないか**確認する。

また Ruckig の戻り値（`Result::Working / Finished / ErrorInvalidInput / ErrorTrajectoryDuration / ErrorExecutionTimeCalculation` 等）をログしているか。エラー時に何をしているか（前回出力保持／measured代入／停止）。`current_acceleration` が `max_acceleration` を超えた状態で渡すと `ErrorInvalidInput` になり、フォールバック処理が不連続を作ることがある。

### 4.8 【Cartesian構成特有】Franka 内部 IK の挙動

Cartesian command が滑らかでも、Franka 内部 IK が生成する joint 動作が滑らかとは限らない。ここは自分のコードの外側なので、**commanded Cartesian が滑らかなことを確認した上で** joint 側を見る。

確認：

- `robot_state.O_T_EE_c / O_dP_EE_c / O_ddP_EE_c`（Franka が受理した commanded Cartesian）が自分の Ruckig 出力と一致しているか。一致しなければ libfranka rate limiter / LPF が介入している
- `robot_state.q_d / dq_d / ddq_d`（Franka 内部 IK の出力）を数値微分して joint jerk を確認。**Cartesian commanded は滑らかなのに q_d に折れがある → Franka 内部 IK / 肘 / 特異点の問題**
- 動作範囲が特異点（腕の伸びきり、手首特異点）に近くないか。J7 / J5 / J6 の可動域端に近くないか
- `elbow` を command していない場合、肘の解が動いていないか（`robot_state.elbow / elbow_c / elbow_d`）
- joint 可動域・速度制限に Franka 側で当たっていないか（Cartesian は範囲内でも joint が制限に張り付くと Cartesian 追従が崩れる）
- FR3 の Cartesian 制限（並進速度・加速度・jerk）に対して Ruckig の制限値が十分下回っているか

対策候補（原因が確定した場合のみ）：

- `elbow` を明示的に command して肘を固定／緩やかに制御する
- 作業空間・初期姿勢を特異点から離す
- 上記で解決しない場合の代替案として、自前 IK（前回 commanded joint 初期値）＋ joint 空間 Ruckig ＋ JointPositions interface への切り替えを検討する（ただし v3 の範囲外。提案に留める）

### 4.9 制御スレッドをブロックするもの

- グリッパ制御ノードは分離されているが、アーム側ノードから**同期的に**gripper action / serviceを呼んでいないか（libfranka gripper API はブロッキング）
- ログ書き込み（ファイルI/O、rclcpp logger）、動的メモリ確保、mutex待ちが1kHzスレッド内にないか
- 1kHzスレッドの `SCHED_FIFO` 優先度、CPUアフィニティ、`mlockall`
- Python制御ループの場合：torch推論とGILを取り合う。推論を**別プロセス**（shared memory / ZMQ）に分離するか、制御ループをC++ controllerに移す
- 通信品質：libfranka の `control_command_success_rate`、`communication_constraints_violation`、パケット欠落ログ

## 5. Pose Filter

BC raw targetにノイズがある場合のみ使用する。

```yaml
pose_filter_tau: 0.08  # s（調整範囲 0.05〜0.10）
```

Ruckigの使い方が間違っている問題をLPFで隠さないこと。優先順位：

```text
Ruckig state continuity → target continuity（target_velocity含む）→ 姿勢表現の連続性 → 必要ならPose Filter
```

位置は一次LPF。Quaternionは各要素LPFではなくSLERP＋正規化＋符号揃え。

## 6. 制限値（初期調整用）

Cartesian 構成なので、以下の Cartesian 制限が **そのまま Ruckig の制限値** になる。加えて libfranka の rate limiter がこれより厳しくないことを確認する（厳しければ libfranka 側でクリップされ Ruckig の滑らかさが壊れる）。

```yaml
cartesian:   # Ruckig に設定（＋ libfranka rate limiter の値と比較）
  max_linear_velocity: 0.25       # m/s
  max_linear_acceleration: 0.50   # m/s^2
  max_linear_jerk: 3.0            # m/s^3
  max_angular_velocity: 0.80      # rad/s
  max_angular_acceleration: 2.0   # rad/s^2
  max_angular_jerk: 10.0          # rad/s^3
```

joint 側の制限は Franka 内部が管理する。監視用として joint velocity / acceleration / jerk をログし、Franka の joint 制限に近づいていないかを確認する。

これらはFrankaの機械的絶対上限ではなく、今回のBC実機制御用の保守的な初期値。実ログを見て調整する。

注意：制限が厳しすぎると target_velocity=0 の問題（4.3）と組み合わさって stop-and-go が顕著になる。制限値を上げて症状が変わるかも切り分けに使う。

## 7. 推奨処理フロー

```text
[Inference Node / Thread]  20 Hz
BC inference
→ delta→絶対target変換（commanded基準）
→ 姿勢の符号揃え・正規化
→ 必要ならPose Filter
→ Cartesian target + 推定target velocity
→ realtime-safe buffer（realtime_tools::RealtimeBuffer / lock-free）

[ros2_control Controller または libfranka callback]  1000 Hz（realtime）
latest target 取得（あれば target / target_velocity 更新）
→ Ruckig update()（Cartesian空間）
→ output.pass_to_input(input)
→ 姿勢を正規直交回転行列に変換 → O_T_EE_d 組み立て
→ CartesianPose interface に書き込み（毎周期必ず）
→ ログ用ring bufferに書く（I/Oはしない）

[franka_ros2 hardware interface / libfranka] 1000 Hz → Franka内部IK → FR3
```

原則：

```text
BC target更新周期 ≠ Ruckig update周期 ≠ Franka内部制御周期
```

## 8. 切り分け実験（決定的な原因特定のために必ず行う）

### 8.1 オフライン再生（ロボット不要）

実機走行時にログした BC raw target 系列を、前処理→Ruckig のパイプラインに**そのまま流し**、commanded Cartesian trajectory を再生成する。ガタつきが再現すれば原因はパイプライン内、再現しなければ command経路以降（ROS2/libfranka/Franka内部IK/実機）。

### 8.2 合成target実験

BCの代わりに、20Hz更新の**滑らかな合成target**（例：振幅50mm、周期4sの正弦波、姿勢固定）を同じ経路に流す。

```text
滑らかな合成targetでもガタつく  → パイプライン／command経路／Franka内部IKの問題（BCは無関係）
合成targetは滑らか             → BC target自体（ノイズ・反転・delta基準）の問題
```

さらに「位置のみ動かす」「姿勢のみ動かす」を分けて実行し、姿勢表現（4.5）が原因か切り分ける。

### 8.3 HDF5トレース再生

既存のsim2real手順（学習用HDF5の軌道をトレース再生し空推論を回す）を同じ経路で実行し、ガタつきの有無を記録する。

このとき HDF5 の `actions` を 1.1 の定義（0.5×action を 50ms あたり変位、base 座標系）で実機に流し、記録された EE 軌道（シム側 observation）と実機の commanded / measured 軌道を重ねてプロットする。**スケール・座標系・回転掛け順の解釈違いはここで数値として出る**（軌道の倍率ズレ、回転方向の反転 等）。

### 8.4 段階バイパス

- target_velocity に推定速度を入れる／入れない を比較
- Pose Filter ON/OFF
- libfranka rate limiter ON/OFF（ログ上でクリップの有無を確認する目的。OFF は dry-run のみ）
- 姿勢固定で位置のみ

### 8.5 定量指標

目視ではなく数値で判定する（修正前後で同じ指標を出す）：

- commanded Cartesian velocity（`O_dP_EE_c`）および joint velocity（`dq_d`）の PSD（Welch）における **20Hz（および40/60Hzの高調波）ピークの大きさ**
- Cartesian / joint の acceleration・jerk の RMS、最大値
- target更新時刻 ±5ms 窓内の jerk spike 発生率
- 各trajectoryの `get_duration()` の分布（50ms未満が多ければ 4.3）
- 制御ループ周期の jitter（平均、p99、最大）
- commanded−measured 追従誤差（`O_T_EE_c` vs `O_T_EE`、`q_d` vs `q`）のRMSと位相遅れ

## 9. ログ・可視化

以下を同一の単調増加時刻軸（`steady_clock` / `robot_state.time`）で保存（CSV / Parquet）・プロットする。1kHzスレッドではring bufferに書き、ファイルI/Oは別スレッドで行う。

BC / target：

```text
BC raw target position / orientation
filtered target position / orientation
target delta position / rotation
推定 target velocity
BC target到着時刻、推論所要時間
```

Ruckig：

```text
input.current_position / current_velocity / current_acceleration
input.target_position / target_velocity / target_acceleration
output.new_position / new_velocity / new_acceleration
Result code、trajectory duration、target更新イベント
```

Franka（commanded）：

```text
O_T_EE_d（自分が書いた値）
O_T_EE_c / O_dP_EE_c / O_ddP_EE_c（Frankaが受理した値）
q_d / dq_d / ddq_d（Franka内部IK出力）とその数値微分 jerk
elbow_c / elbow_d
```

Franka（measured）：

```text
O_T_EE、q、dq、tau_J
制御周期（period）、control_command_success_rate、current_errors / last_motion_errors
```

グラフにはBC target更新時刻（0, 50, 100, 150, 200 ms…）を縦線表示し、velocityの折れ・acceleration spike・jerk spike・Ruckig再計画がその時刻と一致するか確認する。加えてvelocityのPSDを出し、20Hzピークの有無を示す。

## 10. 原因判定

```text
A. BC raw target自体がガタガタ
   → BCモデル、画像入力、正規化、推論ノイズ、Pose Filter不足

B. BC targetは滑らかだが Ruckig input（前処理後）が飛ぶ
   → delta基準（4.4）、quaternion符号 / 角度wrap（4.5）、chunk切替、Pose Filter実装

C. Ruckig inputは滑らかだがRuckig outputに折れが出る
   → Ruckig update周期、delta_time、current stateリセット、
     velocity/accelerationゼロ初期化、pass_to_input不足、
     target_velocity=0（stop-and-go）、Result error時のフォールバック、姿勢DoFの扱い

D. Ruckig outputは滑らかだが O_T_EE_c（Franka受理値）がガタガタ
   → command経路（topic経由 / ZOH / 書き込み漏れ周期）、libfranka rate limiter / LPF との不整合、
     回転行列の非正規直交、1kHzスレッドのブロッキング

E. O_T_EE_c は滑らかだが q_d（Franka内部IK出力）がガタガタ
   → Franka内部IK、肘（elbow）、特異点、joint制限張り付き（4.8）

F. q_d は滑らかだが measured（q / O_T_EE）だけ振動
   → Franka controller gain、機械振動、collision behavior、実機側設定

G. 全て滑らかに見えるが「周期的に減速→加速」している
   → target_velocity=0 による stop-and-go（4.3）。速度グラフに50ms周期の谷が出る

H. 合成targetでは滑らか、BCでは発振
   → delta action の基準が measured（4.4）または chunk 切替の不連続
```

## 11. 必ず回答すること（Phase 1 報告項目）

1. BC推論結果はどこで生成されているか
2. BC target更新周期は何Hzか（設定値と実測）、推論時間の分布
3. BC出力の delta の実機での解釈：基準（measured / commanded / 速度）、×0.5 の適用箇所、座標系、回転掛け順、TCP オフセット。1.1 の学習定義との差分を表で示す（4.4）
4. BC出力の姿勢表現（quaternion / rotation 6D / Euler 等）と実機側での変換
5. action chunking の有無と消費方法
6. command経路は第3節のA〜Eのどれか
7. 使用している libfranka / franka_ros2 のインターフェース名（CartesianPose、elbow の有無）
8. Ruckig objectはどこで生成されているか
9. Ruckig の DoF 数と各 DoF の意味（4.5）
10. Ruckig delta_time は何秒か
11. Ruckig update() は何Hzで呼ばれているか（実測）
12. output.pass_to_input(input) または等価処理があるか
13. Ruckig current_position / velocity / acceleration の生成元（初期値は O_T_EE_c か O_T_EE か）
14. measured stateを毎周期またはtarget更新時にcurrent stateへ代入していないか
15. BC target更新時にvelocity / accelerationを0へリセットしていないか
16. target_velocity / target_acceleration に何を入れているか（4.3）
17. Ruckig の Result code を確認・ログしているか。エラー時の処理
18. Ruckig の control interface（Position / Velocity）と synchronization 設定
19. quaternion符号揃え・正規化・角度wrap処理の有無
20. Ruckig出力から O_T_EE_d を組み立てる処理（回転行列の正規直交性）
21. Ruckig outputをFrankaへ何Hzで書いているか。書かない周期がないか
22. Ruckig output後に別のfilter / limiter / interpolation があるか
23. libfranka rate limiter / LPF cutoff の設定。O_T_EE_d と O_T_EE_c の差（クリップの有無）
24. controller_manager update_rate と hardware interface 周期
25. ROS2や別スレッドの通信遅延・jitterがないか（実測）
26. 1kHzスレッドの優先度・アフィニティ・ブロッキング呼び出し（gripper / I/O / mutex / alloc）
27. BC推論処理がリアルタイム制御スレッドをブロックしていないか。Pythonの場合GIL競合の有無
28. Franka 内部 IK 出力（q_d / dq_d）の jerk と、O_T_EE_c の滑らかさの対応（4.8）
29. elbow の扱い、動作範囲と特異点・可動域端との距離
30. libfranka の通信品質指標・エラー履歴
31. 学習データ周期（20Hz）と実機のtarget消費周期が一致しているか
32. ユーザーの fork で IsaacLab の action 定義（scale / body_offset / use_relative_mode / decimation / sim.dt）が override されていないか。HDF5 の actions が ×0.5 前か後か

## 12. スレッド構成

推論処理は計算時間が変動する。以下の分離が望ましい：

```text
Inference Thread / Process   20 Hz
   ↓ latest target共有（lock-free / RealtimeBuffer）
Realtime Control Thread      1000 Hz
   ↓ Ruckig update → CartesianPose command
```

避けること：

```text
Franka realtime callback → BC inference実行 → 推論終了まで待機 → Ruckig
```

## 13. 修正優先順位

```text
1. command経路の構成（topic経由 / ZOH / 書き込み漏れなら controller内Ruckigへ）
2. Ruckig update周期と delta_time
3. Ruckig current stateの連続性（pass_to_input、初期値は O_T_EE_c）
4. delta を速度指令として解釈し Ruckig velocity interface へ（4.3 と 4.4 を同時に解決。1.1 のスケール・座標系・掛け順を厳守）
5. BC target更新時のvelocity / acceleration初期化有無
6. （4 を採らない場合）target_velocity 推定と delta 基準の commanded 化
7. 姿勢表現の連続性（quaternion符号・正規化・wrap）と O_T_EE_d の正規直交性
8. libfranka rate limiter / LPF との整合
9. 1kHzスレッドのブロッキング要因
10. Franka 内部 IK / elbow / 特異点
11. BC target自体のノイズ
12. Pose Filter調整
13. Franka controller / servo側
```

Ruckigをすでに使用しているため、新しいtrajectory generatorを追加する前に既存Ruckig実装を正しく動かすことを最優先とする。

## 14. 完了条件

- BC推論は20Hzで動作（実測、推論時間<50ms、jitter記録）
- Ruckig updateは1kHz（または制御周期と一致）で動作
- Ruckig delta_time と実周期が整合
- Ruckig current position / velocity / accelerationが周期間で連続。初期値は commanded 基準
- target_velocity に推定速度を与えている（または velocity interface）で stop-and-go が消えている
- BC target更新時にvelocity / accelerationを不必要にゼロリセットしていない
- Ruckig出力を毎周期（1kHz）直接 CartesianPose に書いている
- delta action の解釈（scale 0.5 / base 座標系 / axis-angle 左掛け / TCP +0.107m / 20Hz）が 1.1 の学習定義と一致し、速度指令または commanded 基準で実装されている。HDF5 トレース再生でシム軌道と実機 commanded 軌道が重なる
- 姿勢表現がフレーム間で不連続になっていない（符号・正規化・wrap）
- O_T_EE_d と O_T_EE_c が一致（libfranka 側のクリップなし）
- Franka 内部 IK 出力（q_d）の jerk に 50ms 周期 spike がない。elbow が暴れていない
- TCP速度250mm/s以下、Cartesian acceleration / jerk 上限を確認
- **commanded velocity PSD の 20Hzピークが修正前比で明確に低下**（数値で報告）
- 50ms周期のjerk spikeの発生率が修正前比で明確に低下（数値で報告）
- commanded stateとmeasured stateを比較
- 制御ループ周期jitterを測定（p99 < 制御周期の10%目安）
- オフライン再生・合成target実験で原因が再現・消失することを確認
- 実機投入前にdry-runで position / velocity / acceleration / jerk をプロットしユーザー承認を得る
- 修正前後のログを同一指標で比較

## 15. 最終報告

1. ガタつきの主要原因（実験で裏付けられたもの。推定は「推定」と明記）
2. command経路の分類（第3節）と使用インターフェース
3. Ruckigの実際のupdate周期・DoF構成
4. Franka command周期
5. BC target更新周期・推論時間分布
6. Ruckig current stateの更新方法・初期値
7. pass_to_input相当処理の有無
8. target_velocity の扱い（修正前後）
9. delta action の解釈（修正前後）と 1.1 の学習定義との整合表
10. 姿勢表現の処理（修正前後）
11. O_T_EE_d vs O_T_EE_c の一致状況
12. Franka 内部 IK（q_d）の jerk spike の有無、elbow の挙動
13. 修正したファイル・修正内容
14. 修正前後のログ比較（20Hz PSDピーク、jerk RMS、jitter）
15. 最終的な制限値
16. 残っているリスク・未検証項目（Franka 内部 IK 起因で解決しない場合の joint 空間構成への切替提案を含む）
