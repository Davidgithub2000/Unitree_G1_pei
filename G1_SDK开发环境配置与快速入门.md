# Unitree G1 SDK 开发环境配置与快速入门

> 文档定位：面向第一次使用宇树 G1-EDU 和 `unitree_sdk2` 的开发者，完成“环境准备 → 网络连通 → SDK 编译 → 例程验证 → 自建工程”的最小闭环。  
> 主线范围：官方 C++ SDK。Python 与 ROS 2 仅说明入口，不在本文展开。  
> 核对日期：2026-09-11。官方文档和仓库会持续更新，实机操作前应再次核对机器人型号、固件版本和官方页面。

## 1. 简介

G1 主要使用 DDS 作为机器人内部及开发接口的消息中间件。开发者可通过高层服务接口调用站立、行走、手臂等能力，也可通过底层 DDS 话题读取状态并发送关节指令。

建议把首次开发过程记成一条主线：

`Ubuntu 20.04 → 有线接入 192.168.123.X 网段 → 编译 unitree_sdk2 → 先验证状态与高层接口 → 最后在保护条件下测试底层控制`

本文的目标不是让机器人立即运动，而是先确认以下四件事：

1. 开发机和 SDK 架构匹配；
2. 开发机能够稳定访问机器人网络；
3. 官方示例能够正确编译；
4. 机器人模式、安全保护和控制权满足例程运行条件。

## 2. G1 开发架构

### 2.1 通信方式

| 通信方式 | 适合的数据或操作 | 特点 |
| --- | --- | --- |
| 发布/订阅 | 关节状态、IMU、雷达、持续控制指令 | 适合中高频或连续数据；发送端和接收端通过 DDS 话题解耦 |
| 请求/响应 | 查询状态、功能切换、一次性操作 | 适合低频控制；官方接口既可使用底层 API，也可使用封装后的函数式调用 |

当前 C++ SDK 中，`ChannelFactory::Instance()->Init(domain_id, network_interface)` 用于初始化 DDS 通信。实机示例通常使用 DDS Domain ID `0`，并显式传入连接机器人的网卡名称。若使用带 `enableSharedMemory` 的三参数重载，外部开发机连接 G1 时应将共享内存通信关闭，即传入 `false`。

### 2.2 开发路径选择

| 路径 | 主要工具 | 适用场景 | 风险等级 |
| --- | --- | --- | --- |
| C++ 高层接口 | `unitree_sdk2` 的 Client/RPC 封装 | 状态查询、行走、站立、手臂动作等 | 中 |
| C++ 底层接口 | DDS 发布/订阅、`LowCmd_`、`LowState_` | 自定义关节控制、控制算法研究 | 高 |
| Python | 独立的 `unitree_sdk2_python` 仓库 | 快速原型、算法验证 | 取决于调用层级 |
| ROS 2 | G1 DDS IDL 与适配的 RMW | 接入 ROS 2 节点、感知和上层规划 | 取决于调用层级 |

本文采用官方 C++ 仓库 `unitree_sdk2`。不要把 Python 或 ROS 2 的安装命令混入该 C++ 构建流程。

### 2.3 机器人计算单元边界

G1-EDU 内置两块计算单元：

| 计算单元 | 用途 | 是否用于二次开发 |
| --- | --- | --- |
| PC1 | 运行宇树官方运动控制程序 | 否，官方文档说明不对开发者开放 |
| PC2 | 用户二次开发 | 是；官方架构页给出的地址为 `192.168.123.164` |

PC2 的初始用户名和密码需要向宇树技术支持获取。不要修改 PC1 上的官方服务，也不要把 PC1 当作普通 Ubuntu 开发机使用。

## 3. 开发前准备

### 3.1 实机安全条件

> **高风险提示：** 官方 `g1_ankle_swing_example` 会向多个关节发送底层控制指令。官方快速开发页明确要求：运行该例程前将机器人可靠悬挂。

首次运行会改变机器人姿态或关节状态的程序前，至少满足以下条件：

- 机器人型号、自由度版本和示例相符；
- 控制模式与程序层级匹配：高层 RPC 保持官方主运控可用，底层关节控制则进入开发调试模式并释放主运控；
- 机器人已可靠悬挂或处于适合该例程的安全支撑状态；
- 急停装置可立即触达，周围无人员和障碍物；
- 只有一个控制程序占用运动控制权；
- 操作者已经明确程序的停止方式；
- 先完成网络、状态读取和无运动查询，再发送运动指令。

