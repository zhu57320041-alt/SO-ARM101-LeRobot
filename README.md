# SO-ARM101 × LeRobot

## From WSL2 to Native Windows and NVIDIA Jetson Nano

一个关于 SO-ARM101 双机械臂部署、遥操作、视觉采集与 LeRobot 环境迁移的实践记录。

本项目记录了从最初在 Windows 11 的 WSL2 Ubuntu 环境中运行 LeRobot，到迁移至 Windows 原生环境，再到 NVIDIA Jetson Nano 部署的完整过程。

记录重点包括：

* SO-ARM101 Leader / Follower 双机械臂配置
* Feetech 舵机识别与零位校准
* LeRobot 环境搭建与遥操作
* Windows 原生环境迁移
* USB 摄像头采集与双摄像头并发问题
* Jetson Nano Docker 部署
* 视觉数据采集与 LeRobot Dataset 录制

---

# 1. Project Overview

### Hardware

| Hardware           | Description              |
| ------------------ | ------------------------ |
| SO-ARM101 Leader   | 主控机械臂                    |
| SO-ARM101 Follower | 从动机械臂                    |
| Servo              | Feetech Serial Bus Servo |
| Camera 1           | Hikvision 2K USB Camera  |
| Camera 2           | USB Camera               |
| Main Computer      | Windows 11 Laptop        |
| Edge Device        | NVIDIA Jetson Nano       |

### Software

| Software | Version / Environment |
| -------- | --------------------- |
| LeRobot  | 0.6.1 / 0.6.2         |
| Python   | 3.12.x                |
| Ubuntu   | WSL2                  |
| Windows  | Windows 11            |
| Conda    | Miniforge             |
| OpenCV   | 4.13.0                |
| FFmpeg   | 9.0.2                 |
| Docker   | Jetson deployment     |

### Project Architecture

项目经历了三个主要阶段：

1. WSL2 Ubuntu：完成机械臂底层配置和主从遥操作。
2. Native Windows：尝试脱离 WSL2，在 Windows 原生环境完成机械臂控制和视觉数据采集。
3. NVIDIA Jetson Nano：将 LeRobot 部署到边缘计算设备，探索本地视觉采集和数据集录制。

---

# 2. Stage 1 — WSL2 Ubuntu Deployment

## 2.1 Environment Setup

最初选择在 Windows 11 上通过 WSL2 运行 Ubuntu，并使用 Miniforge / Conda 管理 Python 环境。

主要环境信息：

* Ubuntu on WSL2
* Miniforge
* Conda environment: `lerobot`
* Python 3.12
* LeRobot 0.6.1

LeRobot 安装位置：

`/home/cache/miniforge3/envs/lerobot/lib/python3.12/site-packages`

通过 Python 环境运行 LeRobot 相关程序。

## 2.2 Servo Identification

开始遥操作之前，需要分别确认 Leader 和 Follower 机械臂的舵机连接状态。

使用串口连接机械臂控制板。

最终确认：

| Serial Port    | Device   |
| -------------- | -------- |
| `/dev/ttyACM0` | Leader   |
| `/dev/ttyACM1` | Follower |

在 1,000,000 baud 波特率下扫描舵机。

两台机械臂均成功识别 ID 1–6，型号编号为 777。

完成了两台机械臂的舵机识别工作。

## 2.3 Calibration

分别对 Leader 和 Follower 进行校准。

校准文件：

* `my_awesome_leader_arm.json`
* `my_awesome_follower_arm.json`

后续将这两份校准文件用于其他运行环境。

## 2.4 Teleoperation

完成舵机识别和校准后，进行主从机械臂遥操作测试。

最终实现：

* Leader 机械臂动作采集。
* Follower 机械臂动作跟随。
* 主从机械臂串口通信。
* 双机械臂遥操作正常运行。

**Result:**

成功在 WSL2 Ubuntu 环境下运行 SO-ARM101 主从遥操作。

这是整个项目第一个完整跑通的阶段。

---

# 3. Stage 2 — Native Windows Migration

## 3.1 Why Migrate?

WSL2 阶段已经实现了机械臂遥操作，但后续计划进一步开展：

* 摄像头视觉采集。
* 多摄像头同步采集。
* LeRobot Dataset 录制。
* 日常机械臂控制。

考虑到 USB 设备映射、摄像头访问和 Windows 本地工作流，希望直接在 Windows 原生环境运行 LeRobot。

因此开始从 WSL2 迁移到 Windows 11。

## 3.2 Environment Setup

使用 Miniforge 在 Windows 原生环境中建立独立 Conda 环境。

环境信息：

| Component | Version         |
| --------- | --------------- |
| OS        | Windows 11 25H2 |
| Python    | 3.12.16         |
| LeRobot   | 0.6.1           |
| Git       | 2.55.0          |
| Conda     | Miniforge       |
| OpenCV    | 4.13.0          |
| FFmpeg    | 9.0.2           |

