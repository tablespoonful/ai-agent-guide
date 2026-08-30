# FR3 Control Station 設計書

**Version:** 0.1  
**Target Robot:** Franka Research 3 (FR3)  
**Primary Platform:** Ubuntu 24.04 LTS  
**ROS:** ROS 2 Jazzy  
**Language:** C++20  
**GUI:** Qt 6 / Qt Quick / QML  
**Build:** CMake + colcon  
**Architecture:** Multi-process ROS 2 architecture

---

# 1. 目的

Franka Research 3 を研究・開発用途で安全かつ効率的に操作・監視するための統合デスクトップアプリケーション **FR3 Control Station** を開発する。

以下を1つのGUIから扱えることを目標とする。

- FR3接続状態確認
- Robot State監視
- Joint Jog
- Cartesian Jog
- Gripper操作
- MoveIt 2によるMotion Planning
- 目標Pose指定
- RealSense等のカメラ表示
- 3D Digital Twin
- BC / VLA等のPolicy実行
- Policy入出力監視
- Safety監視
- Controller管理
- rosbag記録
- Telemetry
- Event / Error Log
- Calibration情報確認
- システムDiagnostics

GUIそのものはリアルタイム制御を担当しない。

**GUI・Policy・Camera・Robot Controlを明確に分離する。**

---

# 2. 基本設計思想

最重要原則は以下。

```text
GUI ≠ Robot Controller
```

FR3のFCI/libfrankaは1 kHzでRobot State取得およびリアルタイム制御を行う。公式ドキュメントでも1 kHz制御ループが前提となっており、リアルタイムループ内でのblocking、sleep、頻繁なlogging、dynamic allocation等を避けることが求められている。

参考:
- https://frankarobotics.github.io/docs/doc/libfranka/docs/overview.html

したがってアーキテクチャは、

```text
Qt/QML GUI
    │
    │ ROS 2
    ▼
Application Layer
    │
    ├── Safety
    ├── MoveIt
    ├── Policy
    └── Camera
    │
    ▼
ros2_control
    │
    ▼
Realtime Controller
    │
    ▼
franka_hardware / libfranka
    │
    ▼
FR3
```

とする。

---

# 3. 技術スタック

## 3.1 OS

推奨：

```text
Ubuntu 24.04 LTS
```

現行Franka公式ドキュメントでは `franka_ros2: Jazzy` が提供されており、libfrankaはUbuntu 20.04 / 22.04 / 24.04をサポートしている。リアルタイム用途ではPREEMPT_RT kernelが推奨されている。

参考:
- https://frankarobotics.github.io/docs/

## 3.2 GUI

```text
Qt 6
Qt Quick
QML
C++ backend
```

役割：

```text
QML
 ├─ Layout
 ├─ Animation
 ├─ User interaction
 └─ Visual components

C++
 ├─ ROS 2
 ├─ State management
 ├─ Data models
 ├─ Camera
 ├─ Telemetry
 ├─ Policy management
 └─ Safety interface
```

Qt自身もQMLをUI、C++をapplication logicとして分離する構成を想定している。C++型はQObject/QML typeとして公開する。

参考:
- https://doc.qt.io/qt-6/qtqml-cppintegration-overview.html

**QMLからROS 2を直接扱わない。**

---

# 4. Qtライセンス上の注意

3D Digital TwinにQt Quick 3Dを使用する場合、ライセンス確認を必須とする。

現行Qt documentationではQt Quick 3Dの利用条件について、Qt Commercial Licenseおよびオープンソースライセンス条件を確認して採用すること。

参考:
- https://doc.qt.io/qt-6/licensing.html

研究室内・社内利用ではQt Quick 3Dを第一候補とする。

Closed-source製品として外部配布する場合は、Qt Commercial Licenseを含む利用条件をリリース前に再確認する。

3D rendererは交換可能な境界を維持する。

代替候補：

```text
Qt 3D
Filament
OpenGL/Vulkan custom renderer
```

---

# 5. システム全体構成

