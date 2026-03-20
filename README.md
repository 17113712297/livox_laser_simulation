# livox_laser_simulation

本文档说明基于原始仓库 [Mid360_simulation_plugin](https://github.com/fratopa/Mid360_simulation_plugin) 所做的修改内容。

---

## 原始仓库

> https://github.com/fratopa/Mid360_simulation_plugin

---

## 修改说明

### 1. 依赖的消息类型变更

| 项目 | 原始版本 | 修改后版本 |
|------|----------|------------|
| CustomMsg 消息包 | `livox_laser_simulation::CustomMsg` | `livox_ros_driver2::CustomMsg` |
| 头文件引用 | `#include <livox_laser_simulation/CustomMsg.h>` | `#include <livox_ros_driver2/CustomMsg.h>` |

修改后与 `livox_ros_driver2` 功能包保持一致，方便直接对接使用 `livox_ros_driver2` 的下游算法（如 FAST-LIO2、Point-LIO 等）。

---

### 2. 发布器架构变更

**原始版本**采用**单一发布器 + 类型枚举**的方式，通过 SDF 参数 `publish_pointcloud_type` 在编译/配置时选择发布哪一种消息类型，每次只发布一种：

```cpp
// 原始：单一发布器，通过枚举切换类型
ros::Publisher rosPointPub;

enum PointCloudType {
    SENSOR_MSG_POINT_CLOUD = 0,
    SENSOR_MSG_POINT_CLOUD2_POINTXYZ = 1,
    SENSOR_MSG_POINT_CLOUD2_LIVOXPOINTXYZRTLT = 2,
    livox_laser_simulation_CUSTOM_MSG = 3,
};
```

**修改后版本**改为**三个独立发布器**，每帧数据同时发布全部三种消息类型，无需配置类型选项：

```cpp
// 修改后：三个独立发布器，同时发布所有类型
ros::Publisher rosPointCloudPub;   // sensor_msgs::PointCloud
ros::Publisher rosPointCloud2Pub;  // sensor_msgs::PointCloud2
ros::Publisher rosCustomMsgPub;    // livox_ros_driver2::CustomMsg
```

---

### 3. SDF 参数变更

**原始版本**所需 SDF 参数：

| 参数名 | 说明 |
|--------|------|
| `ros_topic` | 唯一话题名（所有类型共用） |
| `frameName` | 坐标系名称 |
| `publish_pointcloud_type` | 发布类型枚举值（0\~3） |

**修改后版本**所需 SDF 参数（均有默认值，可选填）：

| 参数名 | 默认值 | 说明 |
|--------|--------|------|
| `pointcloud_topic` | `/livox/pointcloud` | `sensor_msgs::PointCloud` 话题 |
| `pointcloud2_topic` | `/livox/pointcloud2` | `sensor_msgs::PointCloud2` 话题 |
| `custom_msg_topic` | `/livox/lidar` | `livox_ros_driver2::CustomMsg` 话题 |
| `frameName` | `livox` | 坐标系名称 |

移除了 `publish_pointcloud_type` 和 `ros_topic` 参数，不再需要指定发布类型。

---

### 4. `OnNewLaserScans` 逻辑变更

原始版本使用 `switch` 语句，每帧只调用一个发布函数：

```cpp
// 原始版本
switch (publishPointCloudType) {
    case SENSOR_MSG_POINT_CLOUD:            PublishPointCloud(points_pair);      break;
    case SENSOR_MSG_POINT_CLOUD2_POINTXYZ:  PublishPointCloud2XYZ(points_pair);  break;
    case livox_laser_simulation_CUSTOM_MSG: PublishLivoxROSDriverCustomMsg(...);  break;
    // ...
}
```

修改后每帧同时调用全部三个发布函数：

```cpp
// 修改后版本
PublishPointCloud(points_pair);
PublishPointCloud2XYZ(points_pair);
PublishLivoxROSDriverCustomMsg(points_pair);
```

---

## 安装与使用

###### 3. livox_laser_simulation

1. **介绍**：这是一个用于 Gazebo 仿真的 Livox Mid-360 激光雷达插件，支持 ROS Noetic 和 Gazebo 11，并修正了原始插件中的点云畸变问题。

2. **安装**：

    ```bash
    # 在 src 目录下
    git clone https://github.com/fratopa/Mid360_simulation_plugin.git

    # 将 Mid360_simulation_plugin 文件夹下的 livox_laser_simulation 文件夹移动到 src 目录下
    mv Mid360_simulation_plugin/livox_laser_simulation .
    # 删除其余部分
    rm -rf Mid360_simulation_plugin

    # 回到工作空间根目录
    cd /home/dio/Project/IndoorUavInspection/catkin_ws
    source /opt/ros/noetic/setup.sh
    # 单独编译此功能包（这样快）
    catkin_make --pkg livox_laser_simulation -DCMAKE_BUILD_TYPE=Release
    source devel/setup.bash
    # 测试启动
    roslaunch livox_laser_simulation test_pattern.launch
    ```

3. **注意**：从 `Mid360_simulation_plugin.git` 仓库中仅需要 `livox_laser_simulation` 文件夹，将其放到 `catkin_ws/src` 目录下，其余部分可以删除。