Conda 安装目录：

`D:\Miniforge3`

独立环境：

`D:\Miniforge3\envs\lerobot-win`

Python 解释器：

`D:\Miniforge3\envs\lerobot-win\python.exe`

期间还排查了 Windows 上多个 Python 安装路径导致的环境混淆问题。

包括：

* Miniforge Base Python。
* 独立 Conda 环境 Python。
* 系统 Python 3.13.7。
* WindowsApps Python。

最终确认 `lerobot-win` 环境使用 Python 3.12.16。

## 3.3 Serial Port Configuration

Windows 环境下不再使用 Linux 的 `/dev/ttyACM*`。

机械臂控制板通过 USB 转串口设备连接 Windows。

设备识别为：

* CH343 USB Serial
* COM10

将原先 WSL2 中的校准文件复制到 Windows：

`C:\Users\31081\.cache\huggingface\lerobot\calibration\`

成功迁移：

* `my_awesome_leader_arm.json`
* `my_awesome_follower_arm.json`

避免重新进行完整校准。

## 3.4 Camera Configuration

Windows 下使用 OpenCV 检测 USB 摄像头。

检测到的设备：

| Camera ID | Resolution  |
| --------- | ----------- |
| 0         | 1920 × 1080 |
| 1         | 640 × 480   |
| 2         | 2560 × 1440 |
| 3         | 1280 × 720  |

其中一台为海康威视 2K USB 摄像头。

摄像头支持不同分辨率和帧率，但在双摄像头同时工作时出现了明显的性能问题。

## 3.5 Dual Camera Concurrency Issue

### Problem

两台 USB 摄像头单独运行时基本正常，但同时启动后，其中一路摄像头帧率严重下降。

实际观察：

* 单独运行摄像头时，均可以达到约 30 FPS。
* 双摄像头同时运行时，其中一路可能下降到约 1 FPS。
* FFmpeg 出现 I/O Error。
* 摄像头之间存在明显的资源竞争现象。

### Investigation

对以下方面进行了排查：

1. 摄像头设备编号。
2. 摄像头支持的分辨率。
3. 摄像头帧率。
4. USB 连接方式。
5. USB Root Hub 与设备之间的关系。
6. OpenCV 摄像头访问情况。
7. FFmpeg 视频采集情况。

还尝试检查两台摄像头是否共享 USB 控制器或带宽资源。

### Current Status

目前尚未完全确认双摄像头并发采集异常的根本原因。

单路摄像头采集可以正常使用，但双路同时采集的稳定性仍需要进一步研究。

### GitHub Issue

针对 Windows 环境下的双 USB 摄像头并发采集问题，整理并提交了 GitHub Issue。

主要描述：

* Windows 11 环境。
* OpenCV / FFmpeg。
* 两台 USB 摄像头。
* 单路正常、双路帧率异常下降。
* 不同分辨率下的测试结果。

---

# 4. Stage 3 — NVIDIA Jetson Nano Deployment

## 4.1 Deployment Motivation

在完成 Windows 原生环境迁移和摄像头测试后，开始尝试将 LeRobot 部署到 NVIDIA Jetson Nano。

希望将机械臂控制、视觉采集等任务逐步转移到边缘计算设备上，减少对个人电脑的依赖。

## 4.2 Docker Deployment

使用 Docker 容器运行 LeRobot。

采用的镜像：

`ghcr.io/ravediamond/lerobot-jetson:latest`

容器启动方式：

```bash
sudo docker run -it --rm --runtime=nvidia \
ghcr.io/ravediamond/lerobot-jetson:latest
```

进入容器后检查 LeRobot 版本。

实际输出：

```text
0.6.2
```

确认 Jetson Nano 容器内已经成功运行 LeRobot 0.6.2。

## 4.3 Camera Testing

开始在 Jetson Nano 上测试 USB 摄像头。

主要测试设备：

* `/dev/video0`
* `/dev/video2`

### Camera 1

设备：

`/dev/video0`

测试结果：

* 分辨率：1280 × 720
* 目标帧率：30 FPS
* 实测帧率：约 31.09 FPS

单路采集表现正常。

### Camera 2

设备：

`/dev/video2`

测试不同分辨率：

| Resolution | Observed FPS |
| ---------- | -----------: |
| 1280 × 720 |  约 16–18 FPS |
| 640 × 480  |  约 28.84 FPS |

虽然降低分辨率后帧率有所改善，但仍然存在视频卡顿现象。

因此，不能仅凭 FPS 数值判断视频采集是否正常。

双摄像头并发采集和实际录制稳定性仍需进一步验证。

## 4.4 Video Recording

进一步尝试使用 LeRobot 的数据录制功能。

重点了解：

* 如何启动录制界面。
* 如何配置摄像头。
* 如何开始和结束 Episode。
* 如何生成视频文件。
* 如何检查录制结果。
* 如何删除无用的录制数据。

逐步从单纯的摄像头测试转向实际 Dataset 采集。

---

# 5. Problems and Solutions

| Problem                | Investigation / Solution | Status             |
| ---------------------- | ------------------------ | ------------------ |
| WSL2 下串口设备识别           | 检查 Linux 串口设备            | Resolved           |
| Leader / Follower 舵机识别 | 1 Mbps 扫描 ID 1–6         | Resolved           |
| 机械臂校准                  | 生成并保存校准文件                | Resolved           |
| 主从机械臂遥操作               | 完成 Teleoperation 测试      | Resolved           |
| Windows Python 环境混乱    | 检查 Conda 环境和 Python 路径   | Resolved           |
| Windows 串口识别           | 识别 CH343 / COM10         | Resolved           |
| 校准文件迁移                 | 复制原有 JSON 文件             | Completed          |
| Windows 双摄像头帧率下降       | 检查设备、分辨率及 USB 资源         | Investigating      |
| Jetson 摄像头帧率不稳定        | 调整分辨率并进行单路测试             | Partially Resolved |
| Jetson 双路采集稳定性         | 仍需进一步测试                  | Investigating      |
| Dataset 录制流程           | 开始测试 LeRobot Record 功能   | In Progress        |

---

# 6. Current Progress

截至 2026 年 11 月：

### Completed

* [x] WSL2 Ubuntu 环境搭建
* [x] LeRobot 0.6.1 部署
* [x] Leader / Follower 舵机识别
* [x] 双机械臂校准
* [x] 主从机械臂遥操作
* [x] Windows 原生 Python 环境搭建
* [x] Windows 串口识别
* [x] 校准文件迁移
* [x] 双摄像头设备识别与单路测试
* [x] 双摄像头并发异常排查
* [x] GitHub Issue 提交
* [x] Jetson Nano Docker 环境部署
* [x] Jetson Nano LeRobot 0.6.2 运行
* [x] Jetson 摄像头单路采集测试

### In Progress

* [ ] Jetson 双摄像头稳定采集
* [ ] 完整 Dataset 录制流程
* [ ] 多 Episode 数据采集
* [ ] Dataset 完整性检查
* [ ] 后续模型训练与验证

---

# 7. Lessons Learned

这次部署过程涉及的不仅是软件安装，还包括硬件、操作系统、通信接口和视觉采集之间的协同。

几个比较重要的经验：

1. **环境迁移不等于简单复制配置。**

   Linux、Windows 和 Jetson 在串口、USB 设备访问、依赖管理等方面存在差异，需要分别验证。

2. **机械臂控制与视觉采集需要分开验证。**

   即使 Teleoperation 已经正常运行，也不代表多摄像头采集和 Dataset 录制一定能够正常工作。

3. **FPS 不完全代表视频质量。**

   摄像头即使能够达到接近目标的帧率，也可能出现卡顿、帧间隔不稳定等问题。

4. **边缘设备部署需要关注实际运行表现。**

   Jetson Nano 能够运行 LeRobot，并不意味着所有视觉采集任务都能够稳定执行。

---

# 8. Future Work

后续计划：

* 完善 Jetson Nano 双摄像头采集。
* 完成 SO-ARM101 数据集录制。
* 检查视频与机器人状态数据的一致性。
* 建立稳定的数据采集流程。
* 探索 LeRobot 模型训练及实际部署。

---

## References

### Hardware & Setup
- [Seeed Studio - SO-ARM101 Documentation](https://wiki.seeedstudio.com/cn/lerobot_so100m_new/)
- [MakerGuides - SO-ARM101 on Windows 11](https://www.makerguides.com/getting-started-so-arm101-win11/)

### LeRobot
- [LeRobot Official Repository](https://github.com/huggingface/lerobot)
- [LeRobot v0.6.1 Source Code](https://github.com/huggingface/lerobot/tree/v0.6.1)
- [LeRobot v0.6.0 Installation](https://huggingface.co/docs/lerobot/v0.6.0/installation)
- [LeRobot v0.6.0 LeLab Documentation](https://huggingface.co/docs/lerobot/v0.6.0/en/lelab)

### Troubleshooting
- [Conda Launchers Issue #35](https://github.com/conda/conda-launchers/issues/35)
- [LeRobot GitHub Issues](https://github.com/huggingface/lerobot/issues)

### Jetson Deployment
- [ravediamond/lerobot-jetson](https://github.com/ravediamond/lerobot-jetson)
- [LeRobot Issue #819 - Jetson Support](https://github.com/huggingface/lerobot/issues/819)


**Project Status: Active Development**

记录日期：2026-11-05