除“悬挂后运行底层例程”外，其余条目是工程安全建议。它们不能替代设备手册、现场风险评估或宇树技术支持的要求。

### 3.2 官方基线环境

| 项目 | 官方基线或要求 | 说明 |
| --- | --- | --- |
| 操作系统 | Ubuntu 20.04 LTS | G1 快速开发页说明暂不支持原生 Windows、macOS 开发 |
| CPU 架构 | `x86_64` 或 `aarch64` | 仓库按 CPU 架构提供预编译 SDK/DDS 库 |
| 编译器 | GCC/G++ 9.4.0 | 官方 README 的预构建环境 |
| CMake | 3.10 或更高 | 官方 README 给出的使用基线 |
| C++ 标准 | C++17 | 仓库顶层 `CMakeLists.txt` 强制启用 |
| 网络 | 独立有线以太网，`192.168.123.X` 网段 | 新手优先使用网线接入 G1 交换机 |

### 3.3 Windows 用户说明

官方快速开发页明确写明当前不支持 Windows。对于首次实机开发，建议使用以下任一方案：

1. 独立 Ubuntu 20.04 电脑；
2. 双系统中的 Ubuntu 20.04；
3. 经宇树确认后，在 G1-EDU 的 PC2 上开发。

WSL2、虚拟机和容器可能受到虚拟网卡、组播转发或 USB 网卡直通影响，不应作为第一次实机联调的基准环境。这是工程建议，并非官方支持声明。

## 4. 依赖库与工具

### 4.1 依赖说明

官方仓库当前给出的 Ubuntu 安装包如下。

| 软件包 | 作用 | 与 G1 SDK 的关系 |
| --- | --- | --- |
| `git` | 下载和更新源码、记录版本 | 本文克隆仓库所需；不是 SDK 链接依赖 |
| `cmake` | 生成构建系统、查找依赖、生成安装配置 | 官方要求 3.10 或更高 |
| `g++` | GNU C++ 编译器 | 官方基线版本为 9.4.0 |
| `build-essential` | Ubuntu C/C++ 基础构建工具集合 | 通常包含 `make`、编译工具和基础开发文件 |
| `libyaml-cpp-dev` | C++ YAML 解析库 | G1 双臂底层例程检测 `yaml-cpp >= 0.6`；缺失或版本过低时该例程会被跳过 |
| `libeigen3-dev` | 矩阵、向量和几何运算 | 机器人算法常用数学依赖，列入官方安装命令 |
| `libboost-all-dev` | Boost C++ 库集合 | G1 的 `g1_termination` 和 `g1_arm_action_example` 使用 `program_options`；缺失时相关目标不会生成 |
| `libfmt-dev` | 类型安全的字符串格式化库 | 列入官方 README 的依赖安装命令 |

`unitree_sdk2` 当前仓库已经按 `x86_64` 和 `aarch64` 打包 DDS C/C++ 共享库，即 `libddsc.so` 和 `libddscxx.so`。因此，本文的 C++ 主线不另外安装系统级 Cyclone DDS。

### 4.2 安装依赖

~~~bash
sudo apt update
sudo apt install -y git cmake g++ build-essential \
  libyaml-cpp-dev libeigen3-dev libboost-all-dev libfmt-dev
~~~

命令解释：

