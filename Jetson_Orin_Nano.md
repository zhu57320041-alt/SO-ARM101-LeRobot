# Jetson Orin Nano + SO-ARM101 + LeRobot 项目全过程记录

## 一、项目目标

本项目旨在 NVIDIA Jetson Orin Nano 上部署 LeRobot，实现 SO-ARM101 机械臂控制、双摄像头数据采集、实时可视化，并为后续机器人模型训练准备数据集。

整体工作分为以下几个阶段：

1. Jetson 系统与 Docker 环境准备。
2. LeRobot 运行环境部署。
3. SO-ARM101 Leader / Follower 连接与校准。
4. 双摄像头识别与并发采集。
5. 双臂遥操作。
6. Rerun 实时可视化。
7. LeRobot Dataset 数据集录制。
8. 数据集检查与模型训练。

目前已经完成前六项的主要工作，并成功录制了 25 个数据集。当前重点转向数据集完整性检查、采集质量评估及后续模型训练准备。

---

## 二、硬件与软件环境

### 2.1 硬件环境

| 硬件   | 配置                          |
| ---- | --------------------------- |
| 主控设备 | NVIDIA Jetson Orin Nano     |
| 内存   | 8GB                         |
| 机械臂  | SO-ARM101 Leader + Follower |
| 摄像头一 | 海康威视 2K USB Camera          |
| 摄像头二 | 普通 USB Camera               |

### 2.2 软件环境

| 项目            | 版本或配置                                       |
| ------------- | ------------------------------------------- |
| 操作系统          | Ubuntu 22.04.5                              |
| JetPack       | 6.2.1                                       |
| L4T           | 36.4.7                                      |
| Docker 镜像     | `ghcr.io/ravediamond/lerobot-jetson:latest` |
| LeRobot       | 0.6.2                                       |
| 宿主机 Python    | 3.10                                        |
| Docker Python | 3.12                                        |
| Rerun SDK     | 0.39.0                                      |

Docker 容器内的主要工作目录：

```text
/opt/lerobot
```

宿主机 Rerun 虚拟环境：

```text
/home/nvidia/rerun-env
```

---

# 三、项目实施过程

## 阶段一：Jetson 环境准备与 LeRobot 部署

### 1. 工作内容

首先在 Jetson Orin Nano 上准备机器人运行环境。

考虑到 Jetson 的 ARM64 架构、CUDA 环境和依赖兼容性，采用 Docker 方式部署 LeRobot，而不是直接在宿主机安装全部依赖。

使用的镜像：

```text
ghcr.io/ravediamond/lerobot-jetson:latest
```

启动容器时使用 NVIDIA Runtime，并配置设备访问和网络：

```bash
sudo docker run -it --rm --runtime=nvidia --network host --privileged ...
```

主要配置包括：

* NVIDIA GPU 运行时。
* Host 网络模式。
* 串口设备访问。
* USB 摄像头设备访问。
* Hugging Face LeRobot 缓存目录挂载。

### 2. 遇到的问题

#### 问题一：Jetson 环境与普通 PC 不同

Jetson 使用 ARM64 架构，不能简单照搬普通 x86_64 Ubuntu 电脑的安装方式。

**解决方案：**

采用针对 Jetson 环境的 Docker 镜像，将 LeRobot 及其运行依赖放入容器，减少宿主机环境污染。

#### 问题二：宿主机与容器 Python 版本不同

* 宿主机：Python 3.10。
* Docker 容器：Python 3.12。

**解决方案：**

明确区分宿主机和容器的运行环境。

后续机械臂控制、摄像头采集和 LeRobot 命令均在容器内执行；Rerun Viewer 则在宿主机运行。

---

## 阶段二：SO-ARM101 机械臂连接与校准

### 1. 工作内容

完成 Leader 和 Follower 两台机械臂的连接、串口识别及校准。

设备串口对应关系：

| 设备             | 串口             | 设备 ID                     |
| -------------- | -------------- | ------------------------- |
| Follower       | `/dev/ttyACM1` | `my_awesome_follower_arm` |
| Leader         | `/dev/ttyACM2` | `my_awesome_leader_arm`   |
| 摄像头 CDC ACM 节点 | `/dev/ttyACM0` | 非机械臂                      |

