# Franka FR3 + RealSense D435i 2台：easy_handeye2 によるハンドアイキャリブレーション引き継ぎ

作成日：2026-10-30
対象：Franka FR3 / ROS 2 Jazzy / franka_ros2 v3.6.1 / RealSense D435i ×2（手首・机）/ easy_handeye2 / aruco_opencv
目的：別セッション・別エージェントが、ここまでの調査結果と決定事項を前提に作業を続けられるようにする
関連ファイル：`franka_bc_ruckig_discussion_summary.md`（BC + Ruckig の検討）、`FR3_CONTROL_STATION_DESIGN.md`

---

## 目次

1. [背景と目的](#sec1)
2. [前提となるシステム構成](#sec2)
3. [easy_handeye2 の概要（ソース確認済み）](#sec3)
4. [処理の流れ](#sec4)
5. [2台のカメラそれぞれの設定](#sec5)
6. [ArUco マーカー検出ノードの選定と設定](#sec6)
7. [TF の前提知識（最小限）](#sec7)
8. [落とし穴（ソースを読んで判明したもの）](#sec8)
9. [推論ノードとの関係](#sec9)
10. [未確認事項](#sec10)
11. [次の作業候補](#sec11)
12. [調査した情報源](#sec12)

---

<a id="sec1"></a>
## 1. 背景と目的

Isaac Lab で学習した BC 方策（行動 = 手先姿勢の差分、観測 = 手首カメラ + 机カメラの RGB 画像 + 手先姿勢など）を FR3 実機で推論させる。そのために、**2台の D435i とロボットの位置関係を TF として正確に求める**必要がある。

キャリブレーション結果の使い道：

- カメラで検出した物体の位置を、ロボットのベース座標系 `fr3_link0` で直接扱えるようにする
- Isaac Lab のシーン内のカメラ配置と実機のずれを確認・修正する（数 cm のずれで BC の性能が大きく落ちることがある）

---

<a id="sec2"></a>
## 2. 前提となるシステム構成

| 項目 | 内容 |
|---|---|
| ロボット | Franka FR3（Franka Hand 付き） |
| ROS | ROS 2 Jazzy / Ubuntu 24.04 |
| franka_ros2 | `frankarobotics/franka_ros2`（旧 `frankaemika`）。既定ブランチ `jazzy`。最新 v3.6.1（2026-10-27）。libfranka 0.20.5、franka_description 2.9.0 を指定 |
| 制御 | デカルト速度の指令（`FrankaCartesianVelocityInterface`）。目標座標から速度を生成し、Ruckig で平滑化する方針 |
| カメラ | D435i ×2。ドライバは Intel の `realsense2_camera`（realsense-ros 4.59.1、リポジトリは `realsenseai/realsense-ros` に移転済み） |
| 推論 | 自作の `policy_inference` パッケージ（ローカルに雛形あり、[9章](#sec9)） |

franka_ros2 にはカメラドライバも D435i 向けの構成もない。カメラ関連で入っているのは D405（手首）、ZED Mini、D455（モバイル）向けの launch ファイルと URDF だけ（`franka_description_extensions/` 配下）。

---

<a id="sec3"></a>
## 3. easy_handeye2 の概要（ソース確認済み）

| 項目 | 内容 |
|---|---|
| リポジトリ | `marcoesposito1988/easy_handeye2`（LGPL-3.0）。ROS 1 の `easy_handeye` の ROS 2 移植 |
| 更新状況 | master の最終更新 2026-08。「Jazzy 対応」の issue は 2025-04 に解決済み |
| 導入 | ソースからビルド（clone → `rosdep install` → `colcon build`）。README に apt での導入方法はない |
| 依存 | rclpy、tf2_ros、OpenCV、transforms3d、rqt。**MoveIt は不要** |
| 計算 | `cv2.calibrateHandEye`。アルゴリズムは Tsai-Lenz / Park / Horaud / Andreff / Daniilidis の 5 種類から選択 |
| デモ | `marcoesposito1988/easy_handeye2_demo`（実機なしで試せる） |

構成要素：

| 要素 | 役割 |
|---|---|
| `calibrate.launch.py` | `handeye_server`（サンプル取得・計算）と `rqt_calibrator`（GUI）を起動。**仮の静的 TF も出す**（[8章](#sec8)） |
| `publish.launch.py` | 保存済みの結果を読み込み、静的 TF として publish するだけ |
| `evaluate.launch.py` | 精度確認用 GUI（`rqt_evaluator`） |
| `easy_handeye2_msgs` | サンプル取得・削除・保存・読み込み、計算、アルゴリズム選択のサービス定義。スクリプトから直接呼べる |

保存先（`easy_handeye2/__init__.py` で固定）：

- 結果：`~/.ros2/easy_handeye2/calibrations/<name>.calib`（YAML）
- サンプル：`~/.ros2/easy_handeye2/samples/`

---

<a id="sec4"></a>
## 4. 処理の流れ

```
① calibrate.launch.py を起動（name、calibration_type、4 つの TF フレーム名を指定）
② ロボットを静止させ、GUI の「Take sample」を押す
     → TF から次の 2 つを組にして記録
        ・robot_base_frame → robot_effector_frame（ロボットの姿勢）
        ・tracking_base_frame → tracking_marker_frame（カメラから見たマーカー）
     ※ 記録するのは「現在時刻の 0.2 秒前」の TF。動かした直後に押さない
③ 姿勢を変えて ② を繰り返す（15〜20 点以上）
④ 「Compute」で計算。複数のアルゴリズムで結果を比べ、大きく違えばサンプルの質を疑う
⑤ 「Save」で保存
⑥ 運用時は publish.launch.py name:=<name> で静的 TF を出す
     ・eye_in_hand：robot_effector_frame → tracking_base_frame
     ・eye_on_base：robot_base_frame → tracking_base_frame
```

評価ツールの仕組み（eye_in_hand の場合）：ロボットを動かしても「ベース座標系で見たマーカー位置」は本来変わらないので、そのばらつき（位置の最大のずれ）を表示する。並進しか見ておらず計算も簡易的なので、目安として使う。

ロボットの動かし方：`franka_example_controllers` の `gravity_compensation_example_controller` を有効にすれば、手で腕を動かしながら TF を出し続けられる。

```bash
ros2 launch franka_bringup example.launch.py controller_names:=gravity_compensation_example_controller
```

---

<a id="sec5"></a>
## 5. 2台のカメラそれぞれの設定

`name` を変えて 2 回キャリブレーションする。

| | 手首カメラ | 机カメラ |
|---|---|---|
| 種類 | `eye_in_hand` | `eye_on_base` |
| マーカーの置き場所 | 机に固定 | ハンドに固定（治具か指で挟む） |
| `robot_base_frame` | `fr3_link0` | `fr3_link0` |
| `robot_effector_frame` | カメラを付けたリンク（`fr3_hand` か `fr3_link8`） | マーカーを付けたリンク（`fr3_hand` など） |
| `tracking_base_frame` | `wrist_cam_link`（RealSense の TF の根元） | `table_cam_link` |
| `tracking_marker_frame` | `marker_<id>`（[6章](#sec6)） | 同左 |
| 求まる結果 | `fr3_hand → wrist_cam_link` | `fr3_link0 → table_cam_link` |

手首カメラの起動例：

```bash
ros2 launch easy_handeye2 calibrate.launch.py \
  name:=fr3_wrist_d435i calibration_type:=eye_in_hand \
  robot_base_frame:=fr3_link0 robot_effector_frame:=fr3_hand \
  tracking_base_frame:=wrist_cam_link tracking_marker_frame:=marker_0
```

サンプルの集め方（README と経験則）：

- **各軸まわりに ±最大 90° 程度まで回転を大きく変える**。並進はあまり変えなくてよい
- マーカーはカメラの近くに置く
- 冗長な姿勢を含めて 15〜20 点以上集める

---

<a id="sec6"></a>
## 6. ArUco マーカー検出ノードの選定と設定

### 選定結果

**`aruco_opencv`（fictionlab）を採用する。** 次点は `aruco_ros`。

| パッケージ | Jazzy の apt | 最終更新 | 評価 |
|---|---|---|---|
| `aruco_opencv` | ✅ `ros-jazzy-aruco-opencv` 6.1.2（Jazzy 専用ブランチあり） | 2026-06 | **採用**。検出したマーカーを全部 TF で出す（`marker_<id>`、ボードは `board_<name>`）。PnP の 2 解から正しい方を選ぶ設定あり。デバッグ画像あり |
| `aruco_ros`（PAL Robotics） | ✅ 5.0.6（`humble-devel` から Jazzy 向けにもリリース） | 2026-10 | 次点。実績は一番多い。`single` ノードで TF のフレーム名を指定できる。README の大半が ROS 1 時代のまま |
| `apriltag_ros` | ✅ 3.4.0 | 2026-10 | AprilTag 用。マーカーの種類にこだわりがなければ有力 |
| `ros2_aruco`（JMU） | ❌ Jazzy の一覧にない | 2024-02 | 避ける |

### 手首カメラでの設定例

```bash
sudo apt install ros-jazzy-aruco-opencv
```

```yaml
# wrist_aruco.yaml
/**:
  ros__parameters:
    cam_base_topic: /wrist_cam/wrist_cam/color/image_raw   # camera_info はここから自動で決まる
    output_frame: ''           # 空ならカメラの光学フレーム基準（easy_handeye2 にはこれでよい）
    marker_dict: 4X4_50        # 印刷したマーカーの辞書に合わせる
    marker_size: 0.10          # m。黒枠の外側の一辺を印刷後に実測した値
    image_is_rectified: false  # camera_info の歪み係数を使わせる
    publish_tf: true
    pose_selector:
      strategy: REPROJECTION_ERROR
```

```bash
ros2 run aruco_opencv aruco_tracker_autostart --ros-args \
  -r __node:=wrist_aruco --params-file wrist_aruco.yaml
```

動作確認：

- `~/debug` の画像を `rqt_image_view` で見て、マーカーの輪郭と座標軸が描かれているか
- 静止状態で `ros2 run tf2_ros tf2_echo wrist_cam_link marker_0` の値が揺れていないか

### マーカーの準備

- 小さい辞書（4X4_50 など）の方が誤検出が少なく、遠くからも検出しやすい
- 「実際のサイズ」で印刷し、**黒枠の一辺を必ず実測**して `marker_size` に入れる（プリンタの自動縮小で数時間無駄にする典型例）
- 硬い平板に貼ってたわみをなくし、周囲に白い余白を残す
- 正面から見ると姿勢推定が 2 解の間で揺れるので、斜めから見る
- 精度を上げたい場合はボード（`board_descriptions.yaml` に配置を記述）を使う

---

<a id="sec7"></a>
## 7. TF の前提知識（最小限）

- TF は座標系どうしの位置関係を木構造で管理する仕組み。**1 つのフレームが持てる親は 1 つだけ**
- `/tf` は動く変換、`/tf_static` は固定の変換
- バッファに数秒分たまるので、**過去の時刻を指定すると補間して返す**（画像のタイムスタンプで手先姿勢を引く用途に使う）
- カメラの座標系は 2 種類：`*_link`（x 前、z 上）と `*_optical_frame`（z 前、x 右、y 下）。画像から求めた 3D 点は光学フレームの値

想定する TF ツリー：

```
fr3_link0
 ├ fr3_link1 → … → fr3_link8 → fr3_hand → fr3_hand_tcp   （robot_state_publisher）
 │                              └ wrist_cam_link          （easy_handeye2 publish）
 │                                  └ wrist_cam_color_optical_frame （realsense2_camera）
 │                                      └ marker_0        （aruco_opencv）
 └ table_cam_link                                          （easy_handeye2 publish）
     └ table_cam_color_optical_frame
```

確認コマンド：`ros2 run tf2_tools view_frames`（ツリー全体を PDF に出力）、`ros2 run tf2_ros tf2_echo <A> <B>`

---

<a id="sec8"></a>
## 8. 落とし穴（ソースを読んで判明したもの）

1. **`calibrate.launch.py` は仮の静的 TF を自分で出す**
   - eye_in_hand：`robot_effector_frame → tracking_base_frame`（z=0.1 m）、eye_on_base：`robot_base_frame → tracking_base_frame`（x=1 m）
   - URDF にカメラを組み込んでいると、同じフレームに親が 2 つできて TF が壊れる。**キャリブレーション中は URDF からカメラを外す**
   - `tracking_base_frame` には **RealSense の TF の根元（`*_camera_link`）** を指定する。光学フレームを指定すると realsense2_camera が出す親と衝突する
2. **運用時も同じ衝突に注意**。`publish.launch.py` と URDF の固定ジョイントの**どちらか一方**だけでカメラ位置を定義する。URDF に反映するなら結果を xacro に書き写し、publish は使わない
3. **自動で姿勢を変える機能は Jazzy では使えない前提にする**（README に「Iron 向けで作業中」）。`freehand_robot_movement:=true` で手動運用
4. **サンプル数の下限チェックは 2 点だけ**（`MIN_SAMPLES = 2`）。計算の最低条件にすぎないので、15〜20 点以上集める
5. **Docker では保存先 `~/.ros2/easy_handeye2/` をボリュームでマウントする**。しないとコンテナ再作成で結果が消える
6. **2 台のカメラで同時に検出ノードを動かすと TF が衝突する**（両方が同じ `marker_<id>` を出す）。キャリブレーション中は対象カメラの検出ノードだけを動かすか、カメラごとに別 ID のマーカーを使う
7. 精度を左右するのはマーカー検出。静止時にマーカーの TF が揺れていないことを確認してからサンプルを取る。D435i の内部パラメータは工場校正済みで、通常はそのままでよい

---

<a id="sec9"></a>
## 9. 推論ノードとの関係

ローカル（`C:\Users\table\frankaros2\policy_inference\`、未 push）に推論ノードの雛形がある。

- 構成：`processing.py`（ROS 非依存の観測・行動変換。Isaac Lab 側からも import できる）、`policy.py`（ONNX Runtime）、`inference_node.py`、`config/params.yaml`、`launch/inference.launch.py`
- 処理：2 台の画像を `ApproximateTimeSynchronizer` で組にし、手先姿勢を TF（`fr3_link0 → fr3_hand_tcp`）から取得し、差分を Isaac Lab の差分 IK 相対モードと同じ規則（並進はベース座標系で加算、回転ベクトルは左から乗算）で絶対姿勢にして `goal_pose` に publish
- 安全機能：`~/arm` で有効化するまで動かない、1 ステップの移動量制限（既定 2 cm / 0.1 rad）、作業範囲制限、画像が古い・TF が取れないときのスキップと姿勢保持
- 検証状況：構文チェックと変換処理の単体テストのみ。**ROS 2 上でのビルド・起動は未実施**

キャリブレーションとの関係：

- 手首カメラの位置関係は観測に直接入らないが、Isaac Lab のシーンのカメラ取り付け位置とのずれを確認するのに使う
- 机カメラの外部パラメータは学習環境のカメラ配置に反映できる

関連する決定事項（同じ会話で検討）：

- 接触で力の上限を守りたい場合、**デカルト速度モードでは守れない**（内部コントローラが指令軌道を硬く追従し、最後は reflex で停止する）
- 守るには、トルク制御の**デカルトインピーダンスに誤差の上限処理を加える**（`F ≈ K·e_max`）。`cartesian_impedance_example_controller` はデモの円弧軌道が毎周期目標位置を上書きするので、そのままでは外部目標を受け付けない
- `service_server/set_cartesian_stiffness` はロボット内部のインピーダンスで、トルク制御には効かない（デカルト速度モードでは追従の硬さになる）

---

<a id="sec10"></a>
## 10. 未確認事項

- easy_handeye2 の rqt GUI が Jazzy で問題なく動くか（issue は解決済みだが実機では未確認）
- D435i の手首マウント治具（純正は D405 + Robotiq 用のみ）の形状とオフセット
- 手首カメラを `fr3_hand` と `fr3_link8` のどちらに付けたものとして扱うか（グリッパー交換の可能性で決める）
- 使用するマーカーの辞書・ID・サイズ、ボードを使うか
- 2 台のカメラのハードウェア同期（同期ケーブル）を使うか、ソフトウェアのペアリングで済ませるか
- Isaac Lab のグリッパー行動の符号（`BinaryJointPositionAction` は負で閉じると記憶しているが未確認）

---

<a id="sec11"></a>
## 11. 次の作業候補

1. 検出ノードと easy_handeye2 をまとめた launch ファイル一式（手首用・机用の 2 種類）を作る
2. `.calib` の結果を xacro に書き写すスクリプトを作る（URDF でカメラ位置を管理する場合）
3. 手首カメラのキャリブレーションを実施し、評価ツールとアルゴリズム間の比較で精度を確認
4. 机カメラのキャリブレーションを実施
5. 結果を Isaac Lab のシーンのカメラ配置と比べ、ずれを記録する

---

<a id="sec12"></a>
## 12. 調査した情報源

すべて 2026-10-29〜30 に GitHub API とリポジトリのソースで確認した。

- `marcoesposito1988/easy_handeye2`（master）：README、`docs/troubleshooting.md`、`launch/*.py`、`handeye_sampler.py`、`handeye_calibration*.py`、`handeye_publisher.py`、`handeye_rqt_evaluator_widget.py`、`__init__.py`
- `fictionlab/aruco_opencv`（jazzy）：`config/aruco_tracker.yaml`、`config/board_descriptions.yaml`、`launch/aruco_tracker.launch.xml`、`src/aruco_tracker.cpp`
- `ros/rosdistro` の `jazzy/distribution.yaml`（各パッケージのリリース状況）
- `IntelRealSense/realsense-ros`（ros2-master）：README、`launch/rs_multi_camera_launch_sync.py`
- `frankarobotics/franka_ros2`（jazzy）：`franka_description_extensions/`、`franka_example_controllers/`、`franka_hardware/`