| 命令片段 | 含义 |
| --- | --- |
| `sudo apt update` | 刷新 Ubuntu 软件包索引，不升级已安装软件 |
| `sudo apt install` | 从 Ubuntu 软件源安装指定软件包 |
| `-y` | 自动确认安装提示；如需逐项确认，可去掉 |
| 行末 `\` | Bash 续行符，只为提高可读性 |

安装后检查关键版本：

~~~bash
gcc --version
g++ --version
cmake --version
git --version
uname -m
~~~

期望 `uname -m` 输出 `x86_64` 或 `aarch64`。若为其他架构，SDK 中可能没有匹配的预编译库。

## 5. 网络环境配置

### 5.1 推荐连接方式

~~~text
Ubuntu 开发机的有线网卡
        │
        │ 网线/USB 转以太网
        ▼
    G1 内部交换机 ── 机器人服务与 PC2
~~~

初次联调应优先使用有线连接。不要先从 Wi-Fi、虚拟网卡或复杂路由开始排查 DDS。

### 5.2 地址含义

| 地址 | 官方文档中的含义 | 使用建议 |
| --- | --- | --- |
| `192.168.123.99` | 快速开发页推荐的用户电脑地址 | 适合作为开发机静态地址，但必须确认未被占用 |
| `192.168.123.222` | 快速开发页配置截图对应的用户电脑示例地址 | 可作为替代示例，同样需要避免地址冲突 |
| `192.168.123.161` | 快速开发页用于 `ping` 的机器人机载端地址 | 用于检查开发机到机器人网络的基本连通性 |
| `192.168.123.164` | 软件架构页给出的 G1-EDU PC2 地址 | 用于 PC2 二次开发；登录凭据需联系技术支持 |

`.161` 与 `.164` 是文档中用途不同的两个地址，不能互相替换。不同批次、型号或固件配置可能变化，最终以设备现场配置和最新官方资料为准。

### 5.3 使用 Ubuntu 图形界面配置

只修改连接 G1 的有线网卡，不要误改仍负责上网的 Wi-Fi 或其他生产网络。

官方快速开发页只要求“与 `192.168.123.X` 同网段”，没有在正文中固定子网掩码、网关和 DNS。下面采用常见的 `/24` 独立直连配置作为示例；若现场网络另有规划，应以设备实际配置或网络管理员要求为准：

1. 打开“设置 → 网络 → 有线连接 → 齿轮图标”；
2. 进入 IPv4；
3. 选择“手动”；
4. 地址填写 `192.168.123.99`；
5. 示例子网掩码填写 `255.255.255.0`，即 `/24`；
6. 对独立机器人直连网卡，网关和 DNS 通常留空；
7. 保存后断开并重新连接该有线连接。

### 5.4 使用 `nmcli` 配置（可选）

先查出“连接名称”和“设备名称”：

~~~bash
nmcli -f NAME,DEVICE,TYPE connection show
ip -br addr
~~~

下面以连接名称 `G1-Wired` 为例。请替换成你机器上真实存在的连接名称：

~~~bash
sudo nmcli connection modify "G1-Wired" \
  ipv4.method manual \
  ipv4.addresses 192.168.123.99/24 \
  ipv4.gateway "" \
  ipv4.dns "" \
  ipv4.never-default yes

sudo nmcli connection up "G1-Wired"
~~~

`ipv4.never-default yes` 可防止机器人专用网卡抢占默认上网路由。如果电脑通过该网口还承担其他网络任务，应由网络管理员确认配置。

### 5.5 验证网络

~~~bash
ip -br addr
ip route
ping -c 4 192.168.123.161
~~~

检查点：

- 机器人网卡拥有 `192.168.123.X/24` 地址；
- `192.168.123.0/24` 路由指向正确的有线网卡；
- `ping` 没有 `Destination Host Unreachable`；
- 记录该网卡名称，例如 `enp3s0` 或 `enxf8e43b808e06`。

官方页面使用 `ifconfig` 查看网卡。现代 Ubuntu 默认可直接使用 `ip -br addr`，无需为了这一项额外安装 `net-tools`。

## 6. 获取、编译与安装 SDK

### 6.1 获取官方仓库

~~~bash
git clone https://github.com/unitreerobotics/unitree_sdk2.git
cd unitree_sdk2
git rev-parse HEAD
~~~

最后一条命令打印当前提交号。建议将该提交号记录到实验日志中，避免以后 `main` 分支更新导致同一项目无法复现。

### 6.2 编译仓库内示例

~~~bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j"$(nproc)"
~~~

这与官方文档中的以下写法等价：

~~~bash
mkdir build
cd build
cmake ..
make
~~~

命令解释：

| 命令 | 作用 | 典型输出 |
| --- | --- | --- |
| `cmake -S . -B build` | 读取当前目录的 `CMakeLists.txt`，在 `build` 中生成构建文件 | 检测编译器、CPU 架构和预编译 SDK 库 |
| `-DCMAKE_BUILD_TYPE=Release` | 使用发布优化配置 | 当前仓库未显式指定时也默认选择 Release |
| `cmake --build build` | 调用生成的底层构建工具完成编译 | 库和 G1 示例可执行文件 |
| `-j"$(nproc)"` | 按 CPU 逻辑核心数并行编译 | 仅提高编译速度，不改变程序行为 |

编译成功后查看 G1 示例：

~~~bash
find build/bin -maxdepth 1 -type f -name 'g1*' -printf '%f\n' | sort
~~~

常见目标包括：

- `g1_loco_client`：高层运动 Client；
- `g1_ankle_swing_example`：底层踝关节摆动例程；
- `g1_arm5_sdk_dds_example` / `g1_arm7_sdk_dds_example`：手臂 DDS 例程；
- `g1_audio_client_example`：音频接口例程；
- `g1_dex3_example`：Dex3-1 灵巧手例程。

具体生成项受机器人版本和已安装依赖影响，应以本机 `build/bin` 的结果为准。

### 6.3 安装 SDK 供自己的 CMake 工程使用

安装到系统默认前缀（常见 Unix CMake 默认值为 `/usr/local`，并不是 `/opt/unitree_robotics`）：

~~~bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j"$(nproc)"
sudo cmake --install build
~~~

官方文档给出的等价安装命令是从 `build` 目录执行 `sudo make install`。

如果希望与系统目录隔离，可安装到官方 README 示例路径：

~~~bash
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX=/opt/unitree_robotics
cmake --build build -j"$(nproc)"
sudo cmake --install build
~~~

在当前终端让 CMake 和动态链接器找到该前缀：

~~~bash
export CMAKE_PREFIX_PATH=/opt/unitree_robotics:$CMAKE_PREFIX_PATH
export LD_LIBRARY_PATH=/opt/unitree_robotics/lib:$LD_LIBRARY_PATH
~~~

- `CMAKE_PREFIX_PATH`：帮助 `find_package(unitree_sdk2)` 找到 CMake 包配置；
- `LD_LIBRARY_PATH`：帮助运行时找到 `libddsc.so` 和 `libddscxx.so`。

确认验证通过后，再决定是否把这两个设置写入 shell 启动文件；不要在尚未确认路径时直接永久修改环境。

## 7. 机器人侧准备与例程运行

### 7.1 运行前检查顺序

1. 检查电量、急停、安全支撑和周边空间；
2. 确认机器人网卡名称并完成无运动的网络检查；
3. 先确定程序属于高层 RPC 还是底层关节控制；
4. 高层 RPC 应使用官方主运控服务，不要为一次高层状态查询先释放主运控；
5. 底层控制必须按官方《快速开始》和《底层运动开发》进入调试模式、确认主运控停止发送指令；
6. 先运行只读状态查询，最后才运行会驱动关节的例程。

G1 开机后，内置运动控制程序会周期发送指令。如果开发程序同时发布底层 `rt/lowcmd`，可能形成指令竞争并引发机器人抖动。因此，调试模式是**底层直接控制**的关键前置条件，而不是所有 SDK 功能的统一前置条件。

### 7.2 高层 Client 的只读查询

当前仓库的 `g1_loco_client` 支持通过命令行传入网卡并查询状态。例如：

~~~bash
./build/bin/g1_loco_client \
  --network_interface=enxf8e43b808e06 \
  --get_fsm_id
~~~

请替换网卡名称。`--get_fsm_id` 查询当前有限状态机 ID，不是行走命令，因此适合在运动类调用前验证高层 Client 通路。即便如此，也应保证机器人现场安全。

### 7.3 底层踝关节例程

当前官方调试资料给出的底层开发检查顺序是：

1. 将机器人固定在保护支架上，并锁死支架底部四个万向轮；
2. 对踝关节摆动例程进一步将机器人可靠悬挂；
3. 按官方《快速开始》的开发调试视频操作；
4. 可按 `L2 + A` 检查是否已进入调试模式；
5. 若机器人行为与视频不符，可多次按 `L2 + R2`，重新确保进入调试模式；
6. 确认主运控不再与底层程序竞争控制权。

遥控器流程可能随机器人版本和固件改变。上述按键来自本文核对日期的官方页面，实机仍应以对应型号的最新页面和调试视频为准。

仅完成上述检查后运行：

~~~bash
./build/bin/g1_ankle_swing_example enxf8e43b808e06
~~~

参数解释：

- `g1_ankle_swing_example`：会发布底层关节指令；
- 最后的 `enxf8e43b808e06`：连接 G1 的真实网卡名称，不是 IP 地址；
- 少传该参数时，程序只打印用法并退出。

不要把 `lo`、Wi-Fi 网卡或 Docker/虚拟机网卡误传给实机程序。

## 8. 底层例程的核心调用链

官方 `g1_ankle_swing_example.cpp` 的主要逻辑可以简化为：

~~~text
main(network_interface)
  └─ G1Example(network_interface)
      ├─ ChannelFactory::Init(0, network_interface)
      ├─ MotionSwitcherClient：检查并释放已有运动控制模式
      ├─ 订阅 rt/lowstate
      ├─ 订阅 rt/secondary_imu
      ├─ 2 ms 周期计算关节目标
      └─ 发布 rt/lowcmd
~~~

| 话题 | 方向 | 作用 |
| --- | --- | --- |
| `rt/lowstate` | 机器人 → 程序 | 关节位置、速度、估计力矩、温度、故障状态等 |
| `rt/secondary_imu` | 机器人 → 程序 | 躯干 IMU 姿态和角速度 |
| `rt/lowcmd` | 程序 → 机器人 | 目标位置、速度、`Kp`、`Kd`、前馈力矩等底层指令 |

该例程还会检查状态消息 CRC，并在启动底层控制前通过 `MotionSwitcherClient` 尝试释放现有运动模式。这一自动动作不能代替保护支架、悬挂、急停和人工模式确认。“DDS 能收到数据”也不等于“可以立即发送关节指令”，控制权切换与安全状态是独立前置条件。

## 9. 创建自己的最小 CMake 工程

安装 SDK 后，可使用如下最小 `CMakeLists.txt`：

~~~cmake
cmake_minimum_required(VERSION 3.10)
project(g1_demo LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

find_package(unitree_sdk2 REQUIRED)

add_executable(g1_demo src/main.cpp)
target_link_libraries(g1_demo PRIVATE unitree_sdk2)
~~~

工程结构：

~~~text
g1_demo/
├── CMakeLists.txt
└── src/
    └── main.cpp
~~~

编译：

~~~bash
cd g1_demo
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j"$(nproc)"
~~~

如果 SDK 安装在自定义前缀，也可以只对本次配置显式传参：

~~~bash
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_PREFIX_PATH=/opt/unitree_robotics
cmake --build build -j"$(nproc)"
~~~

建议从官方同类例程复制最小的初始化、超时设置和消息类型使用方式，再逐步删减；不要直接从底层运动例程复制增益和目标值用于另一型号或自由度版本。

## 10. 常见问题

| 现象 | 常见原因 | 建议检查 |
| --- | --- | --- |
| CMake 报 `Unitree SDK library for the architecture is not found` | 当前 CPU 架构没有对应预编译库 | 运行 `uname -m`；确认是 `x86_64` 或 `aarch64`，检查仓库是否完整 |
| 找不到 `unitree_sdk2Config.cmake` | SDK 未安装或自定义前缀未加入搜索路径 | 检查安装位置；设置 `CMAKE_PREFIX_PATH` 或配置时传 `-DCMAKE_PREFIX_PATH=...` |
| 运行时报找不到 `libddsc.so` | 动态库目录不在运行时搜索路径 | 检查安装是否完整；为自定义前缀设置 `LD_LIBRARY_PATH` |
| 能看到网卡但 `ping` 不通 | 网段、掩码、线缆、USB 网卡或地址冲突 | 检查 `ip -br addr`、`ip route`、接口链路状态和机器人上电状态 |
| `ping` 通但 DDS 无数据 | 程序选错网卡、存在多网卡/虚拟网卡干扰，或组播被网络策略阻断 | 显式传入机器人有线网卡；检查防火墙和网络策略，不要未经评估直接关闭安全功能 |
| 没有生成 `g1_dual_arm_example` | `yaml-cpp` 缺失或版本低于 0.6 | 检查 CMake 输出和 `dpkg -s libyaml-cpp-dev` |
| 没有生成 `g1_termination` 或 `g1_arm_action_example` | 未找到 Boost `program_options` | 确认安装 `libboost-all-dev`，重新配置并编译 |
| 低层例程没有动作 | 未进入正确模式、控制权未释放、通信未建立或安全状态阻止运动 | 不要反复发送命令；停止程序，逐项核对机器人模式、状态话题、网卡和官方调试流程 |
| 持续出现 CRC 错误或状态异常 | 消息不完整、网络异常，或 SDK/固件/机型不匹配 | 停止运动测试，记录 SDK 提交号、固件与机型，联系技术支持核对 |

## 11. 推荐学习和验证顺序

| 阶段 | 目标 | 通过标准 |
| --- | --- | --- |
| 1. 离线构建 | 环境和依赖正确 | 全仓库示例编译完成 |
| 2. 网络检查 | 开发机能访问机器人 | 静态地址、路由和 `ping` 正常 |
| 3. 状态查询 | DDS/高层 Client 链路正确 | 能稳定返回机器人状态，无运动 |
| 4. 高层控制 | 理解官方封装接口 | 在安全场地完成单一、低风险动作 |
| 5. 底层状态读取 | 理解消息、关节顺序和频率 | 只订阅并记录状态，不发控制命令 |
| 6. 底层控制 | 验证控制权、周期和安全限制 | 悬挂条件下运行官方原始例程 |
| 7. 自定义算法 | 逐步替换官方目标值和逻辑 | 有限幅、超时、异常退出和回归记录 |

## 12. 快速命令清单

~~~bash
# 1. 安装依赖
sudo apt update
sudo apt install -y git cmake g++ build-essential \
  libyaml-cpp-dev libeigen3-dev libboost-all-dev libfmt-dev

# 2. 获取 SDK
git clone https://github.com/unitreerobotics/unitree_sdk2.git
cd unitree_sdk2
git rev-parse HEAD

# 3. 编译示例
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j"$(nproc)"

# 4. 查看网络并测试机器人地址
ip -br addr
ip route
ping -c 4 192.168.123.161

# 5. 查看生成的 G1 程序
find build/bin -maxdepth 1 -type f -name 'g1*' -printf '%f\n' | sort

# 6. 先做高层只读查询：替换网卡名
./build/bin/g1_loco_client \
  --network_interface=enxf8e43b808e06 \
  --get_fsm_id

# 7. 高风险：仅在正确调试模式且机器人可靠悬挂后运行
./build/bin/g1_ankle_swing_example enxf8e43b808e06
~~~

## 13. 限制与版本注意事项

- SDK 概述页标注“当前软件版本还不支持 GST 视频流传输”，但该页面更新时间较早。涉及图传时，应以当前固件和最新官方接口页为准。
- 官方架构页说明 G1 的 DDS IDL 可兼容 ROS 2，但需要选择匹配的 RMW；这不等于任意 ROS 2 发行版和 RMW 组合都可直接使用。
- `unitree_sdk2` 仓库包含预编译库。SDK 提交、CPU 架构、机器人型号和固件版本必须一并记录。
- 官方 README 仍提到 `example/cmake_sample`，但当前 `main` 分支不存在该目录；本文给出了可核查的最小 CMake 配置，不依赖这个失效路径。
- 本文没有提供 PC2 登录凭据或自定义关节增益；这些信息应从与设备版本匹配的官方资料或技术支持获取。

## 14. 官方参考资料

1. [G1 SDK 概述](https://support.unitree.com/home/zh/G1_developer/sdk_overview)
2. [G1 软件架构说明](https://support.unitree.com/home/zh/G1_developer/architecture_description)
3. [G1 获取 SDK](https://support.unitree.com/home/zh/G1_developer/get_sdk)
4. [G1 快速开发](https://support.unitree.com/home/zh/G1_developer/quick_development)
5. [G1 快速开始](https://support.unitree.com/home/zh/G1_developer/quick_start)
6. [G1 DDS 通信接口](https://support.unitree.com/home/zh/G1_developer/dds_services_interface)
7. [G1 底层运动开发](https://support.unitree.com/home/zh/G1_developer/basic_motion_development)
8. [G1 调试说明](https://support.unitree.com/home/zh/G1_developer/debugging_specification)
9. [G1 运控切换接口](https://support.unitree.com/home/zh/G1_developer/motion_witcher_service_interface)
10. [unitree_sdk2 官方仓库](https://github.com/unitreerobotics/unitree_sdk2)
11. [unitree_sdk2 README](https://github.com/unitreerobotics/unitree_sdk2/blob/main/README.md)
12. [unitree_sdk2 顶层 CMakeLists.txt](https://github.com/unitreerobotics/unitree_sdk2/blob/main/CMakeLists.txt)
13. [G1 示例目标清单](https://github.com/unitreerobotics/unitree_sdk2/blob/main/example/g1/CMakeLists.txt)
14. [G1 底层踝关节例程源码](https://github.com/unitreerobotics/unitree_sdk2/blob/main/example/g1/low_level/g1_ankle_swing_example.cpp)

---

**实机原则：先证明网络与状态读取正确，再证明模式与控制权正确，最后才允许关节运动。**