```text
┌────────────────────────────────────────────────────┐
│               FR3 Control Station GUI              │
│                Qt Quick / QML                      │
│                                                    │
│ Dashboard / Camera / 3D / Policy / Telemetry      │
└──────────────────────┬─────────────────────────────┘
                       │ ROS 2
                       ▼
┌────────────────────────────────────────────────────┐
│                  App Manager                       │
│                                                    │
│ Mode Management                                    │
│ Controller Arbitration                             │
│ User Commands                                      │
└──────┬──────────────┬──────────────┬────────────────┘
       │              │              │
       ▼              ▼              ▼
 Safety          MoveIt 2        Policy Manager
 Supervisor
       │                              │
       └──────────────┬───────────────┘
                      ▼
                Command Gateway
                      │
                validated command
                      ▼
              ros2_control Controller
                      │
              1000 Hz RT control
                      ▼
                franka_hardware
                      │
                  libfranka
                      │
                     FR3
```

Camera系はRobot Controlから独立させる。

```text
RealSense #1 ─┐
              ├── Camera Service ─── GUI
RealSense #2 ─┘          │
                         └── Policy Runner
```

---

# 6. ROS 2 Process構成

以下を別プロセスとする。

```text
fr3_control_station_ui

fr3_app_manager

fr3_safety_supervisor

fr3_policy_runner

fr3_camera_service

fr3_recorder

move_group

controller_manager

robot_state_publisher
```

ros2_control plugin：

```text
fr3_streaming_controller
```

Franka公式：

```text
franka_hardware
franka_bringup
franka_gripper
franka_description
franka_fr3_moveit_config
```

Franka公式Jazzy構成では `franka_bringup` からFR3を起動でき、MoveIt用の `franka_fr3_moveit_config` も提供されている。

参考:
- https://frankarobotics.github.io/docs/doc/franka_ros2_jazzy/franka_bringup/doc/index.html

---

# 7. GUIプロセス

Executable:

```text
fr3_control_station_ui
```

GUIはRobot Controlに直接アクセスしない。

担当：

```text
Rendering
User Input
Camera View
Digital Twin
Telemetry
Configuration
Diagnostics
Event display
```

通信：

```text
ROS 2 topic
ROS 2 service
ROS 2 action
```

のみ。

---

# 8. QML / C++構造

```text
QML
 │
 ▼
ViewModel
 │
 ▼
Service / Repository
 │
 ▼
ROS 2 Client
```

主要ViewModel：

```text
RobotViewModel
ControlViewModel
SafetyViewModel
CameraViewModel
PolicyViewModel
PlanningViewModel
TelemetryViewModel
DiagnosticsViewModel
SettingsViewModel
EventViewModel
```

C++クラスは、

```cpp
Q_OBJECT
Q_PROPERTY(...)
Q_INVOKABLE
signals:
```

またはQtの推奨する、

```cpp
QML_ELEMENT
```

等でQMLへ公開する。

大量の `QQmlContext::setContextProperty()` に依存しない。

---

# 9. App Manager

Node:

```text
/fr3_app_manager
```

アプリケーション全体のControl Modeを管理する。

## Control Mode

```text
DISCONNECTED

CONNECTED

READY

MANUAL_JOINT

MANUAL_CARTESIAN

PLANNING

EXECUTING_PLAN

POLICY

STOPPING

FAULT
```

Mode transitionはApp Managerのみが決定する。

GUIやPolicy nodeが直接controllerを切り替えてはならない。

---

# 10. Control Arbitration

FR3へのCommand Sourceは同時に1つのみ許可する。

```text
NONE
MANUAL_JOINT
MANUAL_CARTESIAN
MOVEIT
POLICY
```

例えば、

```text
POLICY running
      +
MoveIt Execute
```

は禁止。

同様に、

```text
Manual Jog
      +
Policy
```

も禁止。

Controller切替はatomicなMode Transitionとして扱う。

---

# 11. Robot State

Realtime Robot Stateは最大1 kHzで取得する。

ただしGUIに1 kHzで送信する必要はない。

推奨：

```text
Robot Hardware     1000 Hz

Safety             200～1000 Hz

Logging            100～1000 Hz

Policy             必要レート

GUI Robot State    60 Hz

Telemetry graph    20～60 Hz
```

GUI向けにはdownsampleしたsnapshotをpublishする。

