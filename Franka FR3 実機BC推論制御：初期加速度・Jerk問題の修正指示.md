# Franka FR3 実機BC推論制御：初期加速度・Jerk問題の修正指示

## 背景

現在、BCモデルの1ステップ目の推論結果として、EE位置の変位が約 **8.9 mm** 出ています。

BCの推論周期はFrankaの制御周期より低速ですが、この8.9 mmの目標変位がFrankaの **1 kHz制御ループ側でほぼ即時に位置指令へ反映されている可能性**があります。

その結果、制御開始直後に以下のいずれかが発生し、Frankaの安全制御によって停止している可能性があります。

- position discontinuity
- velocity discontinuity
- acceleration discontinuity
- jerk limit超過
- Cartesian IK後のjoint velocity / acceleration / jerk超過

重要なのは、**BCの8.9 mmという推論結果そのものを単純に小さくすることではありません。**

BC出力は「次に到達したいtarget」として扱い、そのtargetまでFrankaが物理的に実行可能な滑らかなtrajectoryを、別レイヤーで生成してください。

---

# 1. まず原因を調査

現在の実装を確認し、以下を特定してください。

1. BCの推論周期
2. Frankaへcommandを送っている制御周期
3. BCの`Δposition / Δrotation`がどこでabsolute targetへ変換されているか
4. 推論結果が1回のFranka control cycleでそのまま反映されていないか
5. 以下のどの制御方式を使用しているか
   - Cartesian position
   - Joint position
   - Velocity
   - Torque
6. 以下のどのinterfaceを使用しているか
   - libfranka
   - franka_ros2
   - ros2_control
7. 現在以下が存在するか
   - rate limiter
   - low-pass filter
   - trajectory interpolation
   - trajectory generator
8. 制御開始時の最初のcommandが現在のcommanded stateと一致しているか
9. 実際に発生しているFrankaのエラー内容

調査結果を簡潔にまとめてから修正してください。

---

# 2. 必須修正：初期commandを現在のcommanded stateと一致させる

制御開始時にBCの最初のtargetをそのまま送らないでください。

## Joint制御の場合

原則として、

```cpp
q_ref_initial = robot_state.q_c;
```

## Cartesian制御の場合

原則として、

```cpp
T_ref_initial = robot_state.O_T_EE_c;
```

など、Frankaが保持している **last commanded state** を初期値として使用してください。

`measured state`と`commanded state`を混同しないよう注意してください。

制御開始時は、

```text
position     = current commanded position
velocity     = 0
acceleration = 0
```

から滑らかに開始してください。

---

# 3. 必須修正：BC周期とFranka 1 kHz周期を分離する

構造を以下のようにしてください。

```text
BC inference
例: 20〜30 Hz
    ↓
Desired EE Target
    ↓
Trajectory Generator
    ↓
Smooth Command @ 1 kHz
    ↓
Franka
```

BC推論結果を直接1 kHz commandとして使用しないでください。

例えばBCが1ステップで、

```text
Δposition = 8.9 mm
```

を出した場合、

```text
current_position + 8.9 mm
```

を「次の1 ms後の位置」として扱わず、

**新しい目標位置**

としてtrajectory generatorへ入力してください。

---

# 4. 必須修正：Velocity / Acceleration / Jerk制限付きTrajectory Generator

可能であれば **Ruckig** を使用してください。

Ruckigが既存依存関係やリアルタイム要件上適切でない場合は、以下のいずれかを使用してください。

- minimum-jerk trajectory
- S-curve trajectory
- 同等のjerk-limited trajectory generator

最低限、以下を明示的に設定できる構造にしてください。

```text
max_velocity
max_acceleration
max_jerk
```

## Cartesian制御の場合

translationとrotationを分離してください。

```text
max_translational_velocity
max_translational_acceleration
max_translational_jerk

max_rotational_velocity
max_rotational_acceleration
max_rotational_jerk
```

これらはハードコードせず、configから変更可能にしてください。

---

