# Learning for Isaaclab
**目录 (Table of Contents)**

[TOC]


## 创建新项目
创建命令：`./isaaclab.sh --new`

创建后，导航到安装的项目并运行：`python -m pip install -e source/<given-project-name>`

	注：如果 Isaac Lab 没有安装在 conda 环境或（虚拟）Python 环境中，请使用 `FULL_PATH_TO_ISAACLAB/isaaclab.sh -p` （或在 Windows 上使用 `FULL_PATH_TO_ISAACLAB\isaaclab.bat -p` ）来替代 python 运行以下命令。


## 项目结构

## 配置项目
以`isaac_lab_tutorial`为例，项目结构可以分为：

类和配置:`source/isaac_lab_tutorial/isaac_lab_tutorial/tasks/direct/isaac_lab_tutorial`中查找`isaac_lab_tutorial_env_cfg.py `文件

环境：同目录的`isaac_lab_tutorial_env.py`文件

### 创建机器人
定义机器人：我们预谋添加一个名为`robots`的新`module`到我们的教程`extension`中，我们将在其中保留机器人的定义作为单独的 python 脚本。导航到`isaac_lab_tutorial/source/isaac_lab_tutorial/isaac_lab_tutorial`，创建一个名为`robots`的新文件夹。在此文件夹中创建两个文件: `__init__.py` 和 `jetbot.py` 。 `__init__.py` 文件将该目录标记为 python 模块，我们将能够以常规方式导入 `jetbot.py` 的内容。

```
# 1. 导入 Isaac Lab 的模拟器工具包
import isaaclab.sim as sim_utils

# 2. 导入「机器人配置」和「电机配置」工具
from isaaclab.assets import ArticulationCfg
from isaaclab.actuators import ImplicitActuatorCfg

# 3. 导入 Isaac Lab 自带的机器人模型路径
from isaaclab.utils.assets import ISAAC_NUCLEUS_DIR

# 4. 定义 Jetbot 机器人的配置（核心！）
JETBOT_CONFIG = ArticulationCfg(
    # 告诉模拟器：机器人的3D模型文件在哪里
    spawn=sim_utils.UsdFileCfg(usd_path=f"{ISAAC_NUCLEUS_DIR}/Robots/NVIDIA/Jetbot/jetbot.usd"),
    # 告诉模拟器：机器人的轮子用什么电机控制
    actuators={"wheel_acts": ImplicitActuatorCfg(joint_names_expr=[".*"], damping=None, stiffness=None)},
)
```
此外，可以修改换成其他机器人：
```
# 定义 Franka 机械臂配置（替换了 Jetbot）
FRANKA_CONFIG = ArticulationCfg(
    # 👉 只改这里：换成 Franka 的模型路径
    spawn=sim_utils.UsdFileCfg(usd_path=f"{ISAAC_NUCLEUS_DIR}/Robots/Franka/franka.usd"),
    # 电机配置（大部分机器人都能用这个通用配置）
    actuators={"arm_acts": ImplicitActuatorCfg(joint_names_expr=[".*"],  # 控制所有关节
            								   effort_limit=200.0,       # 电机最大力
            								   velocity_limit=10.0      # 电机最大速度)},
)

# 定义 Go2 四足机器人配置
GO2_CONFIG = ArticulationCfg(
    # 👉 只改模型路径
    spawn=sim_utils.UsdFileCfg(usd_path=f"{ISAAC_NUCLEUS_DIR}/Robots/Unitree/Go2/go2.usd"),
    actuators={"leg_acts": ImplicitActuatorCfg(joint_names_expr=[".*"],
            								   effort_limit=30.0,
            								   velocity_limit=5.0)},
)
```

### 环境配置
两个文件：
1. 环境配置文件`isaac_lab_tutorial_env_cfg.py`→ 告诉模拟器：用什么机器人、动作多大、观测多大、仿真速度

2. 环境逻辑文件`isaac_lab_tutorial_env.py`→ 告诉模拟器：机器人怎么动、奖励怎么算、什么时候重置

这是 Isaac Lab 强化学习的标准结构，所有机器人都这么配！

**环境配置文件**从`source/isaac_lab_tutorial/isaac_lab_tutorial/tasks/direct/isaac_lab_tutorial`中查找`isaac_lab_tutorial_env_cfg.py `文件

