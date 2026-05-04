# 耆跑守护者 (elder_guardian)

> 基于 ESP32-C5 的老年人跌倒检测与自动呼救系统

---

## 项目概述

本项目为物联网竞赛项目，基于 **ESP32-C5** (ESP-SensairShuttle 开发板) 和 **Bosch Sensortec** 传感器，实现老年人跌倒检测与自动呼救功能。

### 核心功能

- **实时跌倒检测**：利用 BMI270 IMU 的加速度和陀螺仪数据，通过 SVM + 姿态角双重验证算法
- **语音确认交互**：检测到跌倒后播放语音确认，30 秒倒计时后自动呼救
- **状态指示**：WS2812B 灯带显示系统状态（正常/警告/报警/呼救）
- **自动呼救**：通过 WiFi/BLE 发送呼救消息

---

## 硬件平台

| 组件 | 型号 | 说明 |
|------|------|------|
| **主控模组** | ESP32-C5-WROOM-1-N16R8 | 16MB Flash + 8MB PSRAM |
| **开发板** | ESP-SensairShuttle v1.0 | 乐鑫官方套件 |
| **IMU** | BMI270 (ShuttleBoard) | 6 轴加速度+陀螺仪 |
| **磁力计** | BMM350 (ShuttleBoard) | 3 轴磁力计 |
| **环境传感器** | BME690 (ShuttleBoard) | 温湿气压+空气质量 |
| **LCD 屏幕** | 1.83" ST7789P3 | 240×284 分辨率 |
| **灯带** | WS2812B | 外接 RGB 灯带 |

---

## 软件环境

- **ESP-IDF**: v6.0.1
- **目标芯片**: esp32c5
- **RTOS**: FreeRTOS
- **编程语言**: C（禁止 C++）

---

## 项目结构

```
elder_guardian/
├── main/                          # 主应用入口
│   ├── main.c                     # app_main() 入口
│   └── CMakeLists.txt
├── components/                    # 功能组件（待开发）
│   ├── sensor_bmi270/             # BMI270 驱动
│   ├── sensor_bme690/             # BME690 驱动
│   ├── fall_detection/            # 跌倒检测算法
│   ├── voice_interaction/         # 语音交互
│   ├── led_indicator/             # 灯带控制
│   └── sos_reporter/              # 呼救上报
├── docs/                          # 项目文档
│   ├── AI_CONTEXT.md              # AI 上下文（给 AI 的项目简介）
│   ├── hardware_pinout.md         # 硬件引脚分配表
│   ├── architecture.md            # 系统架构设计
│   └── algorithm.md               # 跌倒检测算法设计
├── CMakeLists.txt
└── README.md
```

---

## 快速开始

### 1. 环境搭建

确保已安装 ESP-IDF v6.0.1 并配置 esp32c5 目标：

```bash
# 设置目标芯片
idf.py set-target esp32c5

# 打开配置菜单
idf.py menuconfig
```

### 2. 编译与烧录

```bash
# 编译
idf.py build

# 烧录（端口 COM7）
idf.py -p COM7 flash

# 监视串口输出
idf.py -p COM7 monitor
```

### 3. 硬件连接

1. 将 ShuttleBoard-BMI270&BMM350 插入主板 Shuttle Board 接口
2. 将 ShuttleBoard-BME690 插入主板 Shuttle Board 接口
3. 连接 LCD 屏幕排线
4. 连接外接麦克风和扬声器
5. 连接 WS2812B 灯带至 RGB 接口
6. 通过 Type-C 连接电脑供电

---

## 开发状态

- [x] 项目初始化
- [x] 文档框架搭建
- [ ] BMI270 传感器驱动
- [ ] 跌倒检测算法
- [ ] BME690 传感器驱动
- [ ] 语音交互模块
- [ ] WS2812B 灯带控制
- [ ] WiFi 连接与呼救上报
- [ ] 系统集成测试

---

## 文档索引

| 文档 | 说明 |
|------|------|
| [AI_CONTEXT.md](docs/AI_CONTEXT.md) | 项目总上下文，给 AI 的项目简介 |
| [hardware_pinout.md](docs/hardware_pinout.md) | 硬件引脚分配和传感器参数 |
| [architecture.md](docs/architecture.md) | 系统架构、任务设计和数据流 |
| [algorithm.md](docs/algorithm.md) | 跌倒检测算法原理和实现 |

---

## 许可证

本项目仅供学习交流使用。
