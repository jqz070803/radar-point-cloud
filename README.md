# 毫米波雷达体感跑酷（Radar Motion Game）

用 24 GHz 毫米波雷达非接触操作一个跑酷游戏：**雷达采集 → 人体动作识别 → 游戏事件 → 计分结算**。
不需要摄像头、不需要穿戴设备，也不依赖环境光照。

本项目为课程项目（企业课题）的可运行原型，重点不在"炫技"，而在**把一条真实硬件链路做到可复现、可证伪、可演示**。

---

## 1. 系统闭环

```
┌─────────────┐   raw bytes   ┌──────────────┐   RadarFrame   ┌───────────────┐
│ 雷达串口     │ ────────────► │ 解析 + 信号处理 │ ─────────────► │ 动作识别（规则） │
│ (CP2102/FTDI)│               │ 2D-FFT/CFAR/AoA│                │ 阈值+滞回+冷却   │
└─────────────┘               └──────────────┘                └───────┬───────┘
                                                                      │ GameEvent
                                                                      ▼
                                                              ┌───────────────┐
                                                              │  跑酷游戏      │
                                                              │ LANE_LEFT/RIGHT│
                                                              │ JUMP / DUCK    │
                                                              └───────────────┘
```

层与层之间**只有一个耦合点**：事件流。

```python
GameEvent = LANE_LEFT | LANE_RIGHT | JUMP | DUCK | CALIBRATED | TARGET_LOST | TARGET_BACK
```

游戏层不直接读串口，也不关心数据来自真机、回放还是模拟器。因此**一个热键即可在真机与模拟器之间切换**，现场演示不会因为设备问题中断。

交互范式是**动作触发**而不是绝对位置映射：身体移动一次 → 角色切换一条道。这是由 7.7 Hz 的帧率上限决定的——在 130 ms 的更新周期里做绝对位置跟随，手感一定糟。

---

## 2. 仓库结构

```
docs/                 设计、验证与决策记录
  proposal-motion-game.md        课程项目提案
  PRD-motion-game-MVP.md         MVP 需求
  TechDesign-motion-game-MVP.md  技术设计（分层与事件契约）
  hardware-baseline.md           硬件与能力基线（公开版）
  jump-duck-verification-plan.md 跳/蹲验证门（P0→T0→T1→T2）
  acceptance-test-table.md       验收测试表
  ai-decision-log.md             AI 使用决策记录
  handoff-*.md                   阶段性交接记录
  radar-control-and-actions.md   采集会话控制与动作定义
  bg-baseline-verification.md    环境底噪基线验证
  research-motion-game.md        前期调研

radar-pointcloud/     雷达侧工具链
  radar_protocol.py             1T2R 帧解析（ICLH/ICLT 包裹 + 40 B 雷达帧）
  radar_protocol_dev113p.py     2T4R 帧解析（136 B 帧 / 32 bin / 4 RX）
  radar_dsp.py                  2D-FFT → CFAR → 相位差测角 → 点云
  radar_actions.py              规则法动作识别（阈值 + 滞回 + 最小切换间隔）
  radar_source.py               数据源抽象：串口 / 回放 / 合成 / 回环
  server.py                     HTTP + SSE 服务，驱动网页点云
  viewer.html                   实时点云与调试面板
  *_capture.py / label_tool.py  带真值标注的采集与标注
  selftest.py                   离线自检（解析、闭环、角度、会话、动作）

bear-run/             游戏侧（Three.js 跑酷原型）
  js/InputSource.js             事件输入源：真机 / 键盘 / 模拟器热切换
  js/Game.js, Player.js, ...    游戏逻辑与渲染

outputs/              面向展示的产物
  eval-*.md                     识别效果评估
  lr2-*-诊断-逐窗.md            关键帧级诊断
  lr-move-*.csv                 左右移动派生特征数据集
  M3-演示流程卡.md              现场演示流程
```

---

## 3. 快速开始

