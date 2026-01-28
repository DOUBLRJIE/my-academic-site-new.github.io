---
title: "二维云台绘图系统 (Two-Axis Drawing Gimbal)"
excerpt: "基于 TI MSPG3507 与 Fusion360 建模的软硬结合项目，实现了复杂图形的高精度绘制。<br/><img src='/images/gimbal-cover.jpg' width='400'>"
collection: portfolio
date: 2025-06-01
header:
  teaser: /images/gimbal-cover.jpg
---
## 🛠 项目概述 (Project Overview)

这是一个典型的**机械电控一体化**项目。我独立设计并制作了一个二自由度（2-DOF）云台，能够控制激光笔或画笔在墙面上绘制预设的图形（如正弦波、圆形、爱心）。

* **核心功能**：步进电机精准控制、复杂轨迹插补算法、G代码解析。
* **精度指标**：定位精度达到 ±0.5°。
* **开发周期**：2025.06 - 2025.09

---

## 💻 技术栈 (Tech Stack)

| 领域 | 核心技术 |
| :--- | :--- |
| **主控芯片** | TI MSPG3507 (Cortex-M0+) |
| **开发环境** | Keil MDK5, VS Code |
| **机械设计** | Fusion 360 (3D建模与仿真) |
| **控制算法** | Bresenham 直线插补, 圆弧插补, k级调速 |
| **硬件驱动** | A4988 步进电机驱动, PWM 脉冲生成 |

---

## ⚙️ 硬件与机械设计 (Hardware Design)

### 3D建模与结构设计
我使用 **Fusion 360** 完成了所有零部件的设计。为了保证转动的平滑性，我在关节处设计了轴承槽，并考虑到 3D 打印的公差（0.2mm）。

<div style="margin: 20px 0;">
  <img src="/images/gimbal-cad.jpg" alt="Fusion360建模图" style="width: 100%; border-radius: 5px;">
  <br>
  <em>图1：Fusion 360 结构设计视图</em>
</div>

---

## 🧠 软件与算法实现 (Software & Algorithm)

### 1. 步进电机 S 型加减速与 k 级调速
为了防止电机在启动和停止时产生抖动（丢步），我没有使用简单的梯形加减速，而是编写了 **S 型曲线（Sigmoid）** 的速度控制算法，并实现了 **k 级调速** 以适应不同绘图速度的需求。

```c
// S型加减速与k级调速核心逻辑 (伪代码)
void Step_Motor_S_Curve(int steps, int speed_k) {
    for(int i=0; i<steps; i++) {
        // 根据k级速度因子动态调整周期
        int current_period = sigmoid_lookup[i] / speed_k; 
        
        // 动态调整定时器重装载值
        DL_TimerG_setLoadValue(TIM2, current_period); 
    }
}

```

### 2. 轨迹插补算法

如何在二维平面上画出一个完美的圆？我实现了 **Bresenham 算法** 的变种，将数学坐标实时转换为两个电机的脉冲差，从而实现多轴联动。

---

## 📝 制作过程与踩坑记录 (Development Log)

### [2025-07-27] 严重的“丢步”问题

**现象**：画圆的时候，起点和终点重合不了，总是差 5mm 左右。
**排查**：

1. 以为是插补算法问题，检查了一周代码，没问题。
2. 用示波器看 PWM 波形，发现波形很完美。
3. **最终原因**：机械结构过重，电机的力矩不够！
**解决**：将云台悬臂的填充率从 100% 降到 20%（3D打印设置），并在电机轴上增加了减速齿轮组。

### [2025-08-10] 串口数据包粘包

**现象**：上位机发送复杂的 G 代码时，单片机经常解析错误。
**解决**：设计了一个环形缓冲区（Ring Buffer），并定义了帧头帧尾协议 `0xAA ... 0x55`，彻底解决了数据解析问题。

---
## 🎥 实物运行演示 (Live Demo)

这里展示系统在实际运行中的表现。视频未加速，可以清晰看到步进电机在绘制不同曲率线条时的加减速控制效果。

### 1. 绘制正弦波 (Sine Wave)
*难点：验证 Bresenham 算法在连续曲线上的插补平滑度。*

<div style="margin: 15px 0;">
  <video width="100%" controls style="display: block; width: 100%; transform: rotate(180deg);">
    <source src="/images/demo-sine.mp4" type="video/mp4">
    您的浏览器不支持 Video 标签，请升级浏览器。
  </video>
</div>

### 2. 绘制爱心 (Heart Shape)
*难点：对称图形的坐标变换与尖角处的路径规划。*

<div style="margin: 15px 0;">
  <video width="100%" controls style="display: block; width: 100%; transform: rotate(180deg);">
    <source src="/images/demo-heart.mp4" type="video/mp4">
    您的浏览器不支持 Video 标签，请升级浏览器。
  </video>
</div>

### 3. 绘制圆 (Pentagram)
*难点：直线段的精确衔接与电机急停急转的惯性抑制。*

<div style="margin: 15px 0;">
  <video width="100%" controls style="display: block; width: 100%; transform: rotate(180deg);">
    <source src="/images/demo-round.mp4" type="video/mp4">
    您的浏览器不支持 Video 标签，请升级浏览器。
  </video>
</div>
---
