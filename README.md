# 🤖 SEBOT T710 智能服务机器人系统
基于 ROS 1 与 SEBOT T710 实机平台的自主探索建图、订单识别、零件抓取与配送一体化项目。

<p align="left">
  <img src="https://img.shields.io/badge/ROS-1-22314E?logo=ros" />
  <img src="https://img.shields.io/badge/C++-14-00599C?logo=cplusplus" />
  <img src="https://img.shields.io/badge/Python-3.x-3776AB?logo=python" />
  <img src="https://img.shields.io/badge/OpenCV-Vision-5C3EE8?logo=opencv" />
  <img src="https://img.shields.io/badge/PaddlePaddle-EdgeBoard-0062B1" />
</p>

## 🎬 项目演示

[![Bilibili](https://img.shields.io/badge/Bilibili-观看完整演示-00A1D6?logo=bilibili&logoColor=white)](https://www.bilibili.com/video/BV1tzgR6hEmd/)

▶ **演示视频：**  
https://www.bilibili.com/video/BV1tzgR6hEmd/

---

## 📖 项目简介

本项将：

```text
环境感知
   ↓
自主建图
   ↓
定位导航
   ↓
订单识别
   ↓
零件识别
   ↓
机械臂抓取
   ↓
物料配送
   ↓
结果汇总
```

串联成一套完整的 ROS 自主任务系统。

系统主要由三部分组成：

- `sebot_factory`：任务状态机、订单识别、抓取、放置与结算
- `sebot_ros_kits`：机器人底盘、导航、SLAM、传感器与驱动
- `sebot_ros_stdr`：STDR 仿真与导航测试

---

## 🗺️ 任务流程

### 1. 自主探索建图

机器人从起始区域出发，通过激光雷达感知环境，并基于 Frontier 自动寻找尚未探索的区域。

```text
启动机器人
   ↓
激光雷达扫描
   ↓
Odometry + IMU 融合
   ↓
GMapping SLAM
   ↓
Frontier Search
   ↓
Move Base 自主探索
   ↓
探索完成
   ↓
返回起始区
   ↓
保存地图
```

自主探索节点会搜索已知自由区域与未知区域之间的 Frontier，并将合适的边界区域作为下一导航目标。

如果某个目标长时间无法到达，会加入黑名单，避免机器人反复尝试同一失败位置。

---

### 2. 自主配送服务

地图建立完成后，系统切换到自主配送模式：

```text
AMCL 初始定位
   ↓
前往工作台
   ↓
识别订单
   ↓
确认所需零件
   ↓
导航至取件台
   ↓
搜索目标零件
   ↓
机械臂抓取
   ↓
返回对应工作台
   ↓
放置零件
   ↓
处理下一零件 / 下一工作台
   ↓
前往结算区
   ↓
显示物料清单
```

整个流程由 `Factory` 状态机自动调度。

---

## 🧠 系统架构

```text
                 ┌────────────────────┐
                 │   Factory 状态机    │
                 └─────────┬──────────┘
                           │
          ┌────────────────┼────────────────┐
          ↓                ↓                ↓
      Navigation        Vision         Manipulation
          │                │                │
      AMCL /          EdgeBoard         Talon Arm
     move_base        Paddle AI        Gripper
          │                │                │
          └────────────────┼────────────────┘
                           ↓
                    SEBOT T710 Robot
                           ↓
        ┌──────────────────┼──────────────────┐
        ↓                  ↓                  ↓
     RPLIDAR            Camera           Odom / IMU
```

---

## ✨ 核心功能

| 模块 | 功能 |
| --- | --- |
| 自主建图 | GMapping + Frontier Search 自动探索 |
| 定位导航 | AMCL + Move Base |
| 路径规划 | 全局 / 局部规划与动态避障 |
| 订单识别 | 工作台订单牌及零件类别检测 |
| 零件识别 | Nut / Screw / PCB / Block / Tape |
| 多帧确认 | 多次视觉采样过滤偶发误检 |
| 底盘精定位 | 激光距离 + 图像位置 PID 调整 |
| 机械臂对准 | ArUco / 颜色区域辅助视觉定位 |
| 抓取与放置 | 机械臂伸展、夹取、抬升、收缩与放置 |
| 多任务调度 | 多工作台、多零件连续配送 |
| 结果汇总 | 自动生成最终物料清单 |
| 语音播报 | ROS `/audio` 任务状态提示 |

---

## 👁️ 订单与零件识别

视觉模块基于 EdgeBoard AI 推理框架实现。

主要流程：

```text
Camera Image
    ↓
Resize 320×320
    ↓
RGB / Normalize
    ↓
Paddle EdgeBoard NNA
    ↓
ONNX Post Processing
    ↓
NMS
    ↓
Detection Results
```

当前识别类别：

```text
order
nut
screw
pcb
block
tape
```

订单识别阶段会首先检测 `order` 区域，再仅统计订单框内部的零件目标。

同时采用多帧采样机制，对检测结果进行出现次数统计和排序，减少单帧误识别对任务的影响。

---

## 🎯 工作台精定位

机器人通过 Move Base 到达工作台后，还会进行局部位置修正。

```text
激光左右距离差
      ↓
朝向 PID

激光平均距离
      ↓
前后距离 PID

订单牌图像位置
      ↓
横向位置 PID
```

这样可以让机器人从“导航到工作台附近”进一步调整到适合视觉识别和机械操作的位置。

---

## 🦾 零件抓取

抓取阶段采用“底盘粗定位 + 机械臂视觉精定位”的两级方式：

```text
导航至取件台
      ↓
AI 搜索目标零件
      ↓
底盘前后 / 横向 / 朝向调整
      ↓
机械臂伸展
      ↓
切换机械臂 RGB 相机
      ↓
ArUco / 颜色区域定位
      ↓
机械爪 PD 对准
      ↓
底盘低速靠近
      ↓
夹爪闭合
      ↓
抬升物料
      ↓
机械臂收缩
      ↓
底盘退出抓取区域
```

机械臂通过 `/dev/talon` 串口控制，支持：

```text
ACTION_RES  → 复位
ACTION_EXT  → 伸展
ACTION_CUR  → 收缩
ACTION_PUT  → 放置
ACTION_DIY  → 自定义动作
```

---

## 📦 零件放置

机器人返回对应工作台后，根据当前零件序号选择不同放置位置：

```text
第 1 个零件 → 中间
第 2 个零件 → 左侧
第 3 个零件 → 右侧
```

单次放置流程：

```text
调整工作台距离
   ↓
机械臂伸展
   ↓
选择放置位置
   ↓
夹爪松开
   ↓
机械爪抬升
   ↓
底盘后退
   ↓
机械臂收缩
```

完成当前工作台所有零件后，系统自动切换到下一工作台。

---

## 🧾 最终物料清单

所有配送任务完成后，机器人自动导航至结算区。

系统根据任务过程中保存的订单数据生成最终清单。

物料编号映射：

| 零件 | 编号 |
| --- | --- |
| Screw | G111 |
| Nut | G112 |
| PCB | G113 |
| Block | G114 |
| Tape | G115 |

系统会根据工作台数量动态生成对应的结算界面。

---

## 🛠 技术栈

**机器人系统**

`Ubuntu` · `ROS 1` · `catkin` · `TF`

**导航与建图**

`GMapping` · `Frontier Search` · `AMCL` · `move_base` · `robot_pose_ekf`

**视觉**

`OpenCV` · `ArUco` · `PaddlePaddle EdgeBoard` · `ONNX Runtime`

**机器人控制**

`RPLIDAR` · `Odometry` · `IMU` · `PID` · `libserial`

**机械操作**

`Talon Arm` · `Gripper Control` · `MoveIt`

**开发**

`C++14` · `Python 3`

---

## 📁 项目结构

```text
sebot-t710-competition/
│
├── sebot_factory/
│   └── src/
│       ├── sebot_factory/
│       │   ├── src/
│       │   │   ├── factory.cpp
│       │   │   ├── confirm.cpp
│       │   │   ├── picking.cpp
│       │   │   └── summary.cpp
│       │   ├── include/
│       │   ├── launch/
│       │   ├── res/
│       │   └── unit/
│       │
│       └── sebot_marking/
│
├── sebot_ros_kits/
│   └── src/
│       ├── sebot_driver/
│       ├── sebot_navigation/
│       ├── sebot_robot/
│       ├── sebot_slam/
│       ├── sebot_speech/
│       └── sebot_visions/
│
├── sebot_ros_stdr/
│   └── src/
│       └── sebot_stdr/
│
├── .gitattributes
└── README.md
```

---

## 🚀 快速开始

### 编译机器人工作空间

```bash
cd sebot_ros_kits
catkin_make
source devel/setup.bash
```

### 编译比赛任务工作空间

```bash
cd sebot_factory
catkin_make
source devel/setup.bash
```

---

### 启动自主建图

```bash
source sebot_ros_kits/devel/setup.bash
roslaunch sebot_slam sebot_auto_gmapping.launch
```

保存地图：

```bash
roslaunch sebot_slam sebot_map_save.launch
```

---

### 启动完整配送任务

```bash
source sebot_ros_kits/devel/setup.bash
source sebot_factory/devel/setup.bash

roslaunch sebot_factory sebot_factory.launch
```

---

## 🔄 主状态机

完整任务由：

```text
FACTORY_STEP_START
        ↓
FACTORY_STEP_INIT
        ↓
FACTORY_STEP_DELIVERY
        ↓
FACTORY_STEP_SUMMARY
```

组成。

配送过程中：

```text
工作台
  ↓
Confirm
  ↓
订单列表
  ↓
取件台
  ↓
Picking
  ↓
抓取零件
  ↓
返回工作台
  ↓
放置
  ↓
还有零件？
 ├─ Yes → 继续配送
 └─ No  → 下一工作台
```

---

## 🔌 主要设备

| 设备 | 接口 |
| --- | --- |
| 底盘控制器 | `/dev/robot` |
| RPLIDAR | `/dev/rplidar` |
| Talon 机械臂 | `/dev/talon` |
| 取件视觉相机 | `/dev/deepCamera` |
| 机械臂 RGB 相机 | `/dev/rgbCamera` |

部分路径仍保留比赛现场部署环境配置，迁移到其他设备时需要根据实际环境修改。

---

## 📌 项目特点

本项目不是单一算法 Demo，而是一套完整的真实机器人任务工程。

它将：

```text
SLAM
+
Navigation
+
Computer Vision
+
AI Inference
+
Robot Control
+
Manipulator Control
+
Task State Machine
```

集成到同一个 ROS 系统中，实现服务机器人从环境感知到最终物料配送的完整自主闭环。

---

## ⚠️ 说明

本仓库主要用于：

- 项目展示
- ROS 服务机器人学习
- SLAM / Navigation 实践
- 视觉识别与机械操作研究
- 智能服务机器人任务流程复现

部分设备驱动、模型文件和路径配置依赖原比赛机器人环境，迁移到其他平台时需要重新进行设备映射、参数标定和环境配置。