### 3.1 无硬件（模拟器 / 回放）

```bash
pip install -r requirements.txt

# 雷达侧：回环源自检，不需要硬件
cd radar-pointcloud
python server.py --source loopback --open

# 游戏侧：直接打开 bear-run/index.html（或任意静态服务器）
```

页面连不上服务时会显示**演示动画**，并明确标注"不是你的雷达数据"，不会把动画伪装成采集结果。

### 3.2 真机

```bash
cd radar-pointcloud
python server.py --port auto --open     # 自动识别串口与板型
```

- 1T2R：CP2102，921600 8-N-1
- 2T4R：FTDI FT232，1500000 8-N-1
- 串口**独占**：采集时厂商上位机必须关闭，反之亦然

### 3.3 自检

```bash
cd radar-pointcloud
python selftest.py
```

自检覆盖：帧解析、合成闭环、方位角误差、会话控制、近距动作规则。报告写入 `selftest-report.txt`。

---

## 4. 关键指标

| 指标 | 1T2R（主控） | 2T4R（备选） |
|---|---|---|
| 雷达帧周期 | **129.5 ms ≈ 7.7 Hz** | ≈520.84 ms ≈ 1.73 Hz |
| 距离分辨率 | **≈0.681 m** | ≈0.150 m |
| 最大不模糊速度 | ≈±1.535 m/s | ≈±0.376 m/s |
| 速度分辨率 | ≈0.048 m/s | ≈0.01175 m/s |
| 接收通道 | 2（仅水平角） | 4（水平 + 俯仰候选） |
| 垂直角 | **不可用** | 通道存在，映射未定 |

**7.7 Hz 是全项目的体验上限**：任何交互设计都必须先过这一关。

---

## 5. 数据集与验证证据

仓库只保留**派生的特征表与统计报告**，不保留原始 `.bin` 采集：

- `outputs/lr-move-frames.csv`：左右移动逐帧特征（3092 帧，含站位、真值标签）
- `outputs/lr-move-actions.csv`：逐动作样本
- `outputs/eval-*.md`、`outputs/lr2-*-诊断-逐窗.md`：识别效果与逐窗诊断
- `docs/ai-decision-log.md`：每个关键决策的"问了什么 / 得到什么 / 为何采纳或否决"

原始采集文件体积大、且属于**不含图像的人体运动数据**，按课程保密要求不外传；需要时可用仓库内脚本重新采集。

---

## 6. 已知边界（诚实说明）

- **跳 / 蹲尚未定案**：已在 MVP 范围内，但实现路径（2T4R 俯仰角测高 / 1T2R 多普勒间接特征）须走完 P0→T0→T1→T2 验证门，**未验证前不作为已实现功能**。
- 2T4R 的距离刻度为**标称参数推导 + 交叉验证**，尚缺卷尺真值点检。
- 1T2R 存在 0.32~0.70 m 的**非恒定系统偏移**，检出率随距离下降，原因未解释。
- 串口自动识别只在单串口场景做过真机验证；多串口消歧未验证。
- 左右换道门限目前基于 1T2R 数据标定，换板需重新标定。

---

## 7. 保密与数据边界

本仓库**不包含**以下内容，且不会对外提供：

- 厂商固件、SDK、评估套件与上位机安装文件；
- 厂商配置文件、通信日志转储、寄存器表与命令序列；
- 授权文件（`.lic`）、产品手册与用户手册原文；
- 原始雷达采集 `.bin` 文件。

因此仓库中的硬件控制层（寄存器级停机、串口探测等）**有意未包含**；已包含的代码只覆盖"读取设备输出并做信号处理"这一层。详见 [NOTICE.md](NOTICE.md)。

本项目全程本地运行：不联网上传数据、不写生产环境、不涉及账号与支付。

---

## 8. 许可

代码以 [LICENSE](LICENSE) 授权。文档中的硬件型号与商标归其各自权利人所有，仅用于技术说明。
