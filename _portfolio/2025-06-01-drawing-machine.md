---
title: "二维云台绘图系统 (Two-Axis Drawing Gimbal)"
excerpt: "基于 TI MSPG3507 与 Fusion360 建模的软硬结合项目，实现了复杂图形的高精度绘制。<br/><img src='/images/gimbal-cover.jpg' width='400'>"
collection: portfolio
date: 2025-09-01
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

![Fusion360建模图](/images/gimbal-cad.jpg)
*图1：Fusion 360 结构设计视图*

---

## 🧠 软件与算法实现 (Software & Algorithm)

### 1. 步进电机 S 型加减速与 k 级调速
为了防止电机在启动和停止时产生抖动（丢步），我没有使用简单的梯形加减速，而是编写了 **S 型曲线（Sigmoid）** 的速度控制算法，并实现了 **k 级调速** 以适应不同绘图速度的需求。

```c
// S型加减速与k级调速核心逻辑 (伪代码)
void Step_Motor_S_Curve(int steps, int speed_k) {
    for(int i=0; i<steps; i++) {
        // 根据k级速度因子动态调整周期
        // speed_k 越大，周期越短，速度越快
        int current_period = sigmoid_lookup[i] / speed_k; 
        
        // 动态调整定时器重装载值，改变脉冲频率
        // 适配 TI MSPG3507 寄存器操作
        DL_TimerG_setLoadValue(TIM2, current_period); 
    }
}