---

# 12. Realtime Controller

Plugin:

```text
fr3_streaming_controller
```

ros2_control ControllerInterfaceとして実装する。

リアルタイムupdate loop内では以下を禁止する。

```text
malloc/new
std::vector resize
file I/O
ROS blocking call
sleep
network request
model loading
printf
std::cout
mutex waiting
```

公式資料でも1 kHzループのread/writeは500 µs以内が推奨されている。

参考:
- https://frankarobotics.github.io/docs/troubleshooting.html

内部データは原則として、

```cpp
std::array
fixed-size Eigen matrix
realtime_tools::RealtimeBuffer
```

等を使用する。

---

# 13. Realtime性能目標

RT loop：

```text
Frequency       1000 Hz

Cycle           1 ms

Target update   < 200 µs

Read-update-write
                < 500 µs
```

500 µsはFranka公式推奨を基準とする。

GUI負荷が増大してもRT threadに影響しないこと。

PREEMPT_RT kernel使用を推奨する。

---

# 14. Policy Control

Policy Runner：

```text
fr3_policy_runner
```

Policy RunnerはFR3を直接制御しない。

```text
Camera
   │
   ▼
Policy inference
   │
   ▼
Policy Action
   │
   ▼
Safety Supervisor
   │
   ▼
Command Gateway
   │
   ▼
Realtime Controller
```

とする。

Policy Action例：

```text
Δx
Δy
Δz

Δroll
Δpitch
Δyaw

gripper
```

Policy側でRobot Controllerを直接呼ばない。

---

# 15. Policy Manifest

Policyモデルには必ずmetadataを付与する。

例：

```yaml
name: stack_cube_bc_v18
version: 18

robot:
  type: fr3

action:
  type: cartesian_delta
  dimensions: 7

observation:
  cameras:
    - camera_1
    - camera_2

control_rate_hz: 30

normalization:
  file: normalization.json

calibration:
  hash: abcdef123456

model:
  file: policy.onnx
  sha256: ...
```

Policyロード時、

```text
Robot type
Camera configuration
Action semantics
Input shape
Calibration version
Normalization
Model checksum
```

を検証する。

一致しない場合は実行禁止。

---

# 16. Policy Safety Watchdog

Policy Modeでは以下を監視する。

```text
Observation freshness

Command freshness

Inference timeout

NaN / Inf

Action bounds

Action discontinuity

Camera availability

Robot state freshness

Controller status
```

初期値例：

```text
Command timeout       100 ms
Observation timeout   150 ms
Camera timeout        200 ms
```

これらはconfigurableとする。

Timeout発生時：

```text
POLICY
  ↓
STOPPING
  ↓
READY / FAULT
```

**自動的にPolicyを再開しない。**

---

# 17. Safety Supervisor

Node:

```text
/fr3_safety_supervisor
```

チェック項目：

```text
Joint position

Joint velocity

Joint acceleration

Torque

Joint limit margin

Cartesian workspace

TCP velocity

TCP acceleration

Command delta

Command freshness

Collision/contact state

Controller state

Policy health

Camera health
```

Safety Supervisorを通過していないPolicy commandはFR3へ送らない。

---

# 18. Workspace Limit

Software Workspaceを設定する。

```yaml
workspace:

  x:
    min: ...
    max: ...

  y:
    min: ...
    max: ...

  z:
    min: ...
    max: ...
```

さらに必要に応じて、

```text
Forbidden Box

Forbidden Cylinder

Table plane

Camera collision region
```

を定義する。

---

# 19. Safety Layer

Safetyは多層化する。

```text
Layer 1
FR3 built-in safety

Layer 2
FCI / libfranka limits

Layer 3
ros2_control Controller limits

Layer 4
Safety Supervisor

Layer 5
Application Mode Arbitration

Layer 6
GUI confirmation
```

GUI上のSTOPは、

**安全規格上のEmergency Stopとして扱わない。**

名称は、

```text
STOP MOTION
```

または、

```text
ABORT
```

とする。

物理的なRobot Safety機構とは区別する。

---

# 20. FCI

FCI使用中はFR3を外部PCが排他的に制御するため、Desk/Appsと同時制御できない。

