[[#latest|跳转到latest记录状态]]
## 12.22
[https://hugging-face.cn/docs/lerobot/hilserl](https://hugging-face.cn/docs/lerobot/hilserl)

## 安装带 HIL-SERL 的 LeRobot
安装`pip install -e ".[hilserl]"` hilserl 的依赖

## 确定机器人工作空间边界
### 第一步使用 find_joint_limits.py
Realman_left_arm:通过拖动机械臂示教程序观测记录如下范围：

```python
"end_effector_bounds": {
    "max": [0.12, 0.46, 0.21],
    "min": [-0.27, 0.11, 0.05]
}

#单位为 m
```



如下的配置需要 robot 类，realman robot 类，teleoperator 类的配置文件才能跑通。

```python
class HILSerlRobotEnvConfig(EnvConfig):
    robot: RobotConfig | None = None    # Main robot agent (defined in `lerobot/robots`)
    teleop: TeleoperatorConfig | None = None    # Teleoperator agent, e.g., gamepad or leader arm, (defined in `lerobot/teleoperators`)
    wrapper: EnvTransformConfig | None = None    # Environment wrapper settings; check `lerobot/scripts/server/gym_manipulator.py`
    fps: int = 10    # Control frequency
```

### to do：
需要适配 lerobot 的 Realman robot 类；

需要适配的 teleoperator 类，如： VR，或者利用已有的 gamepad/ keyboard 类进行实现。



### 参考博客：
[https://zhuanlan.zhihu.com/p/1976053620911906986](https://zhuanlan.zhihu.com/p/1976053620911906986)

![](https://cdn.nlark.com/yuque/0/2025/jpeg/22559595/1766395787084-5c146064-e903-4cff-ba1d-044d1a49f41b.jpeg)

我觉得还是首先需要有 UDRF 文件的， lerobot上适配我们的Realman机械臂，然后将数采的流程使用lerobot框架下适配跑通，紧接着次可以进行后续的配置啦，官方文档也就只给个大概的流程和思想，配置文件，机器人类需要自己适配。



目前确定的流程如下：首先需要在lerobot上适配我们的Realman robot 类；然后进行 teleoperator 的类适配；验证是否通过：能够通过 lerobot 框架进行遥操作采集，获得 EE 末端位姿，并回放 record 采集数据后代表通过；接着才可以进行后续的配置啦。



关于 teleoperator 的控制方式如下：

## teleoperator
### gamepad：
gamepad 是如何遥控机械臂的、常用按键/轴映射、配置项以及如何把 gamepad 输出接入 IK/动作流水线。

要点总结

+ 控制输出：GamepadTeleop 把摇杆输入转换为 end‑effector 空间的增量动作（delta_x, delta_y, delta_z），如果启用夹爪还会输出 `gripper`（整数 0/1/2，对应 close/stay/open）。
+ 格式：action dict 示例 -> `{"delta_x": float, "delta_y": float, "delta_z": float, "gripper": 0|1|2}`；若在数据集里则对应 `action_features` shape 为 (4,)（含夹爪）或 (3,)（不含夹爪）。
+ 运动尺度：由底层 controller 的 step sizes 决定（GamepadController 构造的 x_step_size/y/z_step_size），gamepad 返回的值在 [-1,1]，再乘以 step_size 得到米级增量。
+ 集成流程：teleop.get_action() -> action_processor 的 AddTeleopActionAsComplimentaryDataStep 捕获 teleop 输入 -> MapTensorToDeltaActionDictStep / MapDeltaActionToRobotActionStep -> EEReferenceAndDelta -> EEBoundsAndSafety -> InverseKinematicsRLStep，把 EE 增量转换为 joint 目标并下发机器人。

具体按键/轴映射（两个模式：pygame 与 HID，都在代码里实现）

+ 摇杆：
    - 左摇杆：控制 X–Y 平面（前后/左右）
        * Pygame axes: 使用 axes 0/1（代码中 y_input = axis(0), x_input = axis(1)）
        * delta_x = -x_input * x_step_size（代码中是负号，看是否根据手柄反向）
        * delta_y = -y_input * y_step_size
    - 右摇杆（竖直轴）：控制 Z（上/下）
        * Pygame axis: axis(3) -> delta_z = -z_input * z_step_size
    - 有 deadzone（默认 0.1），避免漂移
+ 按键事件（pygame 模式）
    - B / Circle（按钮编号示例）：退出脚本
    - Y / Triangle（按钮 3）：标记成功（TeleopEvents.SUCCESS）
    - A / Cross（按钮 1）：标记失败（TeleopEvents.FAILURE）
    - X / Square（按钮 0）：重新录制（TeleopEvents.RERECORD_EPISODE）
    - RB（按钮 5，通过 `joystick.get_button(5)`）用于设置“介入（intervention）”标志 —— 当该位为真，系统认为人为接管
    - JOYBUTTONDOWN 中：button 6/7 被用作 gripper close/open（不同控制器/驱动会有偏差）
+ HID 模式：同样功能，但通过 hid API 读原始字节并按 bitmask 解析（Logitech/Xbox/PS 系列支持）。

如何在配置中启用 gamepad

+ 在环境 config（JSON / Python）中：
    - 设置 teleop:  
`"teleop": { "type": "gamepad", "id": "leader", "use_gripper": true }`
    - 设置 processor.control_mode 为 "gamepad"（或在记录/训练脚本里选择 gamepad 模式）  
示例片段：  


```json
{
  "env": {
    "processor": {
      "control_mode": "gamepad",
      "gripper": { "use_gripper": true }
    },
    "teleop": { "type": "gamepad", "id": "leader" }
  }
}
```

如何用 gamepad 找工作空间边界（示例命令）

+ 用 gamepad 做 teleop 时不需要 `--teleop.port`：

```bash
lerobot-find-joint-limits \
  --robot.type=realman_robot \
  --robot.left_ip=192.168.0.18 \
  --robot.right_ip=192.168.0.19 \
  --robot.port=8080 \
  --robot.arm=left_arm \
  --robot.urdf_path=/path/to/realman.urdf \
  --robot.target_frame_name=gripper \
  --teleop.type=gamepad \
  --teleop.id=leader \
  --teleop_time_s=45
```

注意把 IP/URDF/arm/port 替换为我们现实中的值。

安全与调试

+ 先在低速度/小 step_size 下测试（避免撞击），确认轴方向（正负）是否符合预期；若方向反了，修改 sign 或在上层对轴取反。
+ 如果摇杆有漂移，增大 deadzone（默认 0.1）。
+ 初次运行时把 `use_gripper=false` 或保持夹爪安全（避免误触抓取）。
+ 观察 `teleop.get_teleop_events()`：在录制时按钮触发可用来标记成功/失败/重录。



### Keyboard：
基于` src/lerobot/teleoperators/keyboard/teleop_keyboard.py`。

有两种键盘 teleop：

    - `KeyboardTeleop`：按键直接映射为关节/动作键集合（通用）。
    - `KeyboardEndEffectorTeleop`（name="keyboard_ee"）：用于末端执行器控制，输出 `delta_x, delta_y, delta_z`（可选 `gripper`）。
+ 需要库：`pynput`（在无 DISPLAY 的 Linux 环境会跳过本地监听）。

控制映射（`KeyboardEndEffectorTeleop`）

+ 运动（生成 EE 增量）：
    - Arrow Up    → delta_y = -1
    - Arrow Down  → delta_y = +1
    - Arrow Left  → delta_x = +1
    - Arrow Right → delta_x = -1
    - Shift (左)  → delta_z = -1
    - Shift_R     → delta_z = +1
    - 注：代码用 int(val)（按下即 1）乘以 step size（由后端 EE step sizes 决定），方向实现中含有符号，部署前建议在低速下确认方向。
+ 夹爪：
    - Ctrl_R / Ctrl_L 改变 `gripper` 值（代码以按键为依据做 ±1），最终 `gripper` 为 0/1/2 表示 close/stay/open。
+ 事件（按键触发）：
    - 's'：标记成功（TeleopEvents.SUCCESS）
    - 'r'：重录（TeleopEvents.RERECORD_EPISODE）
    - 'q'：退出/终止（失败）
    - ESC：断开监听并退出
+ 干预检测：
    - 任何运动键按下被视为人为“介入”（is_intervention=True），用于训练时判断人为接管。

输出格式

+ `get_action()` 返回示例（含 gripper）：  
{"delta_x": float, "delta_y": float, "delta_z": float, "gripper": int}
+ `action_features` 对应 shape (3,) 或 (4,)（见代码）。

与流水线的连接（高层）

+ teleop 输出由处理器捕获：`AddTeleopActionAsComplimentaryDataStep -> MapTensorToDeltaActionDictStep -> MapDeltaActionToRobotActionStep -> EEReferenceAndDelta -> EEBoundsAndSafety -> InverseKinematicsRLStep → 最终生成关节目标并下发机器人`（参见 gym_manipulator.py 中的处理器链）。

运行与调试建议

+ 确认 `pynput` 可用且已授权（无 DISPLAY 的 headless 环境会跳过监听）；若 headless，可改用远端客户端或手柄。
+ 初次测试用小的 `end_effector_step_sizes`（如 0.01–0.02 m），低速运行，确认方向与范围。
+ 若键位/方向不符合预期，可在上层对轴取反或修改键映射。
+ 想记录/标注示范时使用 `s/r/q` 进行成功/重录/失败标记。



### 是否需要 UDRF
需要 UDRF，HIL-SERL 论文中明确指出：需要使用 EE 空间进行学习，关节空间很多动作学习不到，但是转换到 EE 空间可以学习到。 如果想要在训练过程中以 EE 空间学习，lerobot 支持自动 IK 解算，这也是 HIL-SERL 流水线中不可或缺的部分，需要给出 UDRF 文件。



+ 不需要 URDF 的话，目前我实现了
    - `test_send_action.py`、调用 `RobotController.send_command(..., OSC_CARTESIAN_POSE, delta, ...)` 发送笛卡尔增量到控制器时，不需要 URDF。
    - 优点：实现简单、可立刻控制机器人。
    - 缺点：无法在 LeRobot 的处理流水线里使用基于 IK 的安全裁剪/EE‑bounds/EE‑to‑joint 转换；要靠控制器自身的安全策略或自己在上层做检测。
+ 需要 URDF 的
    - 就可以直接把 teleop（gamepad/keyboard）输出作为“末端增量”送入 LeRobot 的处理器链，并让 lerobot 系统用 RobotKinematics（RobotKinematics(urdf_path, target_frame_name)）做正/逆运动学、应用 `EEBoundsAndSafety`、再执行 `InverseKinematicsRLStep` 得到关节目标时，就必须有 URDF（且要正确的 `target_frame_name`，Realman 常用 "gripper"）。
    - 需要 URDF 的功能：自动裁剪 EE 工作区、把视觉坐标与 EE 变换对齐、**在训练流水线中以 EE 空间学习。**



有 URDF 会明显更方便，尤其是要把遥操作接入 LeRobot 的自动化/安全/训练流水线。要点：

+ 启用 IK 与自动转换：RobotKinematics 依赖 URDF，能做正/逆运动学，把 EE 增量自动转成关节目标，方便用 `EEReferenceAndDelta` / `InverseKinematicsRLStep`（见 `src/lerobot/model/kinematics.py`、gym_manipulator.py）。
+ 工作空间和安全：可用 `EEBoundsAndSafety` 自动裁剪 EE 目标并防止越界，减少人工出错风险。
+ 相机 ↔ EE 坐标变换：URDF 提供工具框架（`gripper`）与变换，便于把视觉检测结果映射到 EE，自动化奖励检测与任务目标定位。
+ 训练/数据处理自动化：有 URDF 能在采集/训练流程中统一 FK/IK、裁剪与观察处理，利于分布式 actor‑learner、reward classifier 等模块无缝工作。
+ 可视化与调试：URDF 让你使用可视化/仿真工具检查姿态和范围（更安全地验证配置）。





## 12.23
1. 新增 URDF 文件： rm_75_6f_description.urdf 并修正 mesh 路径；在 URDF 的末端添加了固定链 gripper（可用 gripper 作为 target frame），适配 lerobot 框架。
2. 用 RobotKinematics(..., "gripper") 成功加载并测试（有 self-collision 警告，但不影响 IK 加载）。
3. 新增 realman 示例环境配置: 新增示例文件 realman_env_example.json，包含 inverse_kinematics 和end_effector_bounds 安全边界参数（可用昨天测试的值替代，等后续适配遥操后，可以自动测试边界；）
4. 扩展 Realman 配置: 在 lerobot 框架下 robots 里新增 realman 支持，包括 realman_config.py，realman_robot.py；在realman_config.py补充/兼容了 tool_frame_id、urdf_path 等字段，使realman_env_example.json能直接被解析为 RealmanRobotConfig。
5. 实例化验证: 通过构造 RealmanRobotConfig 并调用 make_robot_from_config 成功实例化出 RealmanRobot（验证通过）。
6. 前向运动学验证: 成功计算了一组关节设置的初始姿态对应的末端位姿，和 realman 的自带软件进行了比较，在 **base 坐标系下 Arm tip**：x 相差 10mm，y 相差<1mm, z 相差 5mm（lerobot 解算——>Realman 科技示教器： [0.12368393 0.19474141 0.1841405 ]——>[0.1336,0.19393,0.18949]) 

```bash
(lerobot) root@deeptouch-ai:/workspace/lerobot# python - <<'PY'
from lerobot.model.kinematics import RobotKinematics
k = RobotKinematics("Realman_robot/rm_description/urdf/RM75_6F/rm_75_6f_description.urdf","gripper")
print(k.forward_kinematics([0,0,0,0,0,0,0])[:3,3])
PY
WARNING: Robot has the following self collisions in neutral position:
  -base_link_0 collides with Link1_0
  -Link1_0 collides with Link2_0
  -Link2_0 collides with Link3_0
  -Link4_0 collides with Link5_0
  -Link6_0 collides with Link7_0
[0.00000000e+00 7.30565917e-23 8.79000000e-01]
(lerobot) root@deeptouch-ai:/workspace/lerobot#
```

```bash
python - <<'PY'
from lerobot.model.kinematics import RobotKinematics
k = RobotKinematics("Realman_robot/rm_description/urdf/RM75_6F/rm_75_6f_description.urdf","gripper")
print(k.forward_kinematics([15.0, 50.0, 30.0, 90.0, 20.0, 90.0, 90.0])[:3,3])
PY

(lerobot) root@deeptouch-ai:/workspace/lerobot# python - <<'PY'
from lerobot.model.kinematics import RobotKinematics
k = RobotKinematics("Realman_robot/rm_description/urdf/RM75_6F/rm_75_6f_description.urdf","gripper")
print(k.forward_kinematics([15.0, 50.0, 30.0, 90.0, 20.0, 90.0, 90.0])[:3,3])
PY
WARNING: Robot has the following self collisions in neutral position:
  -base_link_0 collides with Link1_0
  -Link1_0 collides with Link2_0
  -Link2_0 collides with Link3_0
  -Link4_0 collides with Link5_0
  -Link6_0 collides with Link7_0
[0.12368393 0.19474141 0.1841405 ]
(lerobot) root@deeptouch-ai:/workspace/lerobot# 
```

8. 新增 realman_find_joint_limits_usehand.py 支持按键拖动自动检测范围。

```bash
python Realman/realman_find_joint_limits_usehand.py --arm left_arm --duration 40
```

9. 键盘遥操作：python Realman_robot/Realman/keyboard_teleop_realman_fallback.py --arm left_arm --left-ip 192.168.0.18 --port 8080（可以通过键盘控制机械臂）

```bash
# GUI（有图形/键盘事件监听，使用交互式 teleop）：
python Realman_robot/Realman/keyboard_teleop_realman.py --arm left_arm --left-ip 192.168.0.18 --port 8080

# 无 GUI（无头 / 通过 stdin 按键字符回退）：
python Realman_robot/Realman/keyboard_teleop_realman_fallback.py --arm left_arm --left-ip 192.168.0.18 --port 8080
```

### 仿真 hil 交互
```bash
# 在服务器上（已在 dev container）
bash /workspace/lerobot_rl/start_vnc.sh

# 在运行仿真前设置环境
export DISPLAY=:99
# 不强制 EGL：让 glfw 使用 X
unset MUJOCO_GL
```

可以在[http://localhost:6080/vnc.html](http://localhost:6080/vnc.html)查看

```bash
# 运行仿真
conda activate hil
cd /workspace/lerobot_rl
python -m lerobot.rl.gym_manipulator --config_path src/lerobot/envs/gym_hil_env.json
```

  pip install Robotic_Arm

  apt-get install v4l-utils

  pip install flask

## 12.24
## 12.25
#### 
RLPD update：[RLPD——利用离线数据实现高效的在线RL：不进行离线RL预训练，直接应用离策略方法SAC，在线学习时对称采样离线数据-CSDN博客](https://blog.csdn.net/v_JULY_v/article/details/151026739)

## 12.26
创建一个新的容器，包含 lerobot，rl，以及 vr 数采流程。

```bash
docker run -it \
    --name lerobot_rl_vr_container \
    --gpus all \
    --pids-limit=-1 \
    --shm-size=100g \
    --net=host \
    -v /home/deeptouch/workspace:/workspace \
    -v /data:/data \
    -v /ssd_cach:/ssd_cach \
    -v /dev/bus/usb:/dev/bus/usb \
    -e NVIDIA_VISIBLE_DEVICES=all \
    -e NVIDIA_DRIVER_CAPABILITIES=all \
    -e DISPLAY=$DISPLAY \
    -v /tmp/.X11-unix:/tmp/.X11-unix \
    --privileged \
    --restart unless-stopped \
    lerobot_rl_vr:v1 \
    /bin/bash
```



## 12.29
1. 使用 keboard teleoperator 操控机械臂，打通记录 Realman record 数据的流程。
2. 解决容器对新插入 usb 设备无法识别的问题。通过挂载全部/dev 可以解决该问题。 

```bash
#检测所有摄像头
lerobot-find-cameras opencv
lerobot-find-cameras realsense
```

```bash
PYTHONPATH=/workspace/lerobot/src python3 -u -m \
lerobot.scripts.lerobot_record \
--robot.type=realman_robot \
--robot.cameras='{"cam0": {"index_or_path": "/dev/video4", "width": 640, "height": 480, "fps": 30}, "cam1": {"index_or_path": "/dev/video10", "width": 640, "height": 480, "fps": 30}, "cam2": {"index_or_path": "/dev/video16", "width": 640, "height": 480, "fps": 30}}' \
--teleop.type=realman_keyboard \
--dataset.repo_id=local/realman_3cams_test_short \
--dataset.single_task="test1" \
--dataset.num_episodes=2 \
--dataset.episode_time_s=20 \
--dataset.fps=10 \
--dataset.root=outputs/recordings_cams_run_short_10fps \
--dataset.push_to_hub=False \
--teleop.gripper_toggle=True \
--play_sounds=False
```

存在的问题：

- [x] 键盘遥操作时，为什么显示无头服务器，即使我在 5090 的主机上直接运行。已解决。
- [x] 相机热机 2s 稳定，热机错误。

## 12.30
- [ ] robot 的各种类实现，是实现数据采集的关键，包括其中的数据 feature 的定义和实现。
- [x] 夹爪关闭按下后，一会儿就打开了，没法完全关闭。从瞬时模式替换成切换模式。
- [x] 相机 record 数据 15s，却仍然只录了 5s。主循环帧率和设置不一致，导致时间戳和真实不一样。datasets.fps=
- [ ] 相机录取视频白平衡不对。排除主循环帧率不匹配问题导致；这个可能跟视频编码方式有关系。
- [x] 键盘遥操作的时候，不输入指令时候，机械臂难以保持静止状态，机械臂会偏向某个方向进行小幅度的连续偏转。
- [x] 增添新功能当录制结束后，自动恢复初始状态。
- [ ] 遥操的时候机械臂很抖，改进遥操作时候机械臂的抖动。
- [x] 数据采集保存的数据特征如何定义？从哪里获得？如何自定义需要保存的数据？在 robot 类中进行定义，对抽象方法进行重新定义，支持自己写。
- [ ] 数据采集时候的图像尺寸如何修改？如何对数据的图像进行处理后保存，并在推理时候使得获取的图像和采集的数据时候的处理方式相同？是否通过一个相同的处理函数 统一接口，使得可以简单复用并保持一致。
- [ ] 

```bash
# 真机: 
export PYTHONPATH=/workspace/lerobot/src
unset LEROBOT_DRY_RUN
python3 -u -m lerobot.scripts.lerobot_record \
  --teleop.discover_packages_path=lerobot.teleoperators.VR \
  --robot.type=realman_robot \
  --robot.dry_run=False \
  --dataset.repo_id=local/realman_vr_tune_test   \
  --robot.cameras='{"cam0": {"type": "intelrealsense", "serial_number_or_name": "335522070753", "width": 640, "height": 480, "fps": 30}, "cam1": {"type": "intelrealsense", "serial_number_or_name": "243722073411", "width": 640, "height": 480, "fps": 30}, "cam2": {"type": "intelrealsense", "serial_number_or_name": "913522071103", "width": 1280, "height": 720, "fps": 30}}' \ 
  --dataset.single_task="VR tune test" \
  --robot.restore_movej_velocity=5.0 \
  --teleop.type=realman_ros \
  --teleop.teleop_trans_mult=100.0 \
  --teleop.teleop_rot_mult=100.0 \
  --dataset.num_episodes=1 \
  --dataset.episode_time_s=5 \
  --dataset.root=outputs/recordings_VR_test_10fps \
  --dataset.push_to_hub=False \
  --dataset.fps=10 \
  --play_sounds=False
```

## 12.31
完成在 lerobot 框架中的 vr 类，通过 websockt 得到 ROS 转发的消息，进行操控 realman 机器人。

- [ ] 后续的优化，目前 vr 遥操不是特别的顺手和跟随，夹爪的状态不稳定。 待重构 robot_controller，以及 gripper 的 controller。

```bash
LEROBOT_REALMAN=1 LEROBOT_DRY_RUN=0 python3 src/lerobot/teleoperators/vr/test_vr_teleop.py
```

配置 realman 机器人类 现在已经修改源代码文件啦。

**1.1-1.4 元旦假期**

## 1.5
  # FileNotFoundError: [Errno 2] No such file or directory: 'spd-say' 这个报错的话，加上--play_sounds=False

```bash
python src/lerobot/scripts/lerobot_record.py \
  --robot.type=realman_robot_v2 \
  --robot.left_ip="192.168.0.18" \
  --robot.right_ip="192.168.0.19" \
  --robot.use_ros_control=true \
  --robot.cameras='{
        "cam0": {"type": "intelrealsense", "serial_number_or_name": "335522070753", "width": 640, "height": 480, "fps": 30},
        "cam1": {"type": "intelrealsense", "serial_number_or_name": "427622274259", "width": 848, "height": 480, "fps": 30},
        "cam2": {"type": "intelrealsense", "serial_number_or_name": "913522071103", "width": 640, "height": 480, "fps": 30}
    }' \
  --teleop.type=ros_vr_teleop \
  --teleop.rosbridge_host="localhost" \
  --teleop.left_topic="/teleoperation/left_cmd" \
  --teleop.right_topic="/teleoperation/right_cmd" \
  --dataset.repo_id="deeptouch/realman_action_test" \
  --dataset.root="/workspace/outputs/recordings_action_test7" \
  --dataset.fps=30 \
  --dataset.single_task="Remote control dual arm task" \
  --dataset.video=true \
  --play_sounds=False
```



性能优化：通过 async_read 解决 RealSense 崩溃和掉帧问题。

数据完整性：修正 ROS Topic 名称，解决 Action 全零问题。

体验提升：优化 lerobot_record.py 日志逻辑，消除 Reset 阶段刷屏。



## 1.6 日
##相机FPS 设置得按照相机规格配置，也就是必须设置为30FPS。

```bash
python /workspace/lerobot/src/lerobot/scripts/lerobot_record_vr.py \
  --robot.type=realman_robot \
  --robot.left_ip="192.168.0.18" \
  --robot.right_ip="192.168.0.19" \
  --robot.use_ros_control=true \
  --robot.cameras='{
        "cam0": {"type": "intelrealsense", "serial_number_or_name": "335522070753", "width": 640, "height": 480, "fps": 30},
        "cam1": {"type": "intelrealsense", "serial_number_or_name": "427622274259", "width": 640, "height": 480, "fps": 30},
        "cam2": {"type": "intelrealsense", "serial_number_or_name": "913522071103", "width": 640, "height": 480, "fps": 30},
        "cam4": {"index_or_path": "/dev/video16", "width": 640, "height": 480, "fps": 30}
    }' \
  --teleop.type=ros_vr_teleop \
  --teleop.rosbridge_host="localhost" \
  --teleop.left_topic="/teleoperation/left_cmd" \
  --teleop.right_topic="/teleoperation/right_cmd" \
  --dataset.repo_id="deeptouch/realman_action_test" \
  --dataset.root="/workspace/outputs/recordings_action_test109" \
  --dataset.fps=10 \
  --dataset.single_task="Remote control dual arm task" \
  --dataset.video=true \
  --play_sounds=False



python /workspace/lerobot/src/lerobot/scripts/lerobot_record_vr.py \
  --robot.type=realman_robot \
  --robot.left_ip="192.168.0.18" \
  --robot.right_ip="192.168.0.19" \
  --robot.use_ros_control=true \
  --robot.cameras='{
        "cam0": {"type": "intelrealsense", "serial_number_or_name": "427622274259", "width": 640, "height": 480, "fps": 30},
        "cam1": {"type": "intelrealsense", "serial_number_or_name": "427622271648", "width": 640, "height": 480, "fps": 30},
        "cam3": {"type": "intelrealsense", "serial_number_or_name": "913522071103", "width": 640, "height": 480, "fps": 30}
    }' \
  --teleop.type=ros_vr_teleop \
  --teleop.rosbridge_host="localhost" \
  --teleop.left_topic="/teleoperation/left_cmd" \
  --teleop.right_topic="/teleoperation/right_cmd" \
  --dataset.repo_id="deeptouch/realman_action_test" \
  --dataset.root="/workspace/outputs/recordings_action_test109" \
  --dataset.fps=10 \
  --dataset.single_task="Remote control dual arm task" \
  --dataset.video=true \
  --play_sounds=False

  
```

按键逻辑没有问题，可以控制开启录制，结束录制，保存。



但目前还有两个小问题：

- [x] 刚开始录制时候前一段时间 夹爪操响应很缓慢，而且表现为：，在第一次按下后，后面无论怎么操作都没有反应会保持关闭的状态，然后松开过一段时间后响应，但是只会小幅度张开又闭合。（检查 录制的逻辑）
- [x] 在录制时候按 A 复位时，机械臂复位地很卡顿。在关闭录制后，则复位是正常顺滑的。（检查复位的逻辑）
- [x] 在保存一个 episode 后，终端一直显示相机的报错，因为还没有开启，这时候可以不输出这段 log，以免干扰录制者观察信息。（当相机的阻塞或者掉线都会引起该报错，解决 4.后观察能否复现该问题。）
- [x] 潜在的相机掉线问题，相机会时不时的掉线。（解决方案：通过硬件更新来解决：通过更换相机，和线材来保证不掉线。）
+ 让 Realman 机器人观测中包含末端完整位姿（pose_quaternion）。
+ 打通 `lerobot_record_vr.py` 录制流程，解决运行中的 KeyError 异常。
+ 梳理并文档化 Robot feature / action 的命名规范，方便后续自行扩展。



## 1.8
原来的 GymManipulator 与现在的Realman + ros_vr_teleop 这条路跑 HIL‑SERL，和Realman 的实现是不兼容的：具体原因和涉及位置：

+ `RobotEnv` 现在是为 SO100 这类关节级机器人写的，强依赖 Dynamixel 风格的总线：
    - 在 __init__ / `_setup_spaces` / `_get_observation` 里用到了  
self.robot.bus.motors、sync_read("Present_Position")、sync_write("Goal_Position") 等接口。
    - 你的 RealmanRobot（在 realman_robot.py）只暴露了 get_observation() / send_action()，并没有 bus / motors 这些属性。
    - 所以用现在的 `env_config` 直接跑 `gym_manipulator`，会在 RobotEnv.__init__ 就抛 AttributeError: 'RealmanRobot' object has no attribute 'bus'。
+ `make_processors` 里也是按「关节空间 + IK」的假设写的：
    - 它从 env.robot.bus.motors.keys() 推出 motor_names，再构建 RobotKinematics 和一串 EEReferenceAndDelta、EEBoundsAndSafety、`InverseKinematicsRLStep`。
    - 你的 Realman 控制链路目前是「VR 直接给末端姿态 → RealmanController 接受末端 pose」，本身就已经在笛卡尔空间，不一定需要这套 IK pipeline。



为 realman 重构 HIL-SERL 强化学习接口：

1. 新增 gym_manipulator_realman.py 为 Realman 的现在实现方式兼容的 gym_manipulator
2. 新增配置文件：在 src/lerobot/configs/realman_rl 目录下：
    1. env_config_realman_vr_hilserl.json
    2. train_config_hilserl_realman_vr.json
3. 

```bash
cd /workspace/lerobot
python -m lerobot.rl.gym_manipulator_realman \
  --config_path src/lerobot/configs/realman_rl/env_config_realman_vr_hilserl.json
```


新增 打分按键，需要重新 build 一下 Ros 的。

1. TeleOptCmd 新增独立 rating 字段

+ 修改了消息定义：TeleOptCmd.msg
    - 现在字段是：
        * string record_state
        * float32 scale（继续只做手部位姿缩放）
        * uint8 primary_thumb
        * uint8 rating // 新增，专门给 HIL‑SERL 用
    - 约定：`rating = 1` 表示成功，`rating = 255` 表示失败，`0` 表示未打分。

你后面要重新 colcon build 一下



## 1.9
1. 已经成功开发了 HIL-SERL 的示范数据采集通道，使用 realman+vr + hil-serl 遥控操作。

待改进：

1. get 到的 action 和vr 里的格式， 是否跟 osc 或者 controller 兼容。 
2. 本质上是希望根据当前图像 image 和 state 按照 language 指定的任务，进行下一帧的 action 输出：是否将 action 作为学习的一部分，实际上 action 的构造就决定了 action 是如何产生的。是按照增量笛卡尔空间的，还是按照相对于 base 坐标系的 绝对坐标的。
3. 使用前后帧的 observation state 数据，去进行差分，得到相对位置变化 即：action。 



我觉得 VLA 应该重点解决的是：1.根据当前视觉观察到的环境，结合 language 任务信息：产生可行的稀疏轨迹末端操作器的，（对于可移动的机器人来说，产生 base 坐标系的移动）：（相对于机器人底部控制器的高频控制来说是稀疏的）；本质上应当是学习末端位置轨迹以及末端操作器的姿态。然后按照轨迹进行相差来构造 action——>交给底层的 controller。

HIL-SERL 默认动作只有四维（x，y，z）+gripper；扩展成 8 维度（x,y,z,qx,qy,qz,qw)+gripper。

**后续开发新 policy 时，不修改现有源代码，只进行引用和继承类。**

#### 长期 to dolist：
- [x] 创建 VR teleoperator 类，并测试能否成功。
- [x] 主要目标：在lerobot 推理容器里，加入 VR 遥操作。
- [x] 使用 VR teleoperator 类 进行 record 数据录制。
- [ ] 强化学习，使用先验演示数据训练 critical 。
- [ ] 人工介入，进行人工纠正，录制数据进入 buffer，进行训练 learner。


## 1.12
两个 D405 的配置：

```bash
 "cam0": {"type": "intelrealsense", "serial_number_or_name": "427622274259", "width": 640, "height": 480, "fps": 30},
 "cam1": {"type": "intelrealsense", "serial_number_or_name": "427622271648", "width": 640, "height": 480, "fps": 30}
```

右手： 427622271648

左手： 427622274259

重新编写了robot_controller，支持realman机器人，笛卡尔空间绝对位置，四元数的形式，进行控制。适配未来的训练策略。

## 1.13:
解决了robot_record_vr流程中，ros控制节点和 robot_controller sdk 共同对机械臂进行控制的问题，sdk静止时会对误差噪声响应，左手手持续向某个位置安装10fps进行微小的漂移。
同时，当数采时ros 节点控制采集数据时，sdk同时10fps进行loop控制，会导致数据采集时候有时候会抖动。

在realman 机器人类中，添加了use_ros_controller 参数，指定当使用vr record时，跳过sdk的controller，controller只进行获取机械臂的states，控制是由Ros控制的。

当replay 回放数据和使用record policy 进行推理时候，依旧使用sdk controller进行控制。兼容框架的其他机器人和功能。

重新标定 左右手初始位置：

右手：[5.875,-72.779,93.664,84.180,11.549,94.480,47.923]

左手：[16.737,52.108,29.185,83.404,18.195,90.774,87.215]

```bash
#录制时候use_ros_controller设置为true，replay和推理时候都设置为false
python /workspace/lerobot/src/lerobot/scripts/lerobot_record_vr.py \
  --robot.type=realman_robot \
  --robot.left_ip="192.168.0.18" \
  --robot.right_ip="192.168.0.19" \
  --robot.use_ros_control=true \
  --robot.cameras='{
        "cam0": {"type": "intelrealsense", "serial_number_or_name": "427622274259", "width": 640, "height": 480, "fps": 30},
        "cam1": {"type": "intelrealsense", "serial_number_or_name": "427622271648", "width": 640, "height": 480, "fps": 30},
        "cam2": {"type": "intelrealsense", "serial_number_or_name": "913522071103", "width": 640, "height": 480, "fps": 30}
    }' \
  --teleop.type=ros_vr_teleop \
  --teleop.rosbridge_host="localhost" \
  --teleop.left_topic="/teleoperation/left_cmd" \
  --teleop.right_topic="/teleoperation/right_cmd" \
  --dataset.repo_id="deeptouch/realman_action_test" \
  --dataset.root="/workspace/outputs/recordings_action_test113" \
  --dataset.fps=30 \
  --dataset.single_task="vr test" \
  --dataset.video=true \
  --play_sounds=False
```

```bash
python /workspace/lerobot/src/lerobot/scripts/lerobot_record_vr.py \
  --robot.type=realman_robot \
  --robot.use_ros_controller=true\
  --robot.left_ip=192.168.0.18 \
  --robot.right_ip=192.168.0.19 \
  --robot.cameras='{
        "cam0": {"type": "intelrealsense", "serial_number_or_name": "427622274259", "width": 640, "height": 480, "fps": 30},
        "cam1": {"type": "intelrealsense", "serial_number_or_name": "427622271648", "width": 640, "height": 480, "fps": 30},
        "cam2": {"type": "intelrealsense", "serial_number_or_name": "913522071103", "width": 640, "height": 480, "fps": 30}
    }' \
  --teleop.type=ros_vr_teleop \
  --teleop.rosbridge_host=localhost \
  --dataset.repo_id="deeptouch/realman_action_test" \
  --dataset.root="/workspace/outputs/recordings_action_test113_2" \
  --dataset.fps=10 \
  --dataset.single_task="vr test" \
  --dataset.video=true \
  --play_sounds=false
```

```bash
lerobot-replay \
  --robot.type=realman_robot \
  --robot.left_ip="192.168.0.18" \
  --robot.right_ip="192.168.0.19" \
  --dataset.repo_id="deeptouch/realman_action_test" \
  --dataset.root="/workspace/outputs/merged_2026.1.16_3" \
  --dataset.episode=0 \
  --play_sounds=False
```

推理时候：
```bash
python /workspace/lerobot/src/lerobot/scripts/lerobot_record_vr.py \
  --robot.type=realman_robot \
  --robot.left_ip="192.168.0.18" \
  --robot.right_ip="192.168.0.19" \
  --robot.use_ros_control=true \
  --robot.cameras='{
        "cam0": {"type": "intelrealsense", "serial_number_or_name": "427622274259", "width": 640, "height": 480, "fps": 30},
        "cam1": {"type": "intelrealsense", "serial_number_or_name": "427622271648", "width": 640, "height": 480, "fps": 30},
        "cam3": {"type": "intelrealsense", "serial_number_or_name": "913522071103", "width": 640, "height": 480, "fps": 30}
    }' \
  --dataset.repo_id="deeptouch/realman_action_test_eval" \
  --dataset.root="/workspace/outputs/recordings_action_test1_eval" \
  --dataset.fps=10 \
  --dataset.single_task="vr policy eval" \
  --dataset.video=true \
  --policy.path=/workspace/lerobot/outputs/train/你的run名/checkpoints/last/pretrained_model \
  --play_sounds=False
```

```bash
#检测所有摄像头
lerobot-find-cameras opencv
lerobot-find-cameras realsense
```

```bash
#使用该脚本来初始化手臂，想初始化哪个就--arm后面接哪个。
python /workspace/lerobot/Realman_robot/Realman/test_send_action.py --arm left_arm --loop
python /workspace/lerobot/Realman_robot/Realman/test_send_action.py --arm right_arm --loop
```



## 1.16

```
rsync -avzP -e "ssh -p 2022" /workspace/cross_attention/
data_images.zip rmc@192.168.1.101:/home/rmc/workspace/datasets/


rsync -avz \
  --rsync-path="mkdir -p /home/deeptouch/workspace/models/diffusion_training_20260114_155220/checkpoints/006990 && rsync" \
  /workspace/lerobot/outputs/diffusion_training_20260114_155220/checkpoints/006990/ \
  deeptouch@192.168.1.159:/home/deeptouch/workspace/models/diffusion_training_20260114_155220/checkpoints/006990/

# 传输模型文件
# 使用简易的自动化传输脚本，只需要模型checkpoint路径，即可自动化传输到159推理机器上。
  ./scripts_rmc/transfer_checkpoint.sh /workspace/lerobot/outputs/IROS_dp_state7_action8_wot_400_30fps_training_20260118_192637/checkpoints/best
```

推理：
```
python /workspace/lerobot/src/lerobot/scripts/lerobot_record.py \
  --robot.type=realman_robot \
  --robot.use_ros_controller=false\
  --robot.left_ip=192.168.0.18 \
  --robot.right_ip=192.168.0.19 \
  --robot.cameras='{
        "cam0": {"type": "intelrealsense", "serial_number_or_name": "427622274259", "width": 640, "height": 480, "fps": 30}, 
        "cam1": {"type": "intelrealsense", "serial_number_or_name": "427622271648", "width": 640, "height": 480, "fps": 30}, 
        "cam3": {"type": "intelrealsense", "serial_number_or_name": "243722073411", "width": 640, "height": 480, "fps": 30}
        }' \
  --dataset.repo_id="deeptouch/eval_diffusion" \
  --dataset.root="/workspace/outputs/eval/diffusion/dp_state7_action8_wot_400_30fps_training_20260118_192637_test1" \
  --dataset.fps=30 \
  --dataset.episode_time_s=15 \
  --dataset.single_task="Place the cube into the square hole in the first row." \
  --dataset.video=true \
  --play_sounds=false \
  --policy.path="/workspace/models/IROS_dp_state7_action8_wot_400_30fps_training_20260118_192637/checkpoints/best" \
  --robot.controlled_arms=left_arm \
  --robot.obs_state=left_arm_joint
```
组会：
1.vr采集，和控制器写好了，采集数据的replay 真机已经是没问题了。就后面能够拟合 EE trajectory
2.数据采集采集了430条
3.ICRA 环境和任务1 已经训练了140epoch


训练日志📔：
1.16:
IROS datasets1:2026.01.16;
训练checpoint 目录：/workspace/lerobot/outputs/act_baseline400_training_20260116_172055
训练6个摄像头，含错误的触觉image。


1.17:
ACT  绝对位姿 + vision only （no state）：学到的trajectory ，测试不同，只有开始有轻微移动。
原因可能是因为当前的轨迹记录的不是下一步要去的轨迹而是就等于当前image的轨迹。
将action 记录改为 后一帧轨迹，作为学习的对象。

python /workspace/shift_action_to_next.py \
  --input_root /workspace/outputs/merged_2026.1.16_3 \
  --output_root /workspace/outputs/merged_2026.1.16_3_action_next \
  --no-link_videos
to do list:

未来改进计划：
diffusion：
1.消除 observation state。 50 个episode 进行实验 简单消融看看。
2.删除掉不动的右臂的action，不作为输出。（避免右臂数据 loss很低，容易学习，导致右臂的loss很低，干扰左臂的学习）

ACT 模型实验：
1.消除 robot：state状态。 
2.消除 

我的动作语义是 7维的，机器人base坐标系下的末端绝对位姿，x,y,z + 表示姿态的四元数。

触觉改进：
- A. 视觉相机： cam4，cam5 触觉相机。
- B. 触觉分支backbone：resnet18.
- C. 融合策略：方案 1（contact，最简单）；方案 2（门控/cross-attn，更复杂实现）
- D. 训练策略：触觉 backbone 是否先冻结若干步/若干 epoch？还是端到端一起训？

tar -C /home/rmc/workspace/datasets -xzf /home/rmc/workspace/datasets/merged_2026.1.16_3_action_next.tar.gz
使用仿真实验就可以做这个；


## 1.18

编写了后处理数据集代码，通过后处理能够自定义裁剪 action 维度，state 维度，保持 action 和 state 为左臂的笛卡尔末端位姿+gripper，以及 7 个关节空间。

todo：待编写对应推理的裁剪维度。

debug 了多卡训练时候的冲突问题。目前已修复：优化设备管理逻辑，确保在多GPU场景下正确处理模型设备分配。push 到 github 上啦。

batchsize 32 num_works 16
训练10w step：大概23.8h。1.19 19:30左右能够训完到10w step。


## 1.19
ssh 密钥 生成。
ssh-keygen -t ed25519 -C "2459944653@qq.com"
cat ~/.ssh/id_ed25519.pub


## 1.20
找出了算法训练不work的原因。
编写了多种转换数据的脚本，发现observation state的值就没有变化，数据采集过程中的observation state更新有问题。 但是action 的 eep是读取正确的，通过复制action的当前帧eep为observation，下一帧作为action 构建了能用的新数据集，进行了diffusion policy的训练。

后续待修改的计划：1. 检查observation state 为什么在采集过程中没有变化？ 而action 却能够正常记录eep。 同时修改数据采集流程中 使得能够及时更新记录当前的state 包括 eep，joint space的，因为baseline中有方案是jointspace。

## 实验方案：
### 数据采集方案1:
数据集采样率：10Hz
**input：**
state：joint space & 笛卡尔末端位姿
image：3个visual相机：两个手部相机，一个正面俯视相机；两个视触觉相机：tactile image；
**output：**
action：当前：frame 的笛卡尔空间 末端位姿 （模型根据图片规划出末端的：trajectory）

action的采集后处理方式：
1.不处理，就用EE 的笛卡尔位姿(机器人base frame下的：绝对位姿， 会在进入网络前经过MEAN_STD 归一化处理)
2.增量笛卡尔空间 四元数表示位姿势（用前后两帧的进行计算，同样的会经过MEAN_STD 归一化处理）
3.增量欧拉角表示位姿

实验方案：都带state，state就只记录关节空间。action就是末端的笛卡尔空间。相当于这两个是求FK和IK的关系。（因为现有的文献都是带state 的，比较时候也必须要比较带state的，至少baseline是需要带上state保持与这些方法的原文是保持一致的）

算法方案：如下：
任务方案：如下：


现有的问题：训练dp时候，共享内存爆照。由于双臂的state，action shape太大导致最后全连接太多，训练速度慢。batchsize 设置为一样的情况下，256G shm直接爆啦。 
改进方法：将action和state都只保留运动的左臂，state还是保留为关节空间state shape为7。
已完成处理数据的脚本：action 和state 维度裁剪，同时更新meta info json里的统计量。

有待给lerobot提的issue：Q：为什么lerobot里训练各种VLA算法，即使是act，dp，在batchsize设置为64，num_works设置为32时候都要炸共享内存。

| 实验方案             | 插入圆柱体第一个孔 | 插入圆柱体第三个孔 | 泛化其他形状物体 |     |
| ---------------- | --------- | --------- | -------- | --- |
| ACT              |           |           |          |     |
| Diffusion policy |           |           |          |     |
| smolvla          |           |           |          |     |
| pi05             |           |           |          |     |
| groot            |           |           |          |     |
| ours             |           |           |          |     |
|                  |           |           |          |     |
|                  |           |           |          |     |
|                  |           |           |          |     |

# latest
