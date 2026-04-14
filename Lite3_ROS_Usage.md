# Lite3_ROS 使用文档

本文档用于说明如何在板子上启动 `Lite3_ROS` 的 UDP 转 ROS 2 桥接节点，并完成机器狗的基础控制、模式切换、心跳保持和调试。当前流程基于实机验证整理，**推荐顺序为：启动桥接 → 给心跳 → 切换 Navi Mode → 发送 `/cmd_vel`**。

## 1. 适用场景

适用于以下场景：

* 板子已经安装好 **ROS 2 Humble**
* `Lite3_ROS` 已经成功编译
* 板子通过网线连接机器狗，且网络已打通
* 机器狗运动主机可通过 `192.168.1.120` 访问

---

## 2. 前置条件

### 2.1 工作区准备

假设工作区路径为：

```bash
~/lite3_ros_ws
```

每次新开终端后，必须先执行：

```bash
source /opt/ros/humble/setup.bash
source ~/lite3_ros_ws/install/setup.bash
```

如果不执行上述命令，可能会出现如下错误：

```text
The passed message type is invalid
```

这通常意味着当前终端没有加载你的工作区环境，系统无法识别自定义消息类型，例如：

```text
transfer_interfaces/msg/MotionSimpleCMD
```

### 2.2 网络要求

板子连接机器狗的网卡需要和机器狗处于同一**网段**。

* 机器狗运动主机 IP：`192.168.1.120`
* 板子连接机器狗的网口 IP：建议设为 `192.168.1.102/24`

临时配置板子网口示例：

```bash
sudo ip addr flush dev eth1
sudo ip addr add 192.168.1.102/24 dev eth1
sudo ip link set eth1 up
```

测试连通性：

```bash
ping -c 4 192.168.1.120
```

如果 `ping` 不通，后续 ROS 2 桥接和控制通常也不会正常工作。

---

## 3. 核心控制流程

经过实机验证，推荐顺序为：

**启动桥接 → 给心跳 → 切 Navi Mode → 发 `/cmd_vel`**

建议分别在不同终端中执行下面操作。

### 步骤 1：启动桥接节点（终端 1）

启动 ROS 2 与 UDP 转换节点：

```bash
source /opt/ros/humble/setup.bash
source ~/lite3_ros_ws/install/setup.bash
ros2 launch transfer transfer_launch.py
```

启动后可用以下命令查看当前话题：

```bash
ros2 topic list
```

正常情况下，应能看到诸如 `/cmd_vel`、`/simple_cmd`、`/imu/data`、`/joint_states`、`/leg_odom`、`/leg_odom2` 等相关话题。
是否已经有数据流，还取决于机器狗当前状态、网络配置和回传配置。

### 步骤 2：持续发送心跳（终端 2）

在切换模式或保持控制链路时，需要持续发送心跳，避免控制通道失效。
**这一步应先于切换 Navi Mode。**

```bash
source /opt/ros/humble/setup.bash
source ~/lite3_ros_ws/install/setup.bash
ros2 topic pub -r 2 /simple_cmd transfer_interfaces/msg/MotionSimpleCMD "{cmd_code: 0x21040001, size: 0, type: 0}"
```

说明：

* 发送频率：`2 Hz`
* 请保持这个终端持续运行，不要关闭

### 步骤 3：切换到 Navi Mode（终端 3）

在心跳已经持续发送的前提下，发送单次命令切换到导航模式：

```bash
source /opt/ros/humble/setup.bash
source ~/lite3_ros_ws/install/setup.bash
ros2 topic pub --once /simple_cmd transfer_interfaces/msg/MotionSimpleCMD "{cmd_code: 0x21010C03, size: 0, type: 0}"
```

### 步骤 4：发送速度指令控制运动（终端 4）

进入 Navi Mode 后，即可通过 `/cmd_vel` 发送基础运动控制命令。

小幅向前测试：

```bash
source /opt/ros/humble/setup.bash
source ~/lite3_ros_ws/install/setup.bash
ros2 topic pub -r 10 -t 5 /cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.05, y: 0.0, z: 0.0}, angular: {x: 0.0, y: 0.0, z: 0.0}}"
```

