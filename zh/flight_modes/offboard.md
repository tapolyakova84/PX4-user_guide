# Offboard 模式

[<img src="../../assets/site/position_fixed.svg" title="需要定位修复（例如GPS）" width="30px" />](../getting_started/flight_modes.md#key_position_fixed)

飞机按照远端控制器通过MAVLink给出的位置，速度或姿态设定值来运行。 The setpoint may be provided by a MAVLink API (e.g. [MAVSDK](https://mavsdk.mavlink.io/) or [MAVROS](https://github.com/mavlink/mavros)) running on a companion computer (and usually connected via serial cable or wifi).

> **Tip** Not all co-ordinate frames and field values allowed by MAVLink are supported for all setpoint messages and vehicles. Read the sections below *carefully* to ensure only supported values are used. Note also that setpoints must be streamed at > 2Hz before entering the mode and while the mode is operational.
> 
> **Note** * This mode requires position or pose/attitude information - e.g. GPS, optical flow, visual-inertial odometry, mocap, etc. * RC control is disabled except to change modes. * 使用此模式前飞机必须先被激活。 * The vehicle must be already be receiving a **stream of target setpoints (>2Hz)** before this mode can be engaged. * 如果没能以 > 2Hz 的速率接收目标设定值飞机将退出该模式。 * Not all co-ordinate frames and field values allowed by MAVLink are supported.

## 描述

Offboard mode is primarily used for controlling vehicle movement and attitude, and supports only a very limited set of MAVLink messages (more may be supported in future).

其他操作, 如起飞、降落、返回起飞点，最好使用其它适当的模式来处理。 上传、下载任务等操作可以在任何模式下执行。

要启用或保持该模式, 飞机必须先接收到一个提供设定值的消息流 (如果消息速率低于 2Hz 飞机将退出该模式)。 如果要在此模式下做位置保持，必须向飞机提供一个包含当前位置设定值的消息流。

Offboard模式需要主动连接到远程 MAVLink 系统 (例如配套计算机或GCS)。 如果连接丢失, 在超时 ([COM_OF_LOSS_T](#COM_OF_LOSS_T)) 后, 飞机将尝试降落或执行其他故障保护操作。 故障保护操作在参数 [COM_OBL_ACT](#COM_OBL_ACT) 和 [COM_OBL_RC_ACT](#COM_OBL_RC_ACT) 中定义。

## Supported Messages

### Copter/VTOL

* [SET_POSITION_TARGET_LOCAL_NED](https://mavlink.io/en/messages/common.html#SET_POSITION_TARGET_LOCAL_NED)
  
  * The following input combinations are supported: <!-- https://github.com/PX4/PX4-Autopilot/blob/master/src/lib/FlightTasks/tasks/Offboard/FlightTaskOffboard.cpp#L166-L170 -->
    
    * Position setpoint (only `x`, `y`, `z`)
    * Velocity setpoint (only `vx`, `yy`, `vz`)
    * *Thrust* setpoint (only `afx`, `afy`, `afz`) > **Note** Acceleration setpoint values are mapped to create a normalized thrust setpoint (i.e. acceleration setpoints are not "properly" supported).
    * Position setpoint **and** velocity setpoint (the velocity setpoint is used as feedforward; it is added to the output of the position controller and the result is used as the input to the velocity controller).
  * * PX4 supports the following `coordinate_frame` values (only): [MAV_FRAME_LOCAL_NED](https://mavlink.io/en/messages/common.html#MAV_FRAME_LOCAL_NED) and [MAV_FRAME_BODY_NED](https://mavlink.io/en/messages/common.html#MAV_FRAME_BODY_NED).

* [SET_POSITION_TARGET_GLOBAL_INT](https://mavlink.io/en/messages/common.html#SET_POSITION_TARGET_GLOBAL_INT)
  
  * The following input combinations are supported: <!-- https://github.com/PX4/PX4-Autopilot/blob/master/src/lib/FlightTasks/tasks/Offboard/FlightTaskOffboard.cpp#L166-L170 -->
    
    * Position setpoint (only `lat_int`, `lon_int`, `alt`)
    * Velocity setpoint (only `vx`, `yy`, `vz`)
    * *Thrust* setpoint (only `afx`, `afy`, `afz`) > **Note** Acceleration setpoint values are mapped to create a normalized thrust setpoint (i.e. acceleration setpoints are not "properly" supported).
    * Position setpoint **and** velocity setpoint (the velocity setpoint is used as feedforward; it is added to the output of the position controller and the result is used as the input to the velocity controller).
  * PX4 supports the following `coordinate_frame` values (only): [MAV_FRAME_GLOBAL](https://mavlink.io/en/messages/common.html#MAV_FRAME_GLOBAL).

* [SET_ATTITUDE_TARGET](https://mavlink.io/en/messages/common.html#SET_ATTITUDE_TARGET)
  
  * The following input combinations are supported: 
    * Attitude/orientation (`SET_ATTITUDE_TARGET.q`) with thrust setpoint (`SET_ATTITUDE_TARGET.thrust`).
    * Body rate (`SET_ATTITUDE_TARGET` `.body_roll_rate` ,`.body_pitch_rate`, `.body_yaw_rate`) with thrust setpoint (`SET_ATTITUDE_TARGET.thrust`).

### 固定翼

* [SET_POSITION_TARGET_LOCAL_NED](https://mavlink.io/en/messages/common.html#SET_POSITION_TARGET_LOCAL_NED)
  
  * The following input combinations are supported (via `type_mask`): <!-- https://github.com/PX4/PX4-Autopilot/blob/master/src/lib/FlightTasks/tasks/Offboard/FlightTaskOffboard.cpp#L166-L170 -->
    
    * Position setpoint (`x`, `y`, `z` only; velocity and acceleration setpoints are ignored).
      
      * Specify the *type* of the setpoint in `type_mask` (if these bits are not set the vehicle will fly in a flower-like pattern):
        
        > **Note** Some of the *setpoint type* values below are not part of the MAVLink standard for the `type_mask` field.
        
        The values are:
        
        * 292: Gliding setpoint. This configures TECS to prioritize airspeed over altitude in order to make the vehicle glide when there is no thrust (i.e. pitch is controlled to regulate airspeed). It is equivalent to setting `type_mask` as `POSITION_TARGET_TYPEMASK_Z_IGNORE`, `POSITION_TARGET_TYPEMASK_VZ_IGNORE`, `POSITION_TARGET_TYPEMASK_AZ_IGNORE`. 
        * 4096: Takeoff setpoint.
        * 8192: Land setpoint.
        * 12288: Loiter setpoint (fly a circle centred on setpoint).
        * 16384: Idle setpoint (zero throttle, zero roll / pitch).
  * PX4 支持坐标系指定 (`coordinate_frame` 字段): [MAV_FRAME_LOCAL_NED](https://mavlink.io/en/messages/common.html#MAV_FRAME_LOCAL_NED) 和 [MAV_FRAME_BODY_NED](https://mavlink.io/en/messages/common.html#MAV_FRAME_BODY_NED)。

* [SET_POSITION_TARGET_GLOBAL_INT](https://mavlink.io/en/messages/common.html#SET_POSITION_TARGET_GLOBAL_INT)
  
  * The following input combinations are supported (via `type_mask`): <!-- https://github.com/PX4/PX4-Autopilot/blob/master/src/lib/FlightTasks/tasks/Offboard/FlightTaskOffboard.cpp#L166-L170 -->
    
    * Position setpoint (only `lat_int`, `lon_int`, `alt`)
      
      * Specify the *type* of the setpoint in `type_mask` (if these bits are not set the vehicle will fly in a flower-like pattern):
        
        > **Note** The *setpoint type* values below are not part of the MAVLink standard for the `type_mask` field.
        
        The values are:
        
        * 4096: Takeoff setpoint.
        * 8192: Land setpoint.
        * 12288: Loiter setpoint (fly a circle centred on setpoint).
        * 16384: Idle setpoint (zero throttle, zero roll / pitch).
  * PX4 supports the following `coordinate_frame` values (only): [MAV_FRAME_GLOBAL](https://mavlink.io/en/messages/common.html#MAV_FRAME_GLOBAL).

* [SET_ATTITUDE_TARGET](https://mavlink.io/en/messages/common.html#SET_ATTITUDE_TARGET)
  
  * The following input combinations are supported: 
    * Attitude/orientation (`SET_ATTITUDE_TARGET.q`) with thrust setpoint (`SET_ATTITUDE_TARGET.thrust`).
    * Body rate (`SET_ATTITUDE_TARGET` `.body_roll_rate` ,`.body_pitch_rate`, `.body_yaw_rate`) with thrust setpoint (`SET_ATTITUDE_TARGET.thrust`).

<!-- Limited for offboard mode in Fixed Wing was added to master after PX4 v1.9.0.
See https://github.com/PX4/PX4-Autopilot/pull/12149 and https://github.com/PX4/PX4-Autopilot/pull/12311 -->

### Rover

* [SET_POSITION_TARGET_LOCAL_NED](https://mavlink.io/en/messages/common.html#SET_POSITION_TARGET_LOCAL_NED)
  
  * The following input combinations are supported (in `type_mask`): <!-- https://github.com/PX4/PX4-Autopilot/blob/master/src/lib/FlightTasks/tasks/Offboard/FlightTaskOffboard.cpp#L166-L170 -->
    
    * Position setpoint (only `x`, `y`, `z`)
      
      * Specify the *type* of the setpoint in `type_mask`:
        
        > **Note** The *setpoint type* values below are not part of the MAVLink standard for the `type_mask` field.
        
        The values are:
        
        * 12288: Loiter setpoint (vehicle stops when close enough to setpoint).
    * Velocity setpoint (only `vx`, `yy`, `vz`)
  * PX4 支持坐标系指定 (`coordinate_frame` 字段): [MAV_FRAME_LOCAL_NED](https://mavlink.io/en/messages/common.html#MAV_FRAME_LOCAL_NED) 和 [MAV_FRAME_BODY_NED](https://mavlink.io/en/messages/common.html#MAV_FRAME_BODY_NED)。

* [SET_POSITION_TARGET_GLOBAL_INT](https://mavlink.io/en/messages/common.html#SET_POSITION_TARGET_GLOBAL_INT)
  
  * The following input combinations are supported (in `type_mask`): <!-- https://github.com/PX4/PX4-Autopilot/blob/master/src/lib/FlightTasks/tasks/Offboard/FlightTaskOffboard.cpp#L166-L170 -->
    
    * Position setpoint (only `lat_int`, `lon_int`, `alt`)
  * Specify the *type* of the setpoint in `type_mask` (not part of the MAVLink standard). The values are: 
    * Following bits not set then normal behaviour.
    * 12288: Loiter setpoint (vehicle stops when close enough to setpoint).
  * PX4 supports the coordinate frames (`coordinate_frame` field): [MAV_FRAME_GLOBAL](https://mavlink.io/en/messages/common.html#MAV_FRAME_GLOBAL).

* [SET_ATTITUDE_TARGET](https://mavlink.io/en/messages/common.html#SET_ATTITUDE_TARGET)
  
  * The following input combinations are supported: 
    * Attitude/orientation (`SET_ATTITUDE_TARGET.q`) with thrust setpoint (`SET_ATTITUDE_TARGET.thrust`). > **Note** Only the yaw setting is actually used/extracted.

## Offboard参数

*Offboard模式*受以下参数影响：

| 参数                                                                                                      | 描述                                                                                                                                                                                                                                                                                                                                        |
| ------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| <span id="COM_OF_LOSS_T"></span>[COM_OF_LOSS_T](../advanced_config/parameter_reference.md#COM_OF_LOSS_T)     | 在丢失Offboard连接时的等待超时 (以秒为单位), 然后将触发offboard丢失的故障保护措施 (`COM_OBL_ACT` 和 `COM_OBL_RC_ACT`)                                                                                                                                                                                                                                                    |
| <span id="COM_OBL_ACT"></span>[COM_OBL_ACT](../advanced_config/parameter_reference.md#COM_OBL_ACT)         | *没有* 连接到遥控器的情况下, 丢失offboard连接后切换到的模式 (取值为- 0: [Land](../flight_modes/land.md), 1: [Hold](../flight_modes/hold.md), 2: [Return ](../flight_modes/return.md))。                                                                                                                                                                              |
| <span id="COM_OBL_RC_ACT"></span>[COM_OBL_RC_ACT](../advanced_config/parameter_reference.md#COM_OBL_RC_ACT)   | 连接到遥控器的情况下，丢失offboard连接后切换到的模式 (取值为 - 0: *Position*, 1: [Altitude](../flight_modes/altitude_mc.md), 2: *Manual*, 3: [Return ](../flight_modes/return.md), 4: [Land](../flight_modes/land.md))。                                                                                                                                            |
| <span id="COM_RC_OVERRIDE"></span>[COM_RC_OVERRIDE](../advanced_config/parameter_reference.md#COM_RC_OVERRIDE) | If enabled, stick movement on a multicopter (or VTOL in multicopter mode) gives control back to the pilot in [Position mode](../flight_modes/position_mc.md) (except when vehicle is handling a critical battery failsafe). This can be separately enabled for auto modes and for offboard mode, and is enabled in auto modes by default. |

## 开发者资源

Typically developers do not directly work at the MAVLink layer, but instead use a robotics API like [MAVSDK](https://mavsdk.mavlink.io/) or [ROS](http://www.ros.org/) (these provide a developer friendly API, and take care of managing and maintaining connections, sending messages and monitoring responses - the minutiae of working with *Offboard mode* and MAVLink).

以下资源可能对开发者有用:

* [基于Linux的Offboard控制](../ros/offboard_control.md) (PX4 Devguide)
* [MAVROS Offboard控制示例](../ros/mavros_offboard.md) (PX4 Devguide)