# Automated Bridge

## Background and Purpose

`ros_gz_bridge` currently requires most bridges to be declared explicitly through command line arguments or YAML configuration. This works for small simulations, but it does not scale well for multi-robot worlds. Each robot can expose command, state, TF, sensor, and service interfaces, which can result in long and repetitive bridge configuration files.

The purpose of automated bridge support is to let `ros_gz_bridge` discover compatible ROS 2 and Gazebo interfaces at runtime and create bridges automatically. If Gazebo exposes a topic or service, ROS has a matching endpoint with a compatible type, and `ros_gz_bridge` already has a known conversion for that type pair, the bridge can be created without a manual YAML entry.

This feature is intentionally opt-in. Existing YAML-based bridge configuration remains fully supported and keeps priority over automated discovery.

## Approach and methods

## Demo and Tests

### 1.Behavior test

#### 1.1 Overview

A [test package](https://github.com/azeey/gsoc2026_multirobot/tree/main/bridge/auto_bridge_test) is provided to validate automated bridge creation in `ros_gz_bridge`. The package contains a simple ROS 2 test node (`ros_pub_sub_node`), a Gazebo test world, and a launch file.

The test verifies that `ros_gz_bridge` can create topic and service bridges automatically when matching ROS endpoints and Gazebo interfaces are present. It covers `GZ_TO_ROS`, `ROS_TO_GZ`, and ROS-to-Gazebo service bridge creation.

#### 1.2 Content

##### 1.2.1 Launch file

The [launch file](https://github.com/azeey/gsoc2026_multirobot/blob/main/bridge/auto_bridge_test/launch/auto_bridge_test.launch.py) is responsible for setting up the full behavior test environment. It:

* Starts a custom Gazebo world with a diff-drive vehicle, IMU, and RGB-D camera.
* Starts `ros_gz_bridge` with `enable_automated_bridge:=true`.
* Loads the `ros_pub_sub_node` test node after startup.

The launch file supports one parameter:

* `use_composition`: controls whether the test node is loaded as a composable node. The default value is `False`.

The bridge action passes automated bridge parameters directly to `ros_gz_bridge`:

```python
RosGzBridge(
    bridge_name='ros_gz_bridge',
    config_file=bridge_file,
    bridge_params=[
        {'expand_gz_topic_names': False},
        {'enable_automated_bridge': True},
    ],
)
```

##### 1.2.2 Test node

The [ros_pub_sub_node](https://github.com/azeey/gsoc2026_multirobot/blob/main/bridge/auto_bridge_test/src/RosPubSubNode.cpp) is parameter-driven. For each test interface, the package can independently configure whether the interface is enabled and which ROS topic or service name should be used.

The node can create:

* ROS subscribers for `GZ_TO_ROS` validation, such as `/clock`, `/imu`, `/rgbd_camera/camera_info`, `/rgbd_camera/depth_image`, `/rgbd_camera/image`, `/rgbd_camera/points`, `/model/vehicle/odometry`, and `/model/vehicle/tf`.
* ROS publishers for `ROS_TO_GZ` validation, such as `/model/vehicle/cmd_vel` and `/model/vehicle/enable`.
* ROS service clients for automated service bridge validation, such as `/world/test_world/control`, `/world/test_world/remove`, `/world/test_world/create`, and `/world/test_world/set_pose`.

#### 1.3 Test result

The behavior test confirms that automated bridge creation works for all expected interface categories in the test world:

* Gazebo-published topics are bridged to ROS when matching ROS subscribers exist.
* ROS-published command topics are bridged to Gazebo when matching Gazebo subscribers exist.
* Gazebo services are exposed to ROS when matching ROS service clients exist.
* The same launch flow works with both standalone and composable test node execution through `use_composition`.

### 2. Overhead test

#### 2.1 Overview

A [test package](https://github.com/azeey/gsoc2026_multirobot/tree/main/bridge/overhead——test) is provided to evaluate the runtime cost of automated bridge creation compared with the existing selective YAML bridge workflow. The test uses a controlled 12-robot Gazebo world with **124 Gazebo topics and 94 Gazebo services in total**.

Each robot has an IMU and RGB-D camera, subscribes to `/cmd_vel` and `/enable`, and publishes odometry, TF, IMU, camera info, RGB image, depth image, and point cloud topics.

#### 2.2 Content

##### 2.2.1 Launch files

The package provides two launch files for comparison:

* [overhead_test_original.launch.py](https://github.com/azeey/gsoc2026_multirobot/blob/main/bridge/overhead_test/launch/overhead_test_original.launch.py): starts the same multi-robot world and creates bridges from a selective YAML configuration file.
* [overhead_test_dynamic.launch.py](https://github.com/azeey/gsoc2026_multirobot/blob/main/bridge/overhead_test/launch/overhead_test_dynamic.launch.py): starts the same multi-robot world and enables automated bridge creation with `enable_automated_bridge:=true`.

##### 2.2.2 Measurement script

The package includes [test_overhead.sh](https://github.com/azeey/gsoc2026_multirobot/blob/main/bridge/overhead_test/test_overhead.sh) to measure bridge process overhead. The script:

* Finds running bridge processes such as `ros_gz_bridge`, `parameter_bridge`, or `bridge_node`.
* Samples CPU usage and RSS once per second.
* Reports average total CPU and average total RSS over the sampling duration.

#### Test result

Test environment:

* Ubuntu 24.04.4 LTS
* Linux 6.8.0-124-generic
* Intel Core Ultra 5 125H, 14 cores / 18 logical CPUs
* 30 GiB RAM

Results:

| Bridge mode | Average CPU | Average RSS |
| --- | ---: | ---: |
| Selective YAML bridge | 2.84% | 141.3 MB |
| Automated bridge | 5.67% | 131.2 MB |

The automated bridge uses more CPU because, compared with the selective YAML setup, it periodically queries Gazebo for available topics and services. However, the additional CPU usage remains within an acceptable range. Memory usage remains in the same range. Overall, the result shows that this approach is feasible for implementing automated bridge creation.

## Project Context

This design document is part of the GSoC 2026 project **Scalable Multi-Robot Integration and Automated Bridging for ROS 2 and Gazebo** under Gazebo.