参考:
- https://frankarobotics.github.io/docs/overview.html

Version 0.1ではFCI有効化はDesk側で明示的に行うものとする。

Control Station起動時：

```text
FR3 reachable?

FCI ready?

ROS hardware connected?

Controller manager ready?
```

を確認する。

---

# 21. Manual Joint Jog

Joint Control画面から、

```text
J1
J2
J3
J4
J5
J6
J7
```

を操作する。

UI：

```text
[-]  Joint 1  [+]
[-]  Joint 2  [+]
...
```

操作はpress-and-hold方式。

ボタンを離した時点でCommandを停止する。

Jog speed：

```text
Slow
Normal
Fast
Custom
```

ただし最大値はconfigのSafety Limitで制限する。

---

# 22. Cartesian Jog

操作軸：

```text
X ±

Y ±

Z ±

Roll ±

Pitch ±

Yaw ±
```

Frame選択：

```text
BASE

TOOL
```

をサポートする。

Cartesian JogにはMoveIt Servoまたは専用streaming controllerを使用できるようBackend Interfaceを抽象化する。

GUIは実装方式を意識しない。

---

# 23. MoveIt 2

Motion Planning：

```text
franka_fr3_moveit_config
```

を利用する。

公式構成では、

```bash
ros2 launch franka_fr3_moveit_config moveit.launch.py \
  robot_ip:=<fci-ip>
```

により実機FR3向けMoveItを開始できる。

参考:
- https://frankarobotics.github.io/docs/doc/franka_ros2_jazzy/franka_fr3_moveit_config/doc/index.html

Control StationではRViz UIを埋め込まず、MoveItをbackendとして利用する。

理由：

```text
Qt dependency separation

UI design consistency

fault isolation

軽量化

独自UX
```

---

# 24. Pose Control

GUIから、

```text
X
Y
Z

Roll
Pitch
Yaw
```

またはQuaternionを指定する。

Flow：

```text
Target Pose

      ↓

MoveIt Plan

      ↓

Trajectory visualization

      ↓

User confirmation

      ↓

Execute
```

PlanとExecuteを分離する。

---

# 25. Named Pose

設定可能なPose：

```text
READY

OBSERVE

CAMERA_CALIBRATION

CUSTOM_1

CUSTOM_2
```

「HOME」という名称はロボット固有の意味と混同しやすいため、基本名称は、

```text
READY
```

を推奨する。

Named PoseもMoveItでcollision checkしてから実行する。

---

# 26. Gripper

Gripper abstraction：

```cpp
class EndEffectorInterface
```

を定義する。

初期実装：

```text
Franka Hand
```

将来的には、

```text
Custom Gripper

5-finger Hand

Vacuum Gripper
```

等へ交換可能にする。

GUIはEndEffector typeを意識しない。

---

# 27. Camera Service

Node：

```text
fr3_camera_service
```

Camera Driverとは分離可能とする。

Responsibilities：

```text
Capture

Timestamp

Calibration information

Frame synchronization

Health monitoring

Frame distribution
```

---

# 28. Camera Data Path

高解像度画像を、

```text
ROS message
→ copy
→ QImage
→ copy
→ QML
```

と何度もcopyする構成は避ける。

推奨構成：

```text
RealSense
   ↓
Camera Service
   ↓
Shared Frame Buffer
   ├── GUI Renderer
   └── Policy Runner
```

Frame Bufferは、

```text
ring buffer

latest-frame mailbox
```

方式とする。

GUI表示では古いframeをqueueしない。

常に最新画像を優先する。

---

# 29. Camera Rendering

Performance Path：

```text
Camera Buffer

     ↓

C++ QQuickItem

     ↓

Qt Scene Graph / GPU texture

     ↓

QML
```

可能な限り不要なCPU copyを避ける。

Camera表示目標：

```text
30 FPS以上

Display latency < 100 ms
```

を初期目標とする。

---

# 30. Digital Twin

3D表示項目：

```text
FR3 links

Gripper

TCP

Target TCP

Trajectory

Workspace

Objects

Camera poses
```

Robot transformは、

```text
TF2
   ↓
C++ Transform Model
   ↓
QML / 3D Scene
```

