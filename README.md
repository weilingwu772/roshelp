# ROS2 硬體開發指南 (roshelp)

歡迎來到 **ROS2 硬體開發指南** 儲存庫。本專案旨在提供將物理硬體組件與 ROS2 (Robot Operating System 2) 生態系統整合的全面知識庫。涵蓋範圍從高階架構框架到低階通訊協定。

---

## 🛠️ 硬體介面 (HWI) 文件導覽

探索我們的詳細指南，了解如何橋接軟體演算法與物理執行器/感測器：

### 1. [硬體整合框架 (Framework)](file:///d:/roshelp/ROS_Hardware_Integration_Framework.md)
*學習「大腦與軀幹」的比喻，以及使機器人系統各模組化且具備擴充性的三層架構（硬體層、通訊層與 ROS 驅動層）。*

### 2. [底層硬體通訊 (Communication)](file:///d:/roshelp/ROS2_Hardware_Communication.md)
*深入研究機器人中常用的通訊協定，包括：*
- **UART / Serial** (Arduino, GPS, 光達)
- **I2C** (IMU 感測器)
- **SPI** (高速編碼器)
- **CAN Bus** (工業級馬達)
- **Ethernet (UDP/TCP)** (3D 光達, 工業相機)

### 3. [ROS2 入門與核心概念](file:///d:/roshelp/ROS2_Introduction.md)
*專為 ROS2 新手準備的基礎指南，涵蓋節點 (Node)、主題 (Topic)、服務 (Service)、動作 (Action) 及 DDS 中介軟體。*

---

## 🚀 核心特點

- **模組化設計**：學習如何抽象化硬體，即使更換馬達，高階控制程式碼也無需修改。
- **分散式開發**：利用 TCP/IP 特性，在樹莓派上執行驅動程式，同時在效能強大的 PC 上執行重型 SLAM 演算法。
- **實戰範例**：提供 Serial 馬達驅動器等實用的 Python 程式碼片段。

## 📂 儲存庫結構

- `ROS_Hardware_Integration_Framework.md`：高階系統架構說明。
- `ROS2_Hardware_Communication.md`：詳細的通訊協定實作指南。
- `ROS2_Introduction.md`：ROS2 基礎概念與 CLI 指令教學。

---
*最後更新日期：2026-04-13*