完成两台机械臂的校准，并保存校准文件：

```text
my_awesome_leader_arm.json
my_awesome_follower_arm.json
```

校准文件位于 LeRobot 缓存目录：

```text
/root/.cache/huggingface/lerobot/calibration
```

宿主机缓存目录已经通过 Docker Volume 挂载到容器。

### 2. 遇到的问题

#### 问题一：串口设备容易混淆

系统中存在多个 `/dev/ttyACM*` 设备，不能直接认为 ACM0、ACM1、ACM2 都是机械臂。

**解决方案：**

通过设备识别和实际连接测试确认：

* ACM1 对应 Follower。
* ACM2 对应 Leader。
* ACM0 是摄像头复合设备的 CDC ACM 节点。

#### 问题二：校准文件需要保留

容器使用 `--rm` 参数时，退出后容器本身会被删除，容器内部未挂载的数据可能随之丢失。

**解决方案：**

将校准文件保存到宿主机挂载的 Hugging Face 缓存目录，避免重建容器后重新校准。

### 3. 阶段成果

* Leader 校准完成。
* Follower 校准完成。
* 两台机械臂均能正常连接。
* Leader 能够正常控制 Follower。

目前发现第六号舵机 Wrist Roll 存在固定角度偏差，暂时决定不处理。

---

## 阶段三：双摄像头识别与并发采集

这是整个项目中排查时间较长的部分之一。

### 1. 摄像头设备识别

通过 V4L2 识别摄像头设备：

```bash
v4l2-ctl --list-devices
```

识别结果：

```text
NVIDIA Tegra Video Input Device (platform:tegra-camrtc-ca):
    /dev/media0

2K USB Camera: 2K USB Camera (usb-3610000.usb-2.1):
    /dev/video0
    /dev/video1
    /dev/media1

USB Camera: USB Camera (usb-3610000.usb-2.3.2):
    /dev/video2
    /dev/video3
    /dev/media2
```

确认两台物理摄像头分别对应：

| 摄像头           | 视频节点                        |
| ------------- | --------------------------- |
| 海康威视 2K       | `/dev/video0`、`/dev/video1` |
| 普通 USB Camera | `/dev/video2`、`/dev/video3` |

每台摄像头出现两个视频节点，并不意味着存在两台独立摄像头。

### 2. 初始测试

分别测试两台摄像头的独立采集能力。

结果：

| 摄像头           | 独立采集能力             |
| ------------- | ------------------ |
| 海康威视 2K       | 2560×1440 @ 30 FPS |
| 普通 USB Camera | 1920×1080 @ 30 FPS |

单独使用时，两台摄像头均能正常工作。

但是，同时启动两台摄像头时出现问题：

* 第二路摄像头读取速度明显下降。
* 双摄并发不稳定。
* FFmpeg 出现 I/O 错误。

### 3. 排查过程

#### 问题一：是否是摄像头本身故障？

分别测试两台摄像头。

结果：单独运行均正常。

因此，暂时排除摄像头单体故障。

#### 问题二：是否是 OpenCV 设备编号错误？

通过 OpenCV 测试：

```python
import cv2

cap = cv2.VideoCapture(0)
print(cap.isOpened())
```

发现两台摄像头均能够被识别，但第二个打开的设备容易出现读取失败。

交换摄像头打开顺序后，问题仍然跟随第二个打开的设备。

因此，问题并不是固定发生在某一台摄像头上。

#### 问题三：是否是 USB 接口问题？

通过：

```bash
lsusb -t
```

检查 USB 拓扑结构。

发现两台摄像头都挂载在同一个 USB2 Root Hub 下：

```text
Bus 01
USB2 Root Hub (480M)
```

随后尝试将普通摄像头更换到 USB3 接口，但问题仍未解决。

因此，USB 带宽或 UVC 资源分配仍然值得怀疑，但尚不能确定唯一原因。

#### 问题四：是否可以通过降低分辨率解决？

逐步降低摄像头分辨率，并调整采集参数。

最终找到可以稳定运行的配置。

