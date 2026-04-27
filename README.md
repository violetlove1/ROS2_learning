[![Readme Card](https://github-readme-stats.vercel.app/api/pin/?username=violetlove1&repo=github-readme-stats)](https://github.com/violetlove1/github-readme-stats)
## ROS2 Install
用ROS2 jazzy做一个mujoco franka机械臂的PD控制方法，如何开始下载ROS2并且完成这个项目呢


---

# 整体思路

这个项目涉及：

1. **ROS 2 Jazzy** — 通信框架
2. **MuJoCo** — 物理仿真引擎
3. **Franka 机械臂模型** — 需要 URDF/MJCF 模型
4. **PD 控制器** — 你自己写的控制逻辑
5. **ROS 2 节点** — 把控制器和 MuJoCo 连起来

所以流程是：

```
安装 ROS 2 Jazzy
    ↓
安装 MuJoCo
    ↓
准备 Franka 模型
    ↓
创建 ROS 2 工作空间
    ↓
写 MuJoCo 仿真节点
    ↓
写 PD 控制节点
    ↓
联调运行
```

---

# 阶段一：安装 ROS 2 Jazzy

 **Ubuntu 24.04**，正好是ROS2 Jazzy 的官方支持版本。

## 1. 设置 locale

```bash
sudo apt update && sudo apt install locales
sudo locale-gen en_US en_US.UTF-8
sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
export LANG=en_US.UTF-8
```

---

## 2. 添加 ROS 2 apt 源

```bash
sudo apt install software-properties-common
sudo add-apt-repository universe
```

```bash
sudo apt update && sudo apt install curl -y
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg
```

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null
```

---
这地方可能会报IP地址类型错误的信息
## 3. 安装 ROS 2 Jazzy Desktop

```bash
sudo apt update
sudo apt upgrade
sudo apt install ros-jazzy-desktop -y
```

这会装完整的桌面版，包括 RViz、rqt 等工具。

---

## 4. 安装开发工具及编译工具

```bash
sudo apt install ros-dev-tools -y

sudo apt update
sudo apt install python3-colcon-common-extensions
colcon --help
```

---

## 5. 配置环境

每次打开终端都要 source，所以建议加到 `.bashrc`：

```bash
echo "source /opt/ros/jazzy/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

---

## 6. 测试安装

```bash
ros2 --version
```

应该看到类似：

```bash
ros2 cli version: 0.x.x
```

再试一下：

```bash
ros2 run demo_nodes_cpp talker
```

另开一个终端：

```bash
ros2 run demo_nodes_cpp listener
```

如果能通信，说明 ROS 2 装好了。

---

# 阶段二：安装 MuJoCo

MuJoCo 现在是开源的，安装很简单。

## 1. 下载 MuJoCo

去官方 GitHub 下载最新版：

https://github.com/google-deepmind/mujoco/releases

或者直接用命令：

```bash
cd ~
wget https://github.com/google-deepmind/mujoco/releases/download/3.2.6/mujoco-3.2.6-linux-x86_64.tar.gz
```

（版本号可能更新，去 releases 页面看最新的）

---

## 2. 解压

```bash
mkdir -p ~/.mujoco
tar -xzf mujoco-3.2.6-linux-x86_64.tar.gz -C ~/.mujoco
```

---

## 3. 设置环境变量

```bash
echo 'export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:~/.mujoco/mujoco-3.2.6/lib' >> ~/.bashrc
source ~/.bashrc
```

---

## 4. 测试 MuJoCo

```bash
cd ~/.mujoco/mujoco-3.2.6/bin
./simulate
```

如果能打开 MuJoCo 的可视化窗口，说明装好了。

---

## 5. 安装 Python 绑定（mujoco-py 或 mujoco）

现在推荐用官方的 `mujoco` Python 包，不是老的 `mujoco-py`。

在你的 conda 环境里：

```bash
conda activate ros2_env_v1
pip install mujoco
```

测试：

```bash
python -c "import mujoco; print(mujoco.__version__)"
```

---

# 阶段三：准备 Franka 机械臂模型

MuJoCo 需要 MJCF 或 URDF 格式的模型。

## 方案 A：用现成的 Franka MJCF 模型

有些开源项目已经提供了 Franka 的 MuJoCo 模型，比如：

- **mujoco_menagerie**（Google DeepMind 官方模型库）

```bash
cd ~
git clone https://github.com/google-deepmind/mujoco_menagerie.git
```

里面有 Franka Emika Panda 的模型：

```bash
cd mujoco_menagerie/franka_emika_panda
```

你会看到 `.xml` 文件，这就是 MuJoCo 的模型文件。

---

## 方案 B：从 URDF 转换

如果你有 Franka 的 URDF（ROS 里常见），可以用工具转成 MJCF。

不过对新手来说，**直接用 mujoco_menagerie 里的模型更快**。

---

# 阶段四：创建 ROS 2 工作空间

## 1. 创建工作空间

```bash
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws/src
```

---

## 2. 创建你的功能包

```bash
ros2 pkg create --build-type ament_python franka_mujoco_control
```

这会创建一个 Python 包，结构类似：

```
franka_mujoco_control/
├── franka_mujoco_control/
│   └── __init__.py
├── package.xml
├── setup.py
└── ...
```

---

## 3. 进入包目录

```bash
cd franka_mujoco_control/franka_mujoco_control
```

---

# 阶段五：写 MuJoCo 仿真节点

你需要一个 ROS 2 节点来：

- 加载 MuJoCo 模型
- 运行仿真
- 发布机械臂状态（joint states）
- 接收控制指令（joint commands）

## 示例代码：`mujoco_sim_node.py`

在 `franka_mujoco_control/franka_mujoco_control/` 下创建：

```python
import rclpy
from rclpy.node import Node
from sensor_msgs.msg import JointState
from std_msgs.msg import Float64MultiArray
import mujoco
import mujoco.viewer
import numpy as np

class MuJoCoSimNode(Node):
    def __init__(self):
        super().__init__('mujoco_sim_node')
        
        # 加载模型
        model_path = '/home/你的用户名/mujoco_menagerie/franka_emika_panda/scene.xml'
        self.model = mujoco.MjModel.from_xml_path(model_path)
        self.data = mujoco.MjData(self.model)
        
        # 发布关节状态
        self.joint_state_pub = self.create_publisher(JointState, 'joint_states', 10)
        
        # 订阅控制指令
        self.cmd_sub = self.create_subscription(
            Float64MultiArray,
            'joint_commands',
            self.command_callback,
            10
        )
        
        # 定时器：仿真步进
        self.timer = self.create_timer(0.002, self.sim_step)  # 500Hz
        
        self.target_torques = np.zeros(7)  # Franka 有 7 个关节
        
        self.get_logger().info('MuJoCo simulation node started')
    
    def command_callback(self, msg):
        self.target_torques = np.array(msg.data)
    
    def sim_step(self):
        # 应用控制力矩
        self.data.ctrl[:7] = self.target_torques
        
        # 仿真步进
        mujoco.mj_step(self.model, self.data)
        
        # 发布关节状态
        joint_state = JointState()
        joint_state.header.stamp = self.get_clock().now().to_msg()
        joint_state.name = [f'joint{i+1}' for i in range(7)]
        joint_state.position = self.data.qpos[:7].tolist()
        joint_state.velocity = self.data.qvel[:7].tolist()
        
        self.joint_state_pub.publish(joint_state)

def main(args=None):
    rclpy.init(args=args)
    node = MuJoCoSimNode()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

---

# 阶段六：写 PD 控制节点

## 示例代码：`pd_controller_node.py`

```python
import rclpy
from rclpy.node import Node
from sensor_msgs.msg import JointState
from std_msgs.msg import Float64MultiArray
import numpy as np

class PDControllerNode(Node):
    def __init__(self):
        super().__init__('pd_controller_node')
        
        # PD 参数
        self.kp = np.array([100.0] * 7)  # 比例增益
        self.kd = np.array([10.0] * 7)   # 微分增益
        
        # 目标位置
        self.target_pos = np.array([0.0, -0.785, 0.0, -2.356, 0.0, 1.571, 0.785])
        
        # 当前状态
        self.current_pos = np.zeros(7)
        self.current_vel = np.zeros(7)
        
        # 订阅关节状态
        self.joint_state_sub = self.create_subscription(
            JointState,
            'joint_states',
            self.joint_state_callback,
            10
        )
        
        # 发布控制指令
        self.cmd_pub = self.create_publisher(Float64MultiArray, 'joint_commands', 10)
        
        # 定时器：控制循环
        self.timer = self.create_timer(0.01, self.control_loop)  # 100Hz
        
        self.get_logger().info('PD controller node started')
    
    def joint_state_callback(self, msg):
        self.current_pos = np.array(msg.position)
        self.current_vel = np.array(msg.velocity)
    
    def control_loop(self):
        # PD 控制
        error_pos = self.target_pos - self.current_pos
        error_vel = -self.current_vel
        
        torques = self.kp * error_pos + self.kd * error_vel
        
        # 发布
        cmd_msg = Float64MultiArray()
        cmd_msg.data = torques.tolist()
        self.cmd_pub.publish(cmd_msg)

def main(args=None):
    rclpy.init(args=args)
    node = PDControllerNode()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()
```

---

# 阶段七：配置 setup.py

编辑 `setup.py`，添加你的节点：

```python
entry_points={
    'console_scripts': [
        'mujoco_sim = franka_mujoco_control.mujoco_sim_node:main',
        'pd_controller = franka_mujoco_control.pd_controller_node:main',
    ],
},
```

---

# 阶段八：编译和运行

## 1. 回到工作空间根目录

```bash
cd ~/ros2_ws
```

---

## 2. 编译

```bash

colcon build --symlink-install
```

---

## 3. Source

```bash
source install/setup.bash
```

---

## 4. 运行仿真节点

```bash
ros2 run franka_mujoco_control mujoco_sim_node
```

---

## 5. 另开终端，运行控制节点

```bash
source ~/ros2_ws/install/setup.bash
ros2 run franka_mujoco_control pd_controller_node
```

---

# 阶段九：可视化（可选）

如果你想看 MuJoCo 的可视化窗口，可以在仿真节点里加：

```python
with mujoco.viewer.launch_passive(self.model, self.data) as viewer:
    while viewer.is_running():
        mujoco.mj_step(self.model, self.data)
        viewer.sync()
```

不过这会阻塞 ROS 2 的 spin，所以通常分开：

- 一个节点纯仿真 + ROS 通信
- 另开一个脚本做可视化

---

# 完整流程总结

```bash
# 1. 安装 ROS 2 Jazzy
sudo apt install ros-jazzy-desktop ros-dev-tools -y
echo "source /opt/ros/jazzy/setup.bash" >> ~/.bashrc

# 2. 安装 MuJoCo
cd ~
wget https://github.com/google-deepmind/mujoco/releases/download/3.2.6/mujoco-3.2.6-linux-x86_64.tar.gz
mkdir -p ~/.mujoco
tar -xzf mujoco-3.2.6-linux-x86_64.tar.gz -C ~/.mujoco
echo 'export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:~/.mujoco/mujoco-3.2.6/lib' >> ~/.bashrc

# 3. 安装 Python 绑定
conda activate ros2_env_v1
pip install mujoco

# 4. 下载 Franka 模型
cd ~
git clone https://github.com/google-deepmind/mujoco_menagerie.git

# 5. 创建 ROS 2 工作空间
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws/src
ros2 pkg create --build-type ament_python franka_mujoco_control

# 6. 写代码（上面的两个节点）

# 7. 编译
cd ~/ros2_ws
colcon build --symlink-install
source install/setup.bash

# 8. 运行
ros2 run franka_mujoco_control mujoco_sim
ros2 run franka_mujoco_control pd_controller
```

---

# 注意事项

## 1. 模型路径
代码里的：

```python
model_path = '/home/你的用户名/mujoco_menagerie/franka_emika_panda/scene.xml'
```

要改成你实际的路径。

---

## 2. 关节数量
Franka Panda 有 7 个关节，代码里写死了 7，如果模型不同要调整。

---

## 3. PD 参数调优
`kp` 和 `kd` 需要根据实际效果调整，初始值可能不稳定。

---

## 4. conda 环境和 ROS 2
如果你用 conda 环境，确保：

```bash
conda activate ros2_env_v1
source /opt/ros/jazzy/setup.bash
source ~/ros2_ws/install/setup.bash
```

---
```
┌──────────────┐                          ┌──────────────┐
│   Node A     │                          │   Node B     │
│  pd_control  │                          │  mujoco_sim  │
│              │──→ /joint_commands ────→│              │
│  publisher   │   Float64MultiArray      │  subscriber  │
│              │                          │   callback   │
│              │                          │              │
│  subscriber  │←── /joint_states ────────│  publisher   │
│   callback   │   sensor_msgs/JointState │              │
└──────────────┘                          └──────────────┘
```

# ROS2_learning
