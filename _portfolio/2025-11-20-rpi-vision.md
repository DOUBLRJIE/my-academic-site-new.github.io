---
title: "树莓派视觉一体化监控系统 (Raspberry Pi Integrated Vision System)"
excerpt: "基于 VS Code Remote 的远程开发实践，分别使用 Python (Gradio) 与 C++ (OpenCV) 实现的高性能端侧视觉应用。<br/><img src='/images/rpi-cover.jpg' width='400'>"
collection: portfolio
date: 2025-11-20
header:
  teaser: /images/rpi-cover.jpg
---

## 🛠 项目概述 (Project Overview)

这是一个**软硬结合**的计算机视觉项目，旨在树莓派 4B 上构建一个低延迟、可交互的视觉监控系统。项目最大的亮点在于**“双模态实现”**：我分别使用了开发效率极高的 **Python (Gradio)** 和运行效率极致的 **C++ (OpenCV + HTTP)** 两种技术栈完成了同一套系统，并进行了性能对比。

* **核心功能**：实时视频流传输、ROI (感兴趣区域) 动态框选、主色调提取 (K-Means)、颜色报警算法。
* **开发模式**：采用 VS Code Remote-SSH 进行全流程远程开发，模拟真实的工业级嵌入式开发场景。
* **教程产出**：编写了 3 份共计 40+ 页的技术文档（VSCode配置、Python实战、C++实战）。

---

## 💻 技术栈 (Tech Stack)

| 维度 | Python 版本 (快速原型) | C++ 版本 (高性能部署) |
| :--- | :--- | :--- |
| **语言标准** | Python 3.11 | C++ 17 |
| **视觉库** | OpenCV-Python (cv2) | OpenCV 4.6 (C++) |
| **Web 框架** | **Gradio** (现代化 UI) | **cpp-httplib** (轻量级 Server) |
| **依赖管理** | **uv** (极速包管理器) | **CMake** (编译构建) |
| **数据交互** | JSON API | JSON (nlohmann/json) |

---

## ⚙️ 模块一：现代化远程开发环境 (Dev Environment)

为了解决传统嵌入式开发“写代码在电脑，跑代码在板子”的割裂感，我搭建了基于 **VS Code Remote-SSH** 的无缝开发环境。

### 关键配置

通过配置 SSH 免密登录与 `~/.ssh/config`，实现了像开发本地项目一样开发树莓派。

```bash
# SSH Config 配置示例
Host rpi
    HostName 192.168.1.180
    User pi
    IdentityFile ~/.ssh/id_ed25519
```

<div style="margin: 20px 0;">
    <img src="/images/rpi-vscode.jpg" alt="VS Code Remote 开发环境" style="width: 100%; border-radius: 5px;">
    <br>
    <em>图1：VS Code 远程连接树莓派进行代码调试</em>
</div>

---

## 🐍 模块二：Python 版本实现 (Gradio 低代码)

在 Python 版本中，我使用了 Gradio 框架，仅用 300 行代码就实现了一个包含实时画面、控制面板、参数调节的现代化 Web UI。

### 核心算法：基于 HSV 的颜色报警

利用 OpenCV 将 BGR 转换到 HSV 空间，并处理红色在 Hue 环上的“跨界”问题（0°和179°）。

```python
# 核心逻辑：构建 HSV 掩膜 (Python)
def build_mask_from_hsv_ranges(bgr, hsv_ranges):
    hsv = cv2.cvtColor(bgr, cv2.COLOR_BGR2HSV)
    mask = None
    for r in hsv_ranges:
        lower = np.array(r["lower"], dtype=np.uint8)
        upper = np.array(r["upper"], dtype=np.uint8)
        m = cv2.inRange(hsv, lower, upper)
        # 逻辑或操作，合并多个颜色区间
        mask = m if mask is None else cv2.bitwise_or(mask, m)
    return cv2.medianBlur(mask, 5) # 中值滤波去噪
```

<div style="margin: 20px 0;">
    <img src="/images/rpi-gradio.jpg" alt="Python Gradio 运行界面" style="width: 100%; border-radius: 5px;">
    <br>
    <em>图2：基于 Gradio 的交互式控制台</em>
</div>

---

## ⚡ 模块三：C++ 版本实现 (高性能重构)

为了追求极致的性能（FPS），我使用 C++ 17 重构了整个系统。移除了 Python 解释器的开销，直接操作内存。

### 难点攻克：MJPEG 流媒体服务器

在 C++ 中没有 Gradio 这样现成的库，我使用 cpp-httplib 手写了一个多线程 HTTP 服务器，实现了 MJPEG 视频流的推送。

```cpp
// MJPEG 推流核心代码 (C++)
svr.Get("/stream.mjpg", [](const httplib::Request&, httplib::Response& res) {
    res.set_content_provider(
        "multipart/x-mixed-replace; boundary=frame",
        [](size_t, httplib::DataSink& sink) {
            while (g_running.load()) {
                std::vector<uchar> jpg;
                {
                    std::lock_guard<std::mutex> lk(g_mtx);
                    jpg = g_latest_jpeg; // 获取最新帧（线程安全）
                }
                if (!jpg.empty()) {
                    // 手动构建 HTTP Multipart 协议头
                    std::ostringstream oss;
                    oss << "--frame\r\nContent-Type: image/jpeg\r\n"
                        << "Content-Length: " << jpg.size() << "\r\n\r\n";
                    std::string hdr = oss.str();
                    sink.write(hdr.data(), hdr.size());
                    sink.write(reinterpret_cast<const char*>(jpg.data()), jpg.size());
                    sink.write("\r\n", 2);
                }
                std::this_thread::sleep_for(std::chrono::milliseconds(50));
            }
            return true;
        }
    );
});
```

<div style="margin: 20px 0;">
    <img src="/images/rpi-cpp.jpg" alt="C++ Web 运行界面" style="width: 100%; border-radius: 5px;">
    <br>
    <em>图3：C++ 编写的轻量级 Web 控制台</em>
</div>

---

## 📝 总结与思考 (Conclusion)

* **性能对比**：在同等分辨率下，C++ 版本的 CPU 占用率比 Python 版本降低了约 40%，帧率更加稳定。
* **开发效率**：Python 版本开发仅耗时 2 天，而 C++ 版本涉及 CMake 配置和手动内存管理，耗时约 5 天。
* **工程价值**：该项目展示了从快速原型验证 (POC) 到高性能产品交付的完整工程链路。

### 📂 原创教程文档下载 (PDF)
这是我编写的完整开发文档，共计 40+ 页，包含详细的配置步骤和代码解析：

* [<i class="fas fa-file-pdf"></i> **VS Code 远程开发配置指南** (PDF)](/files/tutorial-vscode.pdf)
* [<i class="fas fa-file-pdf"></i> **Python (Gradio) 实战教程** (PDF)](/files/tutorial-python.pdf)
* [<i class="fas fa-file-pdf"></i> **C++ (OpenCV) 高性能实战教程** (PDF)](/files/tutorial-cpp.pdf)

[<i class="fab fa-github"></i> 查看完整项目源代码 (GitHub)](https://github.com/DOUBLRJIE)