### 4. 最终解决方案

目前验证成功的双摄参数：

| 参数      | 前置摄像头         | 腕部摄像头         |
| ------- | ------------- | ------------- |
| 设备节点    | `/dev/video0` | `/dev/video2` |
| 分辨率     | 640×480       | 640×480       |
| FPS     | 30            | 30            |
| FourCC  | MJPG          | MJPG          |
| Backend | V4L2（200）     | V4L2（200）     |

最终结论：

两台摄像头均采用 640×480、30 FPS、MJPG 编码，可以同时正常采集。

**尚未解决的问题：** 双摄高分辨率并发采集失败的具体原因仍未完全确定。

### 5. 后续分辨率与画面质量测试

为了改善 Rerun 显示延迟，曾尝试将摄像头分辨率进一步降低至 320×240。

但测试过程中发现摄像头的最低可用分辨率受到设备支持的格式限制，无法直接认定 OpenCV 请求的分辨率就是摄像头实际输出的分辨率。

随后提出尝试 1080P，进一步核实实际输出格式。

但该测试最终暂时搁置。

因此，目前尚未确认：

* LeRobot 请求的分辨率是否与摄像头实际输出完全一致。
* Rerun 显示画面是否经过缩放。
* 显示延迟究竟主要来自采集、传输、解码还是渲染。

---

## 阶段四：SO-ARM101 双臂遥操作

### 1. 工作内容

在机械臂和双摄像头均正常连接后，使用 LeRobot 的 `lerobot-teleoperate` 启动遥操作。

最终成功命令：

```bash
lerobot-teleoperate --robot.type=so101_follower --robot.port=/dev/ttyACM1 --robot.id=my_awesome_follower_arm --teleop.type=so101_leader --teleop.port=/dev/ttyACM2 --teleop.id=my_awesome_leader_arm --robot.cameras="{front: {type: opencv, index_or_path: /dev/video0, width: 640, height: 480, fps: 30, fourcc: MJPG, backend: 200}, wrist: {type: opencv, index_or_path: /dev/video2, width: 640, height: 480, fps: 30, fourcc: MJPG, backend: 200}}" --display_data=false
```

### 2. 阶段成果

成功实现：

* Leader 实时控制 Follower。
* 两台摄像头同时采集。
* 机械臂与摄像头同时运行。
* 遥操作循环约 59 Hz。

至此，机械臂控制和双摄采集的基本功能已经打通。

---

## 阶段五：Rerun 可视化部署

### 1. 工作目标

在机械臂遥操作过程中，实时显示：

* 前置摄像头画面。
* 腕部摄像头画面。
* 机械臂相关状态数据。

最初尝试直接在 Docker 容器内部启动 Rerun Viewer。

### 2. 遇到的问题

容器内运行 GUI 时出现：

```text
neither WAYLAND_DISPLAY nor WAYLAND_SOCKET nor DISPLAY is set
```

原因是 Docker 容器没有可用的桌面显示环境。

虽然机械臂和摄像头已经能够正常连接，但容器内部无法直接弹出 Viewer。

### 3. 解决方案：宿主机启动 Viewer，容器发送数据

经过调整，采用宿主机与 Docker 容器分离的方式：

```text
Jetson 宿主机
    │
    └── Rerun Viewer
          │
          │ 127.0.0.1:9876
          │
Docker 容器
    │
    └── LeRobot
          ├── SO-ARM101
          ├── 前置摄像头
          └── 腕部摄像头
```

#### 第一步：创建宿主机 Python 虚拟环境

```bash
python3 -m venv ~/rerun-env
```

#### 第二步：安装 Rerun SDK

```bash
~/rerun-env/bin/python -m pip install --no-cache-dir --index-url=https://pypi.tuna.tsinghua.edu.cn/simple/ --trusted-host=pypi.tuna.tsinghua.edu.cn rerun-sdk
```

安装版本：

```text
rerun-sdk 0.39.0
```

#### 第三步：启动 Viewer

在 Jetson 宿主机执行：

```bash
~/rerun-env/bin/python -m rerun
```

这是实际验证成功的启动方式。

#### 第四步：确认端口监听

