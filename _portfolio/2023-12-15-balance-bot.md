---
title: "两轮自平衡机器人 (Two-Wheeled Self-Balancing Robot)"
excerpt: "基于 STM32 与 PID 控制算法的倒立摆系统，实现了直立环与速度环的串级控制。<br/><img src='/images/balance-cover.jpg' width='400'>"
collection: portfolio
date: 2024-12-15
header:
  teaser: /images/balance-cover.jpg
---

## 🛠 项目概述 (Project Overview)

这是一个基于**倒立摆原理**设计的两轮自平衡小车。项目核心在于利用 IMU（惯性测量单元）实时解算姿态角，并通过 **PID 闭环控制算法** 驱动直流电机，使系统保持动态平衡。

* **核心难点**：多环 PID 控制系统的参数整定（直立环、速度环、转向环）。
* **传感器融合**：使用 MPU6050 六轴传感器，结合互补滤波算法融合加速度计与陀螺仪数据。

---

## 💻 技术栈 (Tech Stack)

| 模块 | 详细参数 |
| :--- | :--- |
| **主控芯片** | STM32F103C8T6 (Cortex-M3) |
| **传感器** | MPU6050 (加速度计 + 陀螺仪) |
| **电机驱动** | TB6612FNG / L298N |
| **动力系统** | N20 减速电机 (带霍尔编码器) |
| **控制算法** | 串级 PID (直立 PD + 速度 PI) |
| **姿态解算** | 互补滤波 (Complementary Filter) |

---

## ⚙️ 核心算法实现 (Algorithm)

### 1. 姿态解算 (互补滤波)

由于加速度计动态响应慢但长期稳定，陀螺仪动态响应快但有积分漂移。我采用了**互补滤波**来融合两者数据，得到精准的 Pitch（俯仰）角。

```c
// 互补滤波核心代码片段
float angle = 0.0;
void Get_Angle(void) {
    // alpha 为互补系数，通常取 0.98
    // Gyro_Y 为陀螺仪积分角度，Accel_Angle 为加速度计计算角度
    angle = alpha * (angle + Gyro_Y * dt) + (1 - alpha) * Accel_Angle;
}
```

### 2. 串级 PID 控制器

为了让小车既能直立又能受控移动，我设计了双环控制系统：

* **直立环 (PD)**：极性保护，提供高频响应的恢复力矩，让小车“站起来”。(P: 回复力, D: 阻尼力)
* **速度环 (PI)**：作为外环，通过编码器积分消除静差，让小车保持静止或匀速运动。

---

## 📝 调试过程与踩坑 (Debugging Log)

**[2024-11-20] 高频抖动问题**
* **现象**：小车能站立，但电机发出滋滋的电流声，且伴随剧烈的高频抖动。
* **原因**：直立环的 D (微分项) 给得太大了，导致系统对噪声过度敏感。
* **解决**：减小 Kd 参数，并对陀螺仪数据进行软件低通滤波。

**[2024-11-25] "往一边跑" (零点漂移)**
* **现象**：小车平衡时总是慢慢往一个方向加速，最后倒下。
* **原因**：机械零点（重心垂直点）没有校准，MPU6050 安装有微小倾斜。
* **解决**：在静止平衡状态下记录 MPU6050 的原始偏移值，在代码中减去该 Offset。

---

## 🎥 实物运行演示 (Live Demo)

这里展示了从参数整定到最终稳定的两个阶段。

**1. PID 调试阶段 (Tuning Phase)**
视频说明：此时正在整定速度环参数。可以看到小车虽然能勉强直立，但无法静止，会不由自主地向前“漂移”移动，说明速度环积分项（I）还不够强。

<div style="margin: 15px 0;">
    <video width="100%" controls style="display: block; width: 100%;">
        <source src="/images/balance-tuning.mp4" type="video/mp4">
    </video>
</div>

**2. 最终效果：抗干扰测试 (Robustness Test)**
视频说明：参数整定完成。小车不仅能稳稳静止，当用手给予外部推力（干扰）时，能够迅速产生反向力矩回正，表现出良好的鲁棒性。

<div style="margin: 15px 0;">
    <video width="100%" controls style="display: block; width: 100%;">
        <source src="/images/balance-stable.mp4" type="video/mp4">
    </video>
</div>



[Image of PID control loop diagram]


<i class="fab fa-github"></i> [查看该项目完整代码 (GitHub)](your-link-here)