# 5. 単純Linear Interpolationだけで済ませない

以下のように、

```text
8.9 mm / 50 samples
```

として等間隔に分割するだけでは不十分です。

位置自体は滑らかに見えても、開始時に、

```text
velocity:
0 → constant velocity
```

となるため、velocity discontinuity / acceleration / jerk問題が残ります。

したがって、

```text
Position
Velocity
Acceleration
```

が連続になる軌道生成を使用してください。

可能ならjerkも明示的に制限してください。

---

# 6. 必須修正：BC Target更新時もTrajectoryを連続させる

BCは継続的に新しいtargetを出します。

そのため、BC target更新のたびに、

```text
velocity = 0
acceleration = 0
```

からtrajectoryを再生成する実装にはしないでください。

新しいtargetが来た時点での、

```text
current commanded position
current commanded velocity
current commanded acceleration
```

をtrajectory generatorのinitial stateとして使用してください。

つまり、新しいBC targetへ切り替わる際にも、

```text
Position continuity
Velocity continuity
Acceleration continuity
```

を維持してください。

理想的な構造は以下です。

```text
BC Target A
     ↓
Trajectory実行中
     ↓
現在のp, v, a
     ↓
BC Target B到着
     ↓
現在のp, v, aを初期状態として
Target Bへtrajectory再計算
```

---

# 7. 8.9 mmを単純Clampするだけの修正は禁止

今回の問題を、例えば、

```cpp
delta_position = clamp(delta_position, -0.001, 0.001);
```

のような処理だけで解決しないでください。

安全上のabsolute clampを追加すること自体は構いません。

ただし、本質的な構造として、

```text
BC output
    ↓
Desired Target
    ↓
Physically Feasible Trajectory
    ↓
Franka Command
```

に変更してください。

---

# 8. 学習時と実機時のTime Semanticsを確認

BC学習データの1 stepが何秒に相当するか確認してください。

例えば学習データが20 Hzの場合、

```text
1 step = 50 ms
```

です。

この場合、

```text
8.9 mm / step
```

というBC出力は、

```text
50 ms程度で8.9 mm移動する
```

という意味を持っている可能性があります。

これを、

```text
1 msで8.9 mm移動
```

として実機へ適用すると、学習時と実機時でtime semanticsが大きく崩れます。

以下を確認してください。

```text
Training control frequency
Inference frequency
Command frequency
Action horizon
Action representation
```

特に、

```text
BC Δposition
```

が、

- 1 inference stepあたりの変位
- 1 control cycleあたりの変位
- absolute target
- velocity相当

のどれとして学習されているか確認してください。

---

# 9. Cartesian IK後のJoint Limitも監視

Cartesian commandを使用している場合、EE trajectoryが滑らかでも、Franka内部IK後のjoint motionが急激になる可能性があります。

以下を監視してください。

```text
q
dq
ddq
joint jerk（可能なら）
EE linear velocity
EE angular velocity
Jacobian condition number
または manipulability
```

さらに以下も同時にログしてください。

```text
BC raw output
BC target
Trajectory Generator output
Actual commanded value
```

---

# 10. Singularity / IK Configuration変化の確認

以下の場合、Cartesian trajectoryが滑らかでもjoint accelerationが急増する可能性があります。

- 特異点付近
- workspace端
- wrist singularity
- elbow configurationの変更
- orientationの急変
- IK solution branchの切り替わり

Jacobianのcondition numberまたはmanipulabilityを監視し、危険領域では、

```text
速度低下
または
停止
```

できる構造を検討してください。

---

# 11. IK後Joint Limit違反が残る場合

Cartesian trajectoryを十分滑らかにしても、

```text
joint velocity
joint acceleration
joint jerk
```

のlimit violationが残る場合は、次の構成への変更を検討してください。

```text
BC
 ↓
EE Target
 ↓
External IK
 ↓
q_target
 ↓
Joint-space Ruckig
 ↓
q_command @ 1 kHz
 ↓
Franka
```

External IKでは、