```bash
sudo ss -lntp | grep 9876
```

曾经成功得到：

```text
LISTEN 0 128 0.0.0.0:9876
```

说明 Viewer 已经监听 9876 端口。

#### 第五步：让 LeRobot 连接宿主机 Viewer

由于 Docker 使用：

```bash
--network host
```

容器可以通过：

```text
127.0.0.1:9876
```

访问宿主机的 Rerun Viewer。

在原来的遥操作命令中增加：

```bash
--display_data=true --display_ip=127.0.0.1 --display_port=9876
```

最终成功实现：

* Rerun Viewer 正常弹出。
* 前置摄像头画面正常显示。
* 腕部摄像头画面正常显示。
* Leader/Follower 机械臂仍能正常遥操作。

### 4. 当前问题

Rerun 已经可以正常显示，但画面存在较明显延迟。

目前还没有进一步优化。

另外，需要注意：

**Rerun 实时显示不等于 LeRobot 数据集录制。**

Rerun 的 `.rrd` 是其自身的记录格式，而 LeRobot Dataset 是另一套用于机器人训练的数据组织形式。

---

## 阶段六：LeRobot Dataset 数据集录制

### 1. 工作目标

在机械臂遥操作、双摄像头采集和 Rerun 可视化的基础上，进一步使用 LeRobot 的 `lerobot-record` 完成机器人数据集录制。

与单纯的实时遥操作不同，数据集录制需要将机械臂状态、动作以及摄像头图像等信息按照 LeRobot 的数据组织方式保存下来，为后续模型训练提供数据。

### 2. 实施结果

已经成功完成 LeRobot Dataset 录制，累计成功录制：

**25 个数据集。**

这意味着项目已经不只是完成机械臂控制和实时画面显示，而是进一步打通了从机器人遥操作到 LeRobot 数据集采集的基本流程。

### 3. 当前仍需验证的内容

成功录制数据集并不代表数据质量已经完全得到验证。

后续还需要进一步检查：

* Dataset 元数据是否完整。
* 机械臂状态数据是否正常保存。
* 动作数据是否正常保存。
* 两路摄像头视频是否完整。
* 视频帧率是否符合预期。
* 视频与机械臂状态、动作数据的时间戳是否对应。
* Episode 组织和数据结构是否符合预期。
* 数据集是否满足后续 ACT 等机器人模型的训练要求。

因此，当前阶段已经完成数据集录制，但数据完整性和训练适用性仍需进一步评估。

---

# 四、项目中遇到的主要问题及解决结果

| 序号 | 问题                     | 排查方式                        | 解决方案                  | 当前状态       |
| -- | ---------------------- | --------------------------- | --------------------- | ---------- |
| 1  | Jetson 环境部署困难          | 检查架构与依赖环境                   | 使用 Jetson Docker 镜像   | 已解决        |
| 2  | 宿主机与容器 Python 版本不同     | 分别确认 Python 环境              | 区分宿主机与容器操作            | 已解决        |
| 3  | 串口设备混淆                 | 识别 ACM 设备                   | 确认 ACM1、ACM2 对应机械臂    | 已解决        |
| 4  | 校准文件可能丢失               | 检查容器生命周期及缓存                 | 挂载宿主机 Hugging Face 缓存 | 已解决        |
| 5  | 双摄同时采集失败               | OpenCV、FFmpeg、V4L2、USB 拓扑排查 | 降低至 640×480@30 MJPG   | 已解决（降低分辨率） |
| 6  | 第二个打开的摄像头读取失败          | 交换打开顺序测试                    | 确认不是固定某一台摄像头故障        | 已找到可用配置    |
| 7  | Docker 内无法启动 Rerun GUI | 检查显示环境变量                    | Viewer 改在宿主机运行        | 已解决        |
| 8  | 容器无法连接 Viewer          | 检查网络模式及端口                   | Host 网络 + 9876 端口     | 已解决        |
| 9  | Rerun 显示延迟较高           | 尚未深入排查                      | 暂时保留现有配置              | 待优化        |
| 10 | 数据集录制流程                 | 实际运行 `lerobot-record`        | 已成功录制 25 个数据集         | 已完成        |
| 11 | 数据集完整性检查               | 检查视频、状态数据、动作数据及元数据          | 进一步核验数据质量             | 待验证        |