で更新する。

UI向け更新周波数：

```text
30～60 Hz
```

---

# 31. 3D Asset

Robot URDFをGUI runtimeで毎回複雑に解析する構造は避ける。

build/install時に、

```text
URDF
 + meshes
      ↓
GUI optimized asset
```

へ変換できる構成とする。

Runtimeでは主に、

```text
joint/link transforms
```

のみ更新する。

---

# 32. Main UI

基本レイアウト：

```text
┌───────────────────────────────────────────────────────────┐
│ FR3 CONTROL STATION                                      │
│ ● Robot  ● FCI  ● RT  ● Safety  ● Camera  ● Policy      │
├──────────┬──────────────────────────────────┬─────────────┤
│          │                                  │             │
│ Dashboard│                                  │ Robot State │
│ Manual   │          Main Workspace          │             │
│ Planning │                                  │ Controller  │
│ Policy   │                                  │             │
│ Camera   │                                  │ Safety      │
│ Logs     │                                  │             │
│ Settings │                                  │             │
│          │                                  │             │
├──────────┴──────────────────────────────────┴─────────────┤
│ Event / Warning / Error                                  │
└───────────────────────────────────────────────────────────┘
```

---

# 33. Persistent Status Bar

常に表示する。

```text
ROBOT
FCI
CONTROLLER
SAFETY
RT
CAMERA 1
CAMERA 2
POLICY
RECORDING
```

状態色：

```text
Normal

Warning

Fault

Inactive
```

ただし色だけで状態を表現せず、

```text
icon + text + color
```

を使用する。

---

# 34. Dashboard

Dashboardでは重要な情報のみ表示する。

```text
Robot connection

Safety status

Current control mode

3D Robot

Camera preview

EE pose

Policy state

Recording

Recent warning
```

詳細データは別画面へ移す。

---

# 35. Manual Control画面

```text
Joint Jog

Cartesian Jog

Gripper

Speed setting

Frame selection

Current Pose

Current Joint State
```

を表示。

---

# 36. Planning画面

```text
3D View

Current Pose

Goal Pose

Plan

Trajectory preview

Execute

Cancel
```

を配置する。

Execution中はGoal編集を禁止する。

---

# 37. Policy画面

表示：

```text
Model

Version

Model checksum

Inference backend

Inference rate

Inference latency

Observation age

Action

Policy state

Camera health

Safety status
```

Buttons：

```text
LOAD

VALIDATE

ARM

START

STOP
```

`LOAD → START` の一発操作は禁止。

---

# 38. Policy State Machine

```text
UNLOADED
   ↓
LOADED
   ↓
VALIDATED
   ↓
ARMED
   ↓
RUNNING
   ↓
STOPPED
```

異常時：

```text
RUNNING
   ↓
FAULT
```

FAULTからRUNNINGへの自動復帰は禁止。

---

# 39. Telemetry

グラフ：

```text
Joint q

Joint dq

Joint torque

External torque

TCP position

TCP velocity

External wrench

Inference latency

Control latency
```

GUIでは1 kHzすべてを描画しない。

Backendで、

```text
decimation
aggregation
ring buffer
```

を行う。

---

# 40. Logging

ログは3種類に分ける。

### Application Log

```text
spdlog / Qt logging
```

### Robotics Data

```text
rosbag2
```

### Event Log

```text
Mode changes

Safety event

Controller switch

Policy start/stop

Robot error

Camera error
```

Eventには、

```text
timestamp

severity

source

code

message

metadata
```

を記録する。

---

# 41. Recording

GUIから、

```text
Start Recording

Stop Recording
```

可能とする。

記録対象はprofileで選択。

例：

```yaml
record_profiles:

  policy_debug:

    - robot_state
    - joint_states
    - camera_1
    - camera_2
    - policy_action
    - safety_status
```

---

# 42. ROS 2 Interfaces

独自package：

```text
fr3_control_station_interfaces
```

Messages：

```text
RobotSummary.msg

SafetyStatus.msg

PolicyStatus.msg

CameraStatus.msg

ControlStatus.msg

Event.msg

PolicyCommand.msg
```

Services：