```text
previous q
```

をwarm startとして利用し、前回joint stateに最も近い解を優先してください。

ただし、最初から大規模にJoint-space化せず、

**まず現在のCartesian構成にtrajectory generatorを追加する最小修正**

から行ってください。

---

# 12. 必須修正：起動時Soft Start

起動直後にBC commandを100%適用しないでください。

以下のようなstate machineを実装してください。

```text
INITIALIZING
    ↓
Franka commanded state取得
    ↓
Trajectory Generator初期化
    ↓
HOLDING
現在commanded poseを保持
    ↓
BC inference開始
    ↓
最初のBC target取得
    ↓
Trajectory Generatorへtarget登録
    ↓
STARTING
滑らかにtrajectory開始
    ↓
RUNNING
通常BC制御
```

固定時間で単純にBC出力を線形スケールするより、trajectory generatorによる制約を優先してください。

---

# 13. デバッグログ追加

少なくとも制御開始から最初の2秒間について、解析可能な周期で以下を保存してください。

```text
timestamp

BC inference timestamp

raw BC delta position
raw BC delta rotation

target EE position
target EE orientation

commanded EE position
commanded EE orientation

commanded EE linear velocity
commanded EE angular velocity

commanded EE acceleration
commanded EE angular acceleration

joint position
joint velocity
joint acceleration

trajectory generator state

Franka error state
```

特に重要なのは、

```text
最初のBC output = 8.9 mm
```

に対して、

```text
Frankaへ送ったcommandが
1 control cycleで何mm変化したのか
```

を明確に確認できるようにすることです。

---

# 14. 安全設定

最初の実機試験では、かなり保守的な、

```text
velocity
acceleration
jerk
```

上限を使用してください。

これらはconfigへ切り出してください。

例：

```yaml
trajectory:
  translation:
    max_velocity: ...
    max_acceleration: ...
    max_jerk: ...

  rotation:
    max_velocity: ...
    max_acceleration: ...
    max_jerk: ...
```

実際の値はFranka公式仕様、現在のcontroller方式、実験速度、学習データの時間スケールを確認した上で決定してください。

安全値を根拠なく決め打ちしないでください。

---

# 15. libfranka側Rate Limiterだけに依存しない

libfrankaやFranka側にrate limiterが存在していても、それだけで今回の問題を解決しないでください。

構造としては、

```text
BC
 ↓
Application-side Trajectory Generator
 ↓
Franka / libfranka Rate Limiter
 ↓
Robot
```

としてください。

Application側trajectory generatorを主とし、Franka側limiterは最後の安全網として扱ってください。

---

# 16. 実装後の検証

修正後は必ず自身で以下を確認してください。

## Build

```text
build success
```

## Unit Test

既存unit testをすべて実行してください。

## Regression Test

既存機能を壊していないことを確認してください。

## Trajectory Test

Simulationまたはmockでtrajectory生成を確認してください。

以下を必ずチェックしてください。

1. 初期commandにposition jumpがない
2. velocityが連続
3. accelerationが連続
4. jerkが設定上限以内
5. BC target更新時にposition discontinuityが発生しない
6. BC target更新時にvelocity discontinuityが発生しない
7. BC target更新時にacceleration discontinuityが発生しない

---

# 17. 8.9 mm入力の再現テストを追加

以下のtest caseを追加してください。

## Initial State

```text
position = 0
velocity = 0
acceleration = 0
```

## BC Output

```text
delta_position = 8.9 mm
```

## Expected Behavior

NG：

```text
t = 0 ms   : 0 mm
t = 1 ms   : 8.9 mm
```

OK：

```text
t = 0 ms   : 0 mm
t = 1 ms   : very small displacement
t = 2 ms   : slightly larger displacement
...
```

設定された、

```text
max_velocity
max_acceleration
max_jerk
```

を一度も超えず、8.9 mm先のtargetへ滑らかに接近することを確認してください。

---

# 18. BC Target更新テスト

例えば制御途中で、

```text
Target A
```

