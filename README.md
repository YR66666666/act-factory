# act-factory · ALOHA ACT 原厂代码（未修改）

> 本仓库保存 AgileX 出厂包 `cobot_magic.zip`（**2024-04-17**）里的 ACT 代码，**一行未改**。
> 用途：对照、回滚、看清楚"哪些是原厂的、哪些是我们加的"。

改后在真机上跑通、带调试后处理的版本在另一个仓库：
**[cobot-magic-act](https://github.com/YR66666666/cobot-magic-act)** ← 实际使用请用这个

---

## 一、和 GitHub 上那个 `agilexrobotics/act` 不是一回事

| | `agilexrobotics/act`（GitHub） | 本仓库（出厂包） |
|---|---|---|
| 定位 | ACT **仿真版**（MuJoCo 里训练和测试） | ACT **真机版**（ROS + 真实机械臂） |
| 入口 | `imitate_episodes.py` | `train.py`（训练）/ `inference.py`（推理） |
| 依赖 | Python 3.9 + torch 2.0 + mujoco | Python 3.8 + torch 1.10（为 ROS Noetic 的 cv_bridge） |
| 仿真环境 | 有 `sim_env.py` / `ee_sim_env.py` | 没有 |
| 额外策略 | 只有 ACT / CNNMLP | 多一条 **Diffusion** 分支 |
| 最后更新 | 2023-12 | 2024-04 |

两者**不能互换**：仿真版没有 ROS 推理，环境要求也和本机 ROS 冲突。

---

## 二、目录

```
act/                         ACT 算法（17 个文件）
├── inference.py             推理主程序（原厂版：无夹爪后处理）
├── train.py                 训练主程序
├── policy.py                ACTPolicy / CNNMLPPolicy / DiffusionPolicy
├── utils.py                 数据加载 / 归一化 / EpisodicDataset
├── train.sh                 原作者示例脚本（路径是别人机器的，别直接跑）
└── detr/                    模型本体
    ├── main.py
    └── models/              detr_vae.py（ACT 网络）/ backbone.py / transformer.py ...

collect_data.py              采集程序（原厂版：开录后只能录满，不能中途取消）
robomimic/                   精简版 robomimic（提供 ResNet18 backbone，训练必需）
```

### 关键文件（原厂实现要点）

| 文件 | 关键函数 | 作用 |
|------|----------|------|
| `act/train.py` | `train()` / `train_process()` | 读 HDF5 → 算归一化 `dataset_stats.pkl` → 训 3000 epoch → 每 100 epoch 存 ckpt |
| `act/inference.py` | `model_inference()` | ROS 30 Hz 闭环：取帧 → 模型 → 发布关节角 |
| `act/policy.py` | `ACTPolicy.serialize/deserialize` | 把网络封装成可存取的对象 |
| `act/utils.py` | `load_data()` / `EpisodicDataset` | 一个样本 = 一条 episode 的一段 chunk；8:2 按条切分 |
| `act/detr/models/detr_vae.py` | `build()` | ResNet18 编码图像 + Transformer 编解码 + CVAE |

---

## 三、⚠️ 直接跑会有问题（所以才有改后版）

| 原厂行为 | 后果 |
|----------|------|
| 模型输出的**连续夹爪值**直接下发 | 示教里搬运段是 `≈0.041`（方块撑住爪子的**测量值**，非指令值），照抄等于"贴住方块、零夹紧力"→ 一碰就滑；到盒子张不到方块宽度外 → **松不了手** |
| 没有释放兜底 | 模型偶尔停在夹住值 → 抱着方块飞回原位 / 半路掉落 / 在盒子上方停 20 秒 |
| 没有自动停止 | 跑满 `max_publish_step` 才算完，会重复执行任务 |
| `train.py` 没固定 `camera_names` | 只有 2 路相机时会去找不存在的 `cam_left_wrist` |

改后版做的三件事（详见 `cobot-magic-act` 仓库的 README 第 5 节）：
**夹爪两态化 + 到位释放 + 静止兜底**。

---

## 四、环境

| 组件 | 版本 |
|------|------|
| Python | 3.8（conda env `aloha`） |
| PyTorch | 1.10.0 + cu113 |
| ROS | Noetic |
| GPU | RTX 4060 8 GB |

```bash
# 仓库根目录（含 act/ 和 robomimic/）+ ROS 的 python3 路径
cd /path/to/act-factory
export PYTHONPATH=$PWD:$PWD/act:/opt/ros/noetic/lib/python3/dist-packages
```

---

## 五、许可

ACT 原始实现 © Tony Zhao 等（[tonyzhaozh/aloha](https://github.com/tonyzhaozh/aloha)）；
`robomimic/` © Stanford。本仓库仅为内部备份与对照用途。