```text
SetControlMode.srv

LoadPolicy.srv

ValidatePolicy.srv

ArmPolicy.srv

StopMotion.srv

StartRecording.srv

StopRecording.srv
```

可能な限り標準ROS message/actionを利用し、不必要に独自interfaceを増やさない。

---

# 43. PolicyCommand

概念例：

```text
Header

sequence

source

control_mode

timestamp

ttl

Cartesian delta

gripper command
```

必ず、

```text
timestamp

sequence

TTL
```

を持たせる。

古いcommandをcontrollerが実行してはならない。

---

# 44. QoS

Robot State / Camera Preview：

```text
BEST_EFFORT
KEEP_LAST
depth = 1
```

最新値優先。

Safety Event / Mode Event：

```text
RELIABLE
```

Discrete command：

```text
Service / Action
```

Robot motion streamに巨大なReliable queueを作らない。

---

# 45. Configuration

```text
config/

  robot.yaml

  safety.yaml

  cameras.yaml

  control.yaml

  policy.yaml

  ui.yaml

  recording.yaml
```

例：

```yaml
robot:

  type: fr3

  ip: 172.16.0.3

  base_frame: fr3_link0

  tool_frame: fr3_hand
```

---

# 46. Config Validation

起動時にschema validationを行う。

以下を禁止する。

```text
invalid IP

min > max

negative timeout

unknown frame

unknown controller

duplicate camera ID

invalid Policy Action format
```

Configuration Error時はRobot Motionを開始しない。

---

# 47. Thread設計

GUI Process：

```text
Main Thread
  QML / UI

ROS Thread
  rclcpp executor

Camera/Frame Thread
  frame receiving

Worker Thread
  file / heavy processing
```

Qt Main Threadで、

```text
ROS spin

disk I/O

model load

heavy image processing
```

を実行しない。

---

# 48. ROS Executor

GUI内ROS clientは専用threadで、

```cpp
rclcpp::executors::MultiThreadedExecutor
```

または適切なExecutorを動作させる。

ROS callbackからQML objectを直接操作しない。

```text
ROS callback
   ↓
C++ data model
   ↓ signal
Qt event loop
   ↓
QML
```

とする。

---

# 49. Performance Budget

目標：

```text
UI FPS                 60 FPS

Camera                  30 FPS以上

3D                      60 FPS target

Robot UI update         60 Hz

Policy display          30～60 Hz

Telemetry               20～60 Hz

RT controller           1000 Hz
```

GUI負荷によってRealtime loop timingが変化してはならない。

---

# 50. Failure Handling

## GUI Crash

```text
GUI lost
   ↓
command timeout
   ↓
Robot motion stop
```

Robot ControllerがGUI processの生存を前提としてはいけない。

## Policy Crash

```text
Policy command timeout
   ↓
Safety Supervisor
   ↓
STOP
```

## Camera Loss

POLICY中：

```text
Camera timeout
   ↓
Policy Stop
```

Manual / MoveItについてはCamera必須でない場合継続可能。

## ROS Communication Loss

```text
Command timeout
   ↓
controller safe stop
```

## Robot Error

```text
FAULT
```

へ遷移。

User confirmationなしでMotionを再開しない。

---

# 51. Startup Flow

```text
Start Control Station

      ↓

Load config

      ↓

Validate config

      ↓

Start ROS

      ↓

Check robot connectivity

      ↓

Check FCI

      ↓

Check controller_manager

      ↓

Check Safety Supervisor

      ↓

Check cameras

      ↓

READY
```

一部componentが利用できない場合はDegraded Modeを許可する。

例：

```text
Camera unavailable

→ Manual Control可能

→ Policy Control不可
```

---

# 52. Shutdown Flow

```text
Stop active motion

      ↓

Disable command source

      ↓

Stop Policy

      ↓

Stop Recording

      ↓

Controller transition

      ↓

Shutdown ROS clients

      ↓

Close UI
```

アプリ終了によって突然command streamだけが消える設計にしない。

ただし異常終了時もcontroller watchdogが必ず停止させる。

---

# 53. Directory構成