から、

```text
Target B
```

へ切り替わった場合もテストしてください。

Target B受信時に、

```text
velocity = 0
```

へリセットされないことを確認してください。

Target切り替え前後で、

```text
position
velocity
acceleration
```

が連続していることを確認してください。

---

# 19. 修正時の優先順位

以下の順番で対応してください。

1. 現在の制御フローを調査
2. 実際のFrankaエラーを特定
3. 初期commandをcommanded stateへ一致
4. BC周期と1 kHz control loopを分離
5. jerk-limited trajectory generatorを追加
6. 8.9 mm再現テスト
7. target更新時の連続性テスト
8. 実機用デバッグログ追加
9. Cartesian IK後joint limitを確認
10. 必要な場合のみExternal IK + Joint-space trajectoryへ拡張

---

# 20. 避けるべき修正

以下のような「症状を隠すだけ」の変更は避けてください。

```text
8.9mm → 1mmに単純clamp
```

```text
sleep()を追加して遅くする
```

```text
BC推論周期だけ下げる
```

```text
Linear interpolationだけ追加
```

```text
Franka側rate limiter任せ
```

```text
安全limitを単純に緩和
```

特に、Franka側の安全制限を緩めることで問題を解決しないでください。

---

# 21. 望ましい最終アーキテクチャ

まずは以下を目標としてください。

```text
RealSense
    ↓
Preprocessing
    ↓
BC Policy
20〜30 Hz程度
    ↓
Raw ΔEE Action
    ↓
Target Manager
    ├─ action semantics
    ├─ workspace limit
    ├─ rotation limit
    └─ safety checks
    ↓
Desired EE Target
    ↓
Trajectory Generator
1 kHz
    ├─ velocity limit
    ├─ acceleration limit
    └─ jerk limit
    ↓
Cartesian Command
    ↓
Franka Controller
    ↓
FR3
```

もしIK後joint limit問題が残る場合のみ、

```text
Desired EE Target
    ↓
External IK
    ↓
q_target
    ↓
Joint-space Trajectory Generator
    ↓
Joint Command
    ↓
Franka
```

へ拡張してください。

---

# 22. 実装後の報告

修正後は以下を報告してください。

## 1. 根本原因

何が直接の停止原因だったか。

## 2. 変更したファイル

ファイルパスと変更概要。

## 3. 元の制御フロー

```text
Before:
...
```

## 4. 修正後の制御フロー

```text
After:
...
```

## 5. Trajectory Generator方式

例：

```text
Ruckig
Minimum Jerk
S-Curve
```

## 6. 制限値

```text
max translational velocity
max translational acceleration
max translational jerk

max rotational velocity
max rotational acceleration
max rotational jerk
```

## 7. 8.9 mm入力時の結果

最初の10〜20 ms程度について、

```text
time
position
velocity
acceleration
```

を提示してください。

## 8. テスト結果

```text
Build
Unit test
Regression test
Trajectory test
```

## 9. 残っているリスク

特に、

```text
Singularity
IK branch switching
Joint limit
Inference latency
Control jitter
```

について報告してください。

## 10. 実機試験時に最初に確認すべきログ

最初の実機試験で重点的に監視すべき値を列挙してください。

---

# 最重要方針

BCには、

> 「次にどこへ行きたいか」

を決めさせてください。

Frankaへ直接、

> 「次の1 msでここまで移動しろ」

とは指令させないでください。

責務を以下のように分離してください。

```text
BC
= Desired Target Generator

Trajectory Generator
= Physically Feasible Motion Generator

Franka Controller
= Robot Execution Layer
```

今回の8.9 mmについても、推論値そのものを無理に変更するのではなく、

```text
8.9 mm先のTarget
        ↓
Velocity / Acceleration / Jerk constrained trajectory
        ↓
1 kHz command
```

として処理する構成に修正してください。

既存アーキテクチャを無闇に大改造せず、まず実際の原因を確認したうえで、最小限かつ堅牢な変更を実装してください。