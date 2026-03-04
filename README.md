# detect_armor_ros2
<br>
<div>
    <img alt="ROS 2" src="https://img.shields.io/badge/ROS%202-Jazzy-22314E?logo=ros">
    <img alt="OpenCV" src="https://img.shields.io/badge/OpenCV-4-5C3EE8?logo=opencv">
    <img alt="C++" src="https://img.shields.io/badge/C%2B%2B-17-00599C?logo=c%2B%2B">
</div>
<div>
    <img alt="platform" src="https://img.shields.io/badge/platform-Ubuntu%2024.04-E95420?logo=ubuntu">
</div>
<div>
    <a href="https://github.com/luomojim/detect_armor_ros2" target="_blank">
  <img alt="GitHub Repo" src="https://img.shields.io/badge/GitHub-Repository-181717?logo=github">
</a>
    <img alt="license" src="https://img.shields.io/badge/license-MIT-2ea44f">
</div>
<br>

基于 ROS 2 Jazzy + OpenCV + cpp 的装甲板检测与位姿解算工程。从视频流中检测装甲板，进行 PnP 位姿求解，并发布 ROS 话题与 TF 供可视化和其他节点使用。**如果该项目对你有帮助**

![show](docs/show.gif)

## 项目功能

- 装甲板检测：轮廓提取、灯条筛选、左右灯条匹配
- 位姿解算：通过 `solvePnP` 获取旋转向量与平移向量
- 姿态输出：计算 `yaw/pitch/distance`
- 角度滤波：对偏航角变化进行卡尔曼滤波，输出 `kf_yaw`
- 信息发布：发布 `armor_info` 话题、`center` 话题和 `camera_frame -> armor_frame` TF
- 可视化：使用 RViz 联动显示

## 仓库结构

```text
detect_armor_robomaster/
├── src/
│   ├── armor_interfaces/         # 自定义消息包（ArmorInfo.msg）
│   ├── armor_tracker_ros2/       # 检测与追踪节点
│   │   ├── src/main.cpp          # 程序的主逻辑
│   │   ├── include/*.hpp         # 检测/滤波/绘图等模块声明
│   │   ├── asset/armor.mp4       # 默认测试视频
│   │   ├── launch/armor_tracker.launch.py # 启动文件
│   │   ├── rviz/armor_tracker.rviz # rviz预设文件
│   │   └── urdf/armor.urdf       # 装甲板urdf描述文件
│   └── armor_info_subscriber/    # 订阅 armor_info 的示例节点
└── docs/  # 文档
```

## 运行环境

- Ubuntu24.04
- ROS 2 Jazzy
- C++17
- OpenCV 4
- colcon + ament

## 构建

在仓库根目录执行：

```bash
colcon build
source /opt/ros/jazzy/setup.bash
source install/setup.bash
```
如果使用zsh
``` zsh
colcon build
source /opt/ros/jazzy/setup.zsh
source install/setup.zsh
```

## 运行

### 使用launch启动

```bash
ros2 launch armor_tracker_ros2 armor_tracker.launch.py
```

该 launch 会同时启动：

- `armor_tracker_node`
- `static_transform_publisher`（`map -> camera_frame`,因为本项目不确定相机坐标，默认假设为(0,0,1)，即往上抬高1m）
- `robot_state_publisher`
- `armor_info_subscriber`
- `rviz2`

### 分别启动

```bash
ros2 run armor_tracker_ros2 armor_tracker_node
ros2 run armor_info_subscriber armor_info_subscriber
```

## 程序节点说明

本项目主要由 5 个 ROS2 节点协同工作，推荐通过 `armor_tracker.launch.py` 一起启动。

### armor_tracker_node

- 可执行文件：`armor_tracker_ros2/armor_tracker_node`
- 发布内容：
  - `/armor_info`：`armor_interfaces/msg/ArmorInfo`
  - `/center`：`geometry_msgs/msg/PointStamped`
  - TF：`camera_frame -> armor_frame`

### armor_info_subscriber

- 可执行文件：`armor_info_subscriber/armor_info_subscriber`
- 订阅内容：
  - 订阅 `/armor_info`
  - 在终端打印 `yaw/kf_yaw/pitch/distance`

### static_transform_publisher（静态TF）

- 可执行文件：`tf2_ros/static_transform_publisher`
- 主要内容：
  - 发布 `map -> camera_frame` 的转换信息
- 启动参数：
  - 平移：`(0, 0, 1)`，旋转：`(0, 0, 0)`（欧拉角）
- 说明：
  - 用于给 RViz 和 TF 树提供世界系到相机系的连接

### robot_state_publisher（URDF发布）

- 可执行文件：`robot_state_publisher/robot_state_publisher`
- 内容：
  - 读取 `src/armor_tracker_ros2/urdf/armor.urdf`
  - 发布 URDF 描述中的连杆/关节 TF
- 作用：
  - 让模型在 RViz 中正确显示并连接到 TF 树

### 节点图

下面是项目节点关系示意图：

![node graph](docs/node_graph.png)

### TF树

下面是项目 TF 树示意图：
![tf tree](docs/tf_tree.jpg)

## 话题与消息

### 话题

- `/armor_info`（`armor_interfaces/msg/ArmorInfo`）：使用话题通信来发送装甲板的yaw,kf_yaw(卡尔曼滤波后),pitch和distance
- `/center`（`geometry_msgs/msg/PointStamped`）：发送根据装甲板解算出来的旋转中心坐标

### ArmorInfo 字段

```text
float64 yaw
float64 kf_yaw
float64 pitch
float64 distance
```

## 坐标系说明

`main.cpp` 中使用了 OpenCV 坐标系到 ROS 坐标系的转换矩阵：

```cpp
cv_to_ros(0.0, 0.0, 1.0,
          -1.0, 0.0, 0.0,
          0.0, -1.0, 0.0)
```

含义为：

- `x_ros = z_cv`
- `y_ros = -x_cv`
- `z_ros = -y_cv`

用于将 PnP 解算得到的旋转和平移从 OpenCV 相机坐标转换到 ROS 坐标并发布 TF。

## 算法流程简述

1. 读取视频帧
2. 通道分离与阈值处理
3. 轮廓检测与灯条筛选
4. 灯条匹配装甲板，提取角点
5. `solvePnP` 求解旋转/平移向量
6. 计算 `yaw/pitch/distance`
7. 卡尔曼滤波估计 `kf_yaw`
8. 计算并发布旋转中心，发布话题与 TF

## 注意事项

- 当前 `armor_tracker_node` 在 `main.cpp` 中通过绝对路径读取视频：
  - `/home/luomo/Documents/GitHub/detect_armor_robomaster/src/armor_tracker_ros2/asset/armor.mp4`
- 请同步修改该路径读取。

## 参考文档

- [轴角表示、欧拉角、四元数以及罗德里格旋转公式](https://zhuanlan.zhihu.com/p/639468419)
- [ROS教程【ROS2理论与实践】](https://www.bilibili.com/video/BV1VB4y137ys/?share_source=copy_web&vd_source=c3949dd1f7e7b94fa3bad523d2316b9a)
- [本人的ros2学习笔记](https://luomokomorebi.top/)