```text
fr3_control_station_ws/

  src/

    fr3_control_station_ui/
      CMakeLists.txt
      package.xml

      src/
        main.cpp
        ros/
        viewmodels/
        models/
        services/
        rendering/

      qml/
        Main.qml

        pages/
          DashboardPage.qml
          ManualPage.qml
          PlanningPage.qml
          PolicyPage.qml
          CameraPage.qml
          TelemetryPage.qml
          DiagnosticsPage.qml
          SettingsPage.qml

        components/
          StatusBadge.qml
          SafetyBanner.qml
          CameraView.qml
          JointControl.qml
          PoseEditor.qml
          TelemetryChart.qml

        theme/
          Theme.qml
          Typography.qml
          Metrics.qml


    fr3_control_station_interfaces/

    fr3_app_manager/

    fr3_safety_supervisor/

    fr3_streaming_controller/

    fr3_policy_runner/

    fr3_camera_service/

    fr3_recorder/

    fr3_bringup/


  config/

  models/

  assets/

  scripts/

  tests/

  docs/
```

---

# 54. C++ Coding Rules

```text
C++20

RAII

smart pointers

const correctness

strong types where useful

no raw owning pointer

no exceptions across RT boundaries
```

Formatting：

```text
clang-format
```

Static analysis：

```text
clang-tidy
```

Sanitizer：

```text
ASan
UBSan
TSan
```

Realtime processではSanitizerによるtiming影響を考慮し、development buildで使用する。

---

# 55. QML Coding Rules

QMLはViewに限定する。

禁止：

```text
Robot control algorithm

Policy calculation

ROS logic

Safety calculation

filesystem logic

large JavaScript algorithms
```

QML componentは、

```text
presentation

layout

animation

input
```

へ限定する。

---

# 56. UI Design System

共通Design Token：

```text
Spacing

Radius

Typography

Elevation

Status color

Control size
```

をThemeとして定義する。

ページごとに直接pixel値やcolor値を乱立させない。

---

# 57. UI UX原則

Robot Control UIでは装飾性より、

```text
State visibility

Error visibility

Mode visibility

Predictability

Fast STOP access
```

を優先する。

特に、

```text
Manual

MoveIt

Policy
```

のどのControl Sourceが現在Robotを支配しているか常時明確に表示する。

---

# 58. Anti-pattern

以下の実装は禁止する。

```text
Qt Main Threadからlibfranka 1 kHz control

QMLからRobotへ直接command

Policyからlibfranka直接操作

複数Command Sourceの同時利用

Realtime loop内memory allocation

Realtime loop内logging

Realtime loop内network I/O

GUI updateのための1 kHz signal発行

Camera frameの無制限queue

FAULT後の自動Policy再開

アプリ起動直後の自動Robot motion
```

---

# 59. Testing

## Level 1

Unit Test

対象：

```text
State machine

Safety limits

Policy validation

Config parsing

Command arbitration

Coordinate transform
```

Framework：

```text
GoogleTest
```

---

# 60. Simulation Test

実機接続前に、

```text
use_fake_hardware:=true
```

でFR3 + MoveItを試験する。

Franka公式MoveIt packageもfake hardwareによるテストをサポートしている。

参考:
- https://frankarobotics.github.io/docs/doc/franka_ros2_jazzy/franka_fr3_moveit_config/doc/index.html

---

# 61. Integration Test

確認：

```text
UI → App Manager

App Manager → Safety

Safety → Controller

Policy → Safety

Camera → Policy

MoveIt → Controller
```

---

# 62. Fault Injection Test

必須テスト：

```text
Policy process kill

Camera disconnect

ROS topic loss

command timeout

Robot disconnect

GUI kill

Controller failure

NaN Policy output

extreme Policy output

stale timestamp

invalid calibration
```

すべて安全に停止できること。

---

# 63. Hardware Commissioning

実機テストは段階的に行う。

```text
1. State read only

2. Gripper only

3. Low-speed Joint Jog

4. Low-speed Cartesian Jog

5. MoveIt named pose

6. MoveIt arbitrary pose

7. Policy dry-run

8. Policy low-speed

9. Normal Policy
```

Policy Dry-runではRobot commandを送信せず、

```text
Policy output

Safety result

predicted target
```

のみ表示する。

---