发送停止命令：

```bash
ros2 topic pub --once /cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.0, y: 0.0, z: 0.0}, angular: {x: 0.0, y: 0.0, z: 0.0}}"
```

说明：

* `linear.x`：前后运动
* `linear.y`：侧向运动
* `angular.z`：偏航旋转
* 对于四足机器人，`linear.z`、`angular.x`、`angular.y` 一般不作为常规导航控制量使用

---

## 4. 辅助功能与测试

### 4.1 Stand / Sit 基础切换

如需进行基础站立/坐下切换，可发送：

```bash
ros2 topic pub --once /simple_cmd transfer_interfaces/msg/MotionSimpleCMD "{cmd_code: 0x21010202, size: 0, type: 0}"
```

具体效果以机器狗当前固件定义为准。

### 4.2 键盘控制

如果已安装 `teleop_twist_keyboard`，可以通过键盘直接发布 `/cmd_vel`：

```bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

这适合做基础联调，例如：

* 验证 `/cmd_vel` 是否正常发送
* 验证机器人是否响应速度指令
* 替代手工输入 `ros2 topic pub`

---

## 5. 调试方法

### 5.1 检查系统话题与数据

查看当前所有话题：

```bash
ros2 topic list
```

检查关键数据是否有回传：

```bash
ros2 topic echo /imu/data
ros2 topic echo /leg_odom
ros2 topic echo /leg_odom2
ros2 topic echo /joint_states
```

如果这些话题长期没有数据，优先检查：

* 板子与机器狗是否处于同一网段
* 板子是否能 `ping` 通 `192.168.1.120`
* 机器狗内部配置的**数据回传目标 IP** 是否设成了板子的当前 IP

### 5.2 抓取底层 UDP 包

用于调试底层 UDP 通信是否畅通：

```bash
sudo tcpdump -i any port 43893 -vvv -x
```

该命令可用于确认：

* 是否有 UDP 包进出
* 机器狗与板子之间是否正在进行底层通信

---

## 6. 常见问题（FAQ）

### 6.1 报错：`The passed message type is invalid`

原因通常是当前终端没有 source 工作区环境。

解决方法：

```bash
source /opt/ros/humble/setup.bash
source ~/lite3_ros_ws/install/setup.bash
```

### 6.2 能启动 `transfer_launch.py`，但传感器话题一直没数据

优先排查以下问题：

1. 板子连接机器狗的网卡是否配置到了 `192.168.1.x` 网段
2. 是否能正常 `ping 192.168.1.120`
3. 机器狗内部配置的**数据回传目标 IP** 是否为板子的当前 IP

### 6.3 已经发送了 `/cmd_vel`，但机器人不动

优先检查：

* 是否已经先启动心跳
* 是否已经切换到 **Navi Mode**
* 是否存在急停、硬件保护、模式锁定等情况
* `/cmd_vel` 是否真的被桥接节点订阅并转发

---

## 附录：最小可用命令速查表

```bash
# 1. 启动桥接
ros2 launch transfer transfer_launch.py

# 2. 给心跳（新终端保持运行）
ros2 topic pub -r 2 /simple_cmd transfer_interfaces/msg/MotionSimpleCMD "{cmd_code: 0x21040001, size: 0, type: 0}"

# 3. switch stands/sit
ros2 topic pub --once /simple_cmd transfer_interfaces/msg/MotionSimpleCMD "{cmd_code: 0x21010202, size: 0, type: 0}"

# 4. 切到 Navi Mode
ros2 topic pub --once /simple_cmd transfer_interfaces/msg/MotionSimpleCMD "{cmd_code: 0x21010C03, size: 0, type: 0}"

# 5. 控制移动（前移）
ros2 topic pub -r 10 -t 5 /cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.05, y: 0.0, z: 0.0}, angular: {x: 0.0, y: 0.0, z: 0.0}}"

# 6. 停止移动
ros2 topic pub --once /cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.0, y: 0.0, z: 0.0}, angular: {x: 0.0, y: 0.0, z: 0.0}}"
```