```
# 导入我们定义的 Jetbot 机器人配置
from isaac_lab_tutorial.robots.jetbot import JETBOT_CONFIG
# 可以换成 Franka 机械臂: from isaac_lab_tutorial.robots.franka import FRANKA_CONFIG

# 导入 Isaac Lab 必需的基础工具
from isaaclab.assets import ArticulationCfg
from isaaclab.envs import DirectRLEnvCfg
from isaaclab.scene import InteractiveSceneCfg
from isaaclab.sim import SimulationCfg
from isaaclab.utils import configclass

# 装饰器：标记这是一个配置类（固定写法）
@configclass
class IsaacLabTutorialEnvCfg(DirectRLEnvCfg):
    # 1. 强化学习基础参数
    decimation = 2                  # 动作重复次数（越高训练越快）
    episode_length_s = 5.0          # 每局游戏最大时长 5 秒
    action_space = 2                # 动作维度 = 2（左右轮速度）
    observation_space = 3           # 观测维度 = 3（机器人三维速度）
    state_space = 0                 # 不用额外状态，设为 0

    # 2. 仿真器配置
    sim: SimulationCfg = SimulationCfg(dt=1 / 120, render_interval=decimation)

    # 3. 机器人配置（关键！）
    robot_cfg: ArticulationCfg = JETBOT_CONFIG.replace(prim_path="/World/envs/env_.*/Robot")

    # 4. 场景配置（同时开 100 个机器人训练）
    scene: InteractiveSceneCfg = InteractiveSceneCfg(num_envs=100, env_spacing=4.0, replicate_physics=True)

    # 5. 机器人关节名称（Jetbot 只有两个驱动轮）
    dof_names = ["left_wheel_joint", "right_wheel_joint"]
	# 机械臂的话：action_space = 7， Franka 7个关节； dof_names = [".*"] 控制所有关节
```

**环境逻辑文件**从`source/isaac_lab_tutorial/isaac_lab_tutorial/tasks/direct/isaac_lab_tutorial`中查找`isaac_lab_tutorial_env.py `文件

```
# 必需导入
import torch
import numpy as np
from typing import Sequence

from isaaclab.envs import DirectRLEnv
from isaaclab.assets import Articulation
from isaaclab.sim import spawn_ground_plane, GroundPlaneCfg
from isaaclab.utils.math import *

# 导入上面的配置
from .isaac_lab_tutorial_env_cfg import IsaacLabTutorialEnvCfg

class IsaacLabTutorialEnv(DirectRLEnv):
    # 绑定配置
    cfg: IsaacLabTutorialEnvCfg

    def __init__(self, cfg: IsaacLabTutorialEnvCfg, render_mode: str | None = None, **kwargs):
        super().__init__(cfg, render_mode, **kwargs)
        # 获取关节索引（找到左右轮）
        self.dof_idx, _ = self.robot.find_joints(self.cfg.dof_names)

    # --------------------------
    # 场景初始化（建地面、灯光、机器人）
    # --------------------------
    def _setup_scene(self):
        self.robot = Articulation(self.cfg.robot_cfg)
        # 加地面
        spawn_ground_plane(prim_path="/World/ground", cfg=GroundPlaneCfg())
        # 克隆 100 个环境
        self.scene.clone_environments(copy_from_source=False)
        # 把机器人加入场景
        self.scene.articulations["robot"] = self.robot
        # 加灯光
        light_cfg = sim_utils.DomeLightCfg(intensity=2000.0, color=(0.75, 0.75, 0.75))
        light_cfg.func("/World/Light", light_cfg)

    # --------------------------
    # 动作处理（AI 输出 → 机器人执行）
    # --------------------------
    def _pre_physics_step(self, actions: torch.Tensor) -> None:
        self.actions = actions.clone()

    def _apply_action(self) -> None:
        # 给机器人左右轮设置速度
        self.robot.set_joint_velocity_target(self.actions, joint_ids=self.dof_idx)

    # --------------------------
    # 观测（AI 看到什么）
    # --------------------------
    def _get_observations(self) -> dict:
        # 机器人自身坐标系的线速度 (x,y,z)
        self.velocity = self.robot.data.root_com_lin_vel_b
        return {"policy": self.velocity}

    # --------------------------
    # 奖励（跑得越快分越高）
    # --------------------------
    def _get_rewards(self) -> torch.Tensor:
        # 奖励 = 机器人速度大小
        total_reward = torch.linalg.norm(self.velocity, dim=-1, keepdim=True)
        return total_reward

    # --------------------------
    # 结束条件（时间到就重置）
    # --------------------------
    def _get_dones(self) -> tuple[torch.Tensor, torch.Tensor]:
        time_out = self.episode_length_buf >= self.max_episode_length - 1
        return False, time_out

    # --------------------------
    # 机器人重置（回到出生点）
    # --------------------------
    def _reset_idx(self, env_ids: Sequence[int] | None):
        if env_ids is None:
            env_ids = self.robot._ALL_INDICES
        super()._reset_idx(env_ids)

        default_root_state = self.robot.data.default_root_state[env_ids]
        default_root_state[:, :3] += self.scene.env_origins[env_ids]

        self.robot.write_root_state_to_sim(default_root_state, env_ids)
```

**总结：**
`_cfg.py` = 设置机器人、动作、观测
`_env.py` = 写机器人怎么动、怎么奖励
动作 = 控制轮子 / 关节
奖励 = 跑得快就得分