---

# 五、当前项目状态

## 5.1 已经完成

- [x] Jetson Orin Nano 环境准备
- [x] Docker 环境部署
- [x] LeRobot 0.6.2 运行环境
- [x] SO-ARM101 Leader 连接
- [x] SO-ARM101 Follower 连接
- [x] Leader 校准
- [x] Follower 校准
- [x] 双臂遥操作
- [x] 海康威视摄像头识别
- [x] 普通 USB 摄像头识别
- [x] 双摄并发采集（640×480）
- [x] Rerun SDK 安装
- [x] Rerun Viewer 启动
- [x] LeRobot 连接 Rerun
- [x] 双摄画面实时可视化
- [x] LeRobot Dataset 录制流程
- [x] 成功录制 25 个数据集

## 5.2 尚未完成

- [ ] Rerun 延迟优化
- [ ] 双摄高分辨率并发问题解决
- [ ] 摄像头实际输出分辨率核实
- [ ] 数据集元数据完整性检查
- [ ] 机械臂状态与动作数据检查
- [ ] 两路摄像头视频完整性检查
- [ ] 视频与动作数据时间戳同步检查
- [ ] Episode 组织及录制质量验证
- [ ] 数据集质量评估
- [ ] ACT 等机器人模型训练
- [ ] 模型部署与实际操作验证

---

# 六、下一阶段工作计划

尝试进入模型训练。

验证数据完整性与可靠性。

### 第一阶段：正式数据集录制（已完成）

使用：

```bash
lerobot-record --help
```

确认 LeRobot 0.6.2 的实际录制参数。

随后完成：

1. 配置 Leader、Follower。
2. 配置前置摄像头和腕部摄像头。
3. 配置 Dataset 保存位置及任务描述。
4. 录制一个短 Episode。
5. 检查录制过程中机械臂和摄像头是否正常。

### 第二阶段：数据集检查（已完成）

重点检查：

* 是否生成 Dataset 元数据。
* 是否保存机械臂状态。
* 是否保存动作数据。
* 是否生成两路摄像头视频。
* 视频帧率是否正常。
* 视频与动作数据是否对应。
* Episode 是否完整。

### 第三阶段：优化采集系统

在数据集录制成功的基础上，再考虑：

* Rerun 显示延迟。
* 双摄分辨率。
* USB 带宽。
* 摄像头采集稳定性。
* Jetson CPU、GPU、内存占用。

### 第四阶段：模型训练

当数据集质量满足要求后，再开展 LeRobot 模型训练与部署。

---

# 七、最终总结

从项目开始到现在，我们已经完成了 Jetson Orin Nano 上 LeRobot 运行环境的搭建，并打通了 SO-ARM101 双臂控制、双摄像头采集和 Rerun 实时可视化链路。

过程中最主要的两个技术问题是：

1. **双摄并发采集问题**：通过排查 OpenCV、FFmpeg、V4L2 和 USB 拓扑，最终采用 640×480@30 FPS、MJPG 编码，实现了双摄稳定运行，但高分辨率并发问题仍未彻底解决。
2. **Docker 内 Rerun GUI 无法启动问题**：通过将 Rerun Viewer 部署在 Jetson 宿主机、LeRobot 保留在 Docker 容器内，并利用 Host 网络连接，成功实现远程数据可视化。

目前已经实现的完整链路：

```text
SO-ARM101 Leader
        │
        │ 实时控制
        ▼
SO-ARM101 Follower
        │
        ├──────────────┐
        │              │
        ▼              ▼
前置摄像头          腕部摄像头
        │              │
        └──────┬───────┘
               │
               ▼
          LeRobot 0.6.2
               │
          ┌────┴────┐
          │         │
          ▼         ▼
     实时遥操作   Rerun Viewer
```

**项目当前所处阶段：机器人控制与实时可视化已经打通，已经成功录制 25 个数据集，正在进入数据集质量检查与模型训练准备阶段。**

首次撰写时间：2026-11-03
第一次增订：2026-11-13
