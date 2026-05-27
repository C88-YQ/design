# Namespace Support

## Background and Purpose

In multi-robot simulations, being able to reuse the same SDF / URDF robot description file to create multiple robot instances through mechanisms such as `<include>`, ROS spawn, or Gazebo service spawn would significantly simplify  simulation configuration. 

However, to avoid topic collisions between robot instances, users currently often need to manually modify the description file, for example by adding prefixes to topic names, sensor parameters, or plugin parameters. This introduces duplicated SDF / URDF content and increases maintenance cost. 

The goal of this design is to provide more consistent native namespace support in Gazebo, allowing different model instances to be assigned different namespaces without duplicating or heavily modifying the original description file, while keeping their communication interfaces isolated and enabling more scalable multi-robot simulation.

## SDFormat attribute

* **Related repo**: [sdformat](https://github.com/gazebosim/sdformat)

* **Related discussion**: [sdformat #1659 issue: Add namespace support to SDF elements for easier multi-robot simulation](https://github.com/gazebosim/sdformat/issues/1659)

* **Scope**: `world`,` model`, `sensor`, `plugin`,` particle_emitter`,` include` 

* **Approach**: 

  1. Add an **optional** `namespace` **attribute** to `world`, `model`, `sensor`, and `plugin`

     Since `namespace` is similar to `name` in that it describes metadata of the element itself, it may be reasonable to define it as an **attribute**. For example:

     ```xml
     <attribute name="namespace" type="string" default="" required="0">
       <description>
         Namespace used for communication interfaces related to this element.
       </description>
     </attribute>
     ```

     To avoid making this a breaking change, we could make namespace an **optional attribute** with an **empty default value**.

  2. Add an **optional** `namespace` **element** to `include`

     For `include`, the design may need to be different from `world`, `model`, `sensor`, and `plugin`. It may be better to follow the current `name` **override** style in `include`, and add `namespace` as an **element** inside `include`. For example:

     ```xml
     <include>
       <uri>model://my_robot</uri>
       <name>robot1</name>
       <namespace>robot1</namespace>
     </include>
     ```

     The `namespace` value in `include` could override the `namespace` of the top-level included `element`.

  3. Support a **placeholder** such as `__name__`

     In many cases, the desired `namespace` is the same as the `name`. For this common case, it may be useful to support a **placeholder** such as `__name__`, which means “use the final resolved name of this element as the namespace”. For example:

     ```xml
     <model name="robot" namespace="__name__">
       ...
     </model>
     ```

     For `ros_gz` users, they could specify only the `entity` name when spawning the robot, and the `namespace` could automatically follow that `name`.

     ```bash
     ros2 run gazebo_ros spawn_entity.py \
       -entity robot1 \
       -topic robot_description \
     ```

  4. Fallback option: use a Gazebo-specific extension

     Since changes to SDFormat need to consider a broader set of users and use cases, this design should not be based only on Gazebo-specific needs. So if adding a standard `namespace` attribute to SDFormat does not reach agreement, we could support a custom extension instead. For example:

     ```
     <model name="robot" gz:namespace="robot1">
     	...
     </model>
     ```

## Entity Level: Namespace component

* **Related repo**: [gz-sim](https://github.com/gazebosim/gz-sim)

* **Purpose**: Get the `namespace` attribute from SDF file and assign it to the component of the corresponding Entity.

* **Scope**: This is an Entity-level behavior, so it mainly applies to `world`, `model`, and `sensor`.

* **Approach**:

  1. Define a `namespace` component.

     Reference: [gz-sim/include/gz/sim/components/Name.hh](https://github.com/gazebosim/gz-sim/blob/11ab49c62b2791521f2000c1482c2c42d75788c1/include/gz/sim/components/Name.hh)

  2. Get the value of the `namespace` component when creating the Entity.

     Reference: [gz-sim/src/SdfEntityCreator.cc L263](https://github.com/gazebosim/gz-sim/blob/839e621c852df43b8092d103f304779b47b91b61/src/SdfEntityCreator.cc#L264)

## Plugin Level: Generate topic/service name with the namespace

* **Related repo**: [gz-sim](https://github.com/gazebosim/gz-sim) ; [gz-sensors](https://github.com/gazebosim/gz-sensors) 

* **Purpose** : Get the `namespace` attribute from plugins and use it to prefix topic names. To support this, plugins will need to be represented as **entities** so that namespace-related information can be stored and queried through **components**.

* **Approach**:

  1. Add entity for plugin in `gz-sim`.

  2. Get `namespace` during the plugin’s `Configure` phase. 

  3. Search the namespaces carried by each model layer by layer upward and combine them into the final full namespace. This change could be implemented as a common utility function in [gz-sim/src/Util.cc](https://github.com/gazebosim/gz-sim/blob/main/src/Util.cc). 

  4. Process the topic/service name by prefixing them with the namespace

     Some plugins already implement similar logic. This spreadsheet provides a summary of all files in `gz-sim` and `gz-sensors` that subscribe to or publish topics, or request or respond to services: [gz topic/service name (google sheet)](https://docs.google.com/spreadsheets/d/e/2PACX-1vRW_W8nroG_7Hcvlt8TP_o85oyIVJ_FXLulsFX6GcIHwTTIL7DPSbUV34uSMJxfD4j1Cxv78Ouv3BOe/pubhtml)

     1. `common`:

        - **Scope**: Users can configure these topic / service names through SDF parameters. The final topic / service name may either directly use the user-defined parameter, or append additional content to the user-defined parameter.

        - **Example**: [gz-sensors/src/DepthCameraSensor.cc](https://github.com/gazebosim/gz-sensors/blob/main/src/DepthCameraSensor.cc)

          | topic                | sdf param         | customized topic name | default topic name                                  |
          | -------------------- | ----------------- | --------------------- | --------------------------------------------------- |
          | /camera/depth        | topic             | /<topic>              | /camera/depth                                       |
          | /camera_info         | camera_info_topic | /<camera_info_topic>  | /<camera_depth_topic without last part>/camera_info |
          | /points              |                   |                       | /<camera_depth_topic>/points                        |
          | /set_rate            | topic             | /<topic>/set_rate     | /{sensor_name}/set_rate                             |
          | /performance_metrics |                   |                       | /<camera_depth_topic>/performance_metrics           |
          | /trigger             | trigger_topic     | /<trigger_topic>      | /<camera_depth_topic>/trigger                       |

        - **Approach**: If a namespace attribute is set at any level, the final full namespace will be prepended to both customized topic names and default topic names. If no namespace attribute is set, the topic names will remain unchanged.

     2. `fixed but safe`: 

        - **Scope**: These topic / service names are fixed or generated from fixed templates, and cannot be configured by users. They are usually used for top-level global control commands.

        - **Example**: [gz-sim/src/gui/plugins/spawn/Spawn.cc](https://github.com/gazebosim/gz-sim/blob/main/src/gui/plugins/spawn/Spawn.cc)

          | topic   | sdf param | customized topic name | default topic name         |
          | ------- | --------- | --------------------- | -------------------------- |
          | /create |           |                       | /world/{world_name}/create |

        - **Approach**: These names should remain fixed global names, so we do not need to prefix them with a namespace.

     3. `fixed`:

        - **Scope**: These topic / service names are fixed or generated from fixed templates, and cannot be configured by users. However, they are usually plugin-level commands, so name conflicts may occur when creating multiple robots from the same SDF / URDF file.

        - **Example**: [gz-sim/src/systems/diff_drive/DiffDrive.cc](https://github.com/gazebosim/gz-sim/blob/main/src/systems/diff_drive/DiffDrive.cc)

          | topic   | sdf param | customized topic name | default topic name         |
          | ------- | --------- | --------------------- | -------------------------- |
          | /enable |           |                       | /model/{model_name}/enable |

        - **Approach**: If a namespace attribute is set at any level, the final full namespace will be prepended to both customized topic names and default topic names. If no namespace attribute is set, the topic names will remain unchanged.

     4. `mixed`:

        * **Scope**: These topic / service names can be configured by users. However, additional fixed content, or content generated from a fixed template, is **prepended** to the user-defined parameter.

        * **Example1**:[gz-sim/src/systems/ackermann_steering/AckermannSteering.cc](https://github.com/gazebosim/gz-sim/blob/main/src/systems/ackermann_steering/AckermannSteering.cc)

          | topic    | sdf param | customized topic name           | default topic name          |
          | -------- | --------- | ------------------------------- | --------------------------- |
          | /cmd_vel | topic     | /<topic>                        | /model/{model_name}/cmd_vel |
          |          | sub_topic | /model/{model_name}/<sub_topic> | /model/{model_name}/cmd_vel |

        * **Example2**:[gz-sim/src/systems/buoyancy_engine/BuoyancyEngine.cc](https://github.com/gazebosim/gz-sim/blob/main/src/systems/buoyancy_engine/BuoyancyEngine.cc)

          | topic   | sdf param | customized topic name                             | default topic name              |
          | ------- | --------- | ------------------------------------------------- | ------------------------------- |
          | /cmd    | namespace | /model/<namespace>/buoyancy_engine/               | /buoyancy_engine/               |
          | /status | namespace | /model/<namespace>/buoyancy_engine/current_volume | /buoyancy_engine/current_volume |

        * **Approach**: If a namespace attribute is set at any level, the final full namespace will be prepended to both customized topic names and default topic names. If no namespace attribute is set, the topic names will remain unchanged.

     5. `namespace`: 

        - **Scope**: These topics already have a namespace-like concept, which overlaps with the namespace scheme proposed in this design.

        - **Example**:[src/systems/multicopter_control/MulticopterVelocityControl.cc](https://github.com/gazebosim/gz-sim/blob/main/src/systems/multicopter_control/MulticopterVelocityControl.cc)

          | topic    | sdf param                                            | customized topic name               | default topic name        |
          | -------- | ---------------------------------------------------- | ----------------------------------- | ------------------------- |
          | /cmd_vel | robotNamespace(required) & commandSubtopic(optional) | /<robotNamespace>/<commandSubtopic> | /<robotNamespace>/cmd_vel |
          | /enable  | robotNamespace(required) & enableSubtopic(optional)  | /<robotNamespace>/<enableSubtopic>  | /<robotNamespace>/enable  |

          **Note**: For this plugin, the namespace should first be resolved from the plugin namespace or the namespace of the model that directly contains the plugin. If either one is specified, it will be used as `robotNamespace`, regardless of whether the existing `robotNamespace` parameter is still present. If neither namespace is specified, the plugin will fall back to the existing `robotNamespace` parameter. If no namespace can be resolved, the plugin should report an error indicating that `robotNamespace` is required.

        - **Approach**: Deprecate the existing parameter and recommend the new namespace definition as the preferred approach. The existing logic will still be kept in the code to make backporting easier and preserve compatibility.

     6. `tf`:

        * **Scope**: `/tf` in `AckermannSteering`, `DiffDrive`, `MecanumDrive`, `TrackedVehicle.cc`, `OdometryPublisher`

        * **Approach**:
          1. Add a `gz:policies` option to make frame IDs hierarchical by taking model nesting into account.
          2. Treat TF topics the same as other topics. When a namespace is specified, it will be prepended to the TF topic name.
          3. With automatic topic bridging in `ros_gz`, all TF topics will be published to `/tf` on the ROS side by default. Advanced users can still override this behavior if they need separate TF topics in ROS.

     7. `actuators`: 

        * **Scope**:`AckermannSteering`, `JointController`, `JointPositionController`

        * **Related discussion**: [zulip topic: Question about the design and usage of /actuators](https://openrobotics.zulipchat.com/#narrow/channel/526040-Gazebo-General/topic/Question.20about.20the.20design.20and.20usage.20of.20.2Factuators/near/595615090)

        * **Example**:[gz-sim/src/systems/ackermann_steering/AckermannSteering.cc](https://github.com/gazebosim/gz-sim/blob/main/src/systems/ackermann_steering/AckermannSteering.cc)

          | topic      | sdf param | customized topic name           | default topic name |
          | ---------- | --------- | ------------------------------- | ------------------ |
          | /actuators | topic     | /<topic>                        | /actuators         |
          |            | sub_topic | /model/{model_name}/<sub_topic> | /actuators         |

        * **Approach**:

          - Controlling part or all of the actuators of a single robot: Keep the existing `topic` and `sub_topic` handling logic. If the user sets a namespace parameter, the complete namespace generated above will be added before the user-defined topic name.
          - Controlling all actuators across all robots: For users who intentionally want to use one shared global actuator topic, an extra parameter could be provided to keep `/actuators` global. Reusing the same SDF file with a shared global `/actuators` topic may still require extra templating or computation, such as `xacro:macro`, to assign the correct actuator index range for each robot.

## ROS Spawn: Add a new param

* **Approach**: Following the existing handling of the `name` parameter, add support for a new `namespace` parameter in [ros_gz/ros_gz_sim/src/spawn_entity.cpp](https://github.com/gazebosim/ros_gz/blob/ros2/ros_gz_sim/src/spawn_entity.cpp). Users can pass the `namespace` through a ROS command, and it will be injected into the `model` / `world` attribute.

## gz service Spawn: Add a new .msg

* **Related repo**: [ros_gz](https://github.com/gazebosim/ros_gz)

* **Approach**:

  1. Create a new `EntityFactoryWithNs.msg`

     Reference: [ros_gz/ros_gz_interfaces/msg/EntityFactory.msg](https://github.com/gazebosim/ros_gz/blob/ros2/ros_gz_interfaces/msg/EntityFactory.msg)

  2. Inject the `namespace` into the `model` / `world` attribute

     Reference:[ros_gz/ros_gz_sim/src/gz_simulation_interfaces/services/spawn_entity.cpp](https://github.com/gazebosim/ros_gz/blob/ros2/ros_gz_sim/src/gz_simulation_interfaces/services/spawn_entity.cpp)

## Project Context

This design document is part of the GSoC 2026 project **Scalable Multi-Robot Integration and Automated Bridging for ROS 2 and Gazebo** under Gazebo.