# 64. Policy Shadow Mode

実装を推奨する。

```text
Robot:
Manual / MoveIt Control

Policy:
Inference only
```

Policyが出すActionをRobotに送らずTelemetryに表示する。

これにより実機Policy Debuggingを安全に行える。

---

# 65. Acceptance Criteria

Version 1.0では最低限以下を満たす。

```text
FR3接続状態表示

Joint State表示

EE Pose表示

Joint Jog

Cartesian Jog

Gripper

MoveIt Plan

MoveIt Execute

Camera ×2

Digital Twin

Policy Load

Policy Validate

Policy Run

Policy Stop

Safety Watchdog

Command Arbitration

Telemetry

rosbag Record

Error/Event Log
```

さらに、

```text
GUI crash

Policy crash

Camera disconnect

Command timeout
```

によって危険なmotionが継続しないこと。

---

# 66. MVP

最初の実装では機能を絞る。

```text
Phase 1

Robot connection

Robot State

Joint Jog

Cartesian Jog

Gripper

Camera

Safety status

Basic 3D

MoveIt Plan/Execute

Event log
```

---

# 67. Phase 2

```text
Policy Runner

Policy Manifest

Policy Shadow Mode

Policy Safety

rosbag recording

Telemetry

Calibration management
```

---

# 68. Phase 3

```text
Advanced Digital Twin

Trajectory visualization

GPU camera pipeline

Advanced diagnostics

Experiment manager

Dataset management

Policy comparison

Replay mode
```

---

# 69. 将来的な拡張

Architecture上、以下を追加可能にする。

```text
VLA

LBM

Teleoperation

SpaceMouse

Gamepad

VR / Vision Pro

Multi-camera

Force control

Custom gripper

Dual FR3

Simulation

Isaac Sim

Isaac Lab

MuJoCo
```

Application LayerとRobot Layerを分離することで、UIを大幅変更せずControllerやPolicyを交換できる設計とする。

---

# 70. 最終Architecture

```text
                       USER
                        │
                        ▼
             ┌────────────────────┐
             │     Qt / QML       │
             │       GUI          │
             └──────────┬─────────┘
                        │
                       ROS2
                        │
             ┌──────────▼─────────┐
             │    App Manager     │
             │ State / Arbitration│
             └────┬───────┬───────┘
                  │       │
         ┌────────┘       └─────────┐
         ▼                          ▼
 ┌───────────────┐          ┌───────────────┐
 │    MoveIt     │          │ Policy Runner │
 └───────┬───────┘          └───────┬───────┘
         │                          │
         │                  ┌───────▼────────┐
         │                  │ Camera Service │
         │                  └────────────────┘
         │
         └──────────┬───────────────┘
                    ▼
          ┌────────────────────┐
          │ Safety Supervisor  │
          └──────────┬─────────┘
                     ▼
          ┌────────────────────┐
          │ Command Arbitration│
          └──────────┬─────────┘
                     ▼
          ┌────────────────────┐
          │ ros2_control       │
          │ RT Controller      │
          │      1 kHz         │
          └──────────┬─────────┘
                     ▼
          ┌────────────────────┐
          │ franka_hardware    │
          └──────────┬─────────┘
                     ▼
                 libfranka
                     │
                     ▼
                    FR3
```

---

# 71. Architecture Decision

本プロジェクトでは以下を基本方針として固定する。

**UI**

```text
Qt Quick / QML
```

**Application**

```text
C++20
```

**Communication**

```text
ROS 2
```

**Motion Planning**

```text
MoveIt 2
```

**Realtime Robot Control**

```text
ros2_control
franka_hardware
libfranka
```

**AI**

```text
Dedicated Policy Process
```

**Camera**

```text
Dedicated Camera Process
```

**Safety**

```text
Independent Safety Supervisor
+
Realtime controller watchdog
```

最大の設計原則は、

> GUI・AI・Cameraが停止しても、Realtime Robot Controllerが危険なCommandを継続しないこと。

である。

また、

> Robotを制御できるCommand Sourceは常に1つだけであること。

をシステム全体のInvariantとする。

この2条件をFR3 Control Stationの最優先Architecture Requirementとする。
