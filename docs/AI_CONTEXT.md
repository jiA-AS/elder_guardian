# AI_CONTEXT.md — 耆跑守护者 (elder_guardian) 项目上下文

> **用途**：每次与 AI 对话时，将此文件作为上下文提供给 AI，确保 AI 理解项目全貌。
> **维护**：每次新增模块、修改引脚、变更协议后，同步更新此文件。

---

## 一、项目概述

| 项目 | 内容 |
|------|------|
| **项目名称** | 耆跑守护者 (elder_guardian) |
| **项目目标** | 基于 ESP32-C5 的老年人跌倒检测与自动呼救系统 |
| **应用场景** | 老年人居家/社区活动时，实时监测跌倒事件，触发语音确认后自动呼救 |
| **竞赛背景** | 物联网竞赛项目 |

---

## 二、硬件平台

### 2.1 主控

| 参数 | 值 |
|------|-----|
| **主控模组** | ESP32-C5-WROOM-1-N16R8 |
| **芯片** | ESP32-C5 (RISC-V 单核) |
| **开发板** | ESP-SensairShuttle v1.0 (乐鑫官方套件) |
| **Flash** | 16 MB |
| **PSRAM** | 8 MB |
| **无线能力** | 2.4 & 5 GHz 双频 Wi-Fi 6, BLE 5, Zigbee, Thread |

### 2.2 传感器与外设

| 外设 | 型号 | 连接方式 | 地址/引脚 | 用途 |
|------|------|---------|-----------|------|
| **IMU** | BMI270 (ShuttleBoard) | I2C | 0x68 | 6 轴加速度+陀螺仪，跌倒检测核心 |
| **磁力计** | BMM350 (ShuttleBoard) | I2C | 与 BMI270 共用 | 磁场方向检测 |
| **环境传感器** | BME690 (ShuttleBoard) | I2C | 0x76 | 温度、湿度、气压、空气质量 |
| **麦克风** | 外接麦克风 | 排针接口 | 需查原理图 | 语音确认交互 |
| **扬声器** | 外接扬声器 | 排针接口 | 需查原理图 | 语音提示、报警 |
| **LCD 屏幕** | 1.83" ST7789P3 | 4-line SPI | PWR_CTRL: GPIO5 | 显示交互信息 |
| **RGB 灯带** | WS2812 (外接) | 3-pin 接口 | 需查原理图 | 状态指示 |

### 2.3 引脚分配总表

| GPIO | 功能 | 连接设备 | 备注 |
|------|------|---------|------|
| GPIO4 | 通用 IO | 外置引脚接口 | 可用于外接传感器/设备 |
| GPIO5 | LCD_PWR_CTRL | LCD 屏幕 | **默认不可用**作普通 IO，需上件 R14 |
| GPIO8 | RGB 信号 | WS2812 灯带接口 | 外接 RGB 灯带 |
| I2C_SDA | I2C 数据线 | BMI270 + BMM350 + BME690 | 通过 Shuttle Board 接口 |
| I2C_SCL | I2C 时钟线 | BMI270 + BMM350 + BME690 | 通过 Shuttle Board 接口 |

> **注意**：I2C 和 RGB 接口的具体 GPIO 编号需查阅 ESP-SensairShuttle 原理图后补充。

---

## 三、软件环境

### 3.1 开发环境

| 工具 | 版本/配置 |
|------|----------|
| **ESP-IDF** | v6.0.1 |
| **目标芯片** | esp32c5 |
| **RTOS** | FreeRTOS (ESP-IDF 内置) |
| **编程语言** | **C 语言（严格禁止 C++）** |
| **IDE** | VS Code + ESP-IDF 插件 |
| **烧录端口** | COM7 |
| **烧录方式** | UART |
| **OpenOCD 配置** | board/esp32c5-builtin.cfg |

### 3.2 关键约束

- ❌ **禁止使用 C++**（包括 `.cpp` 文件、类、模板、STL、`new`/`delete`）
- ✅ 允许使用 C 标准库 + ESP-IDF 提供的 C API
- ✅ 允许使用 FreeRTOS 的 C API（任务、队列、信号量等）
- ✅ 推荐使用 ESP-IDF 组件化结构（`components/` 目录）

---

## 四、系统模块划分

```
elder_guardian/
├── main/                          # 主应用入口
│   ├── main.c                     # app_main() 入口，初始化所有模块
│   └── CMakeLists.txt
├── components/                    # 功能组件（后续添加）
│   ├── sensor_bmi270/             # BMI270 驱动封装
│   ├── sensor_bme690/             # BME690 驱动封装
│   ├── fall_detection/            # 跌倒检测算法状态机
│   ├── voice_interaction/         # 语音录制/播放/识别
│   ├── led_indicator/             # WS2812B 灯带控制
│   └── sos_reporter/              # 呼救上报模块（WiFi/MQTT/BLE）
├── docs/                          # 项目文档
│   ├── AI_CONTEXT.md              # ← 本文件：AI 上下文
│   ├── hardware_pinout.md         # 硬件引脚分配表
│   ├── architecture.md            # 系统架构设计
│   └── algorithm.md               # 跌倒检测算法设计
├── CMakeLists.txt
└── README.md
```

### 4.1 模块职责

| 模块 | 职责 | 优先级 |
|------|------|--------|
| **sensor_bmi270** | BMI270 初始化、加速度/陀螺仪数据读取、FIFO 管理 | P0 |
| **sensor_bme690** | BME690 初始化、环境数据读取 | P1 |
| **fall_detection** | 跌倒检测状态机：SVM 计算 → 阈值判断 → 姿态验证 → 事件触发 | P0 |
| **voice_interaction** | PDM 麦克风录音、I2S 扬声器播放、语音确认倒计时 | P0 |
| **led_indicator** | WS2812B 灯带控制：正常/警告/报警/确认 状态指示 | P1 |
| **sos_reporter** | 跌倒事件上报：WiFi 连接 → HTTP/MQTT 发送 → BLE 备选 | P0 |

---

## 五、通信协议与接口

### 5.1 I2C 总线

| 参数 | 值 |
|------|-----|
| **总线** | 单 I2C 总线，主模式 |
| **速率** | 400kHz (Fast Mode) |
| **设备 1** | BMI270 (地址: 0x68) |
| **设备 2** | BME690 (地址: 0x76) |

### 5.2 跌倒检测事件接口

```c
// 跌倒检测回调函数原型
typedef void (*fall_event_callback_t)(fall_event_type_t event, void *user_data);

// 事件类型
typedef enum {
    FALL_EVENT_NONE = 0,
    FALL_EVENT_SUSPECTED,       // 疑似跌倒（SVM 超阈值）
    FALL_EVENT_CONFIRMED,       // 确认跌倒（姿态角验证通过）
    FALL_EVENT_CANCELLED,       // 取消（用户语音确认未跌倒）
    FALL_EVENT_TIMEOUT,         // 倒计时结束，自动呼救
} fall_event_type_t;
```

### 5.3 呼救上报接口

```c
// 呼救消息结构体
typedef struct {
    char device_id[32];         // 设备 ID
    uint32_t timestamp;         // 事件时间戳
    float latitude;             // 纬度（可选，如有 GPS）
    float longitude;            // 经度（可选，如有 GPS）
    uint8_t fall_confidence;    // 跌倒置信度 0-100
} sos_message_t;
```

---

## 六、编码规范

### 6.1 命名规则

| 类型 | 规则 | 示例 |
|------|------|------|
| **函数** | `snake_case`，模块前缀 | `bmi270_init()`, `fall_detect_task()` |
| **变量** | `snake_case` | `accel_data`, `sensor_ready` |
| **宏/常量** | `UPPER_SNAKE_CASE` | `FALL_THRESHOLD_G`, `BMI270_I2C_ADDR` |
| **类型定义** | `snake_case_t` | `fall_event_type_t`, `bmi270_config_t` |
| **枚举** | `UPPER_SNAKE_CASE` | `FALL_EVENT_CONFIRMED` |

### 6.2 注释要求

- 所有公共函数必须带 **doxygen 风格注释**
- 复杂算法逻辑必须写 **块注释说明原理**
- 硬件相关的魔数必须注释其来源

```c
/**
 * @brief 初始化 BMI270 传感器
 * @param i2c_bus I2C 总线句柄
 * @param config 配置参数指针
 * @return esp_err_t ESP_OK 成功，否则失败
 */
esp_err_t bmi270_init(i2c_bus_handle_t i2c_bus, bmi270_config_t *config);
```

### 6.3 错误处理模式

- 所有 ESP-IDF API 调用必须检查返回值
- 错误处理使用 `ESP_ERROR_CHECK()`（开发阶段）或 `esp_err_t` 传播（生产阶段）
- 传感器读取失败应有重试机制（3 次重试，间隔 10ms）

---

## 七、当前开发状态

### 已完成
- [x] 项目初始化（ESP-IDF v6.0.1, esp32c5）
- [x] 项目文档框架搭建

### 进行中
- [ ] BMI270 传感器驱动开发
- [ ] 跌倒检测状态机设计

### 待开始
- [ ] BME690 传感器驱动
- [ ] 语音交互模块
- [ ] WS2812B 灯带控制
- [ ] WiFi 连接与呼救上报
- [ ] 系统集成测试

---

## 八、常见问题与决策记录

| 日期 | 决策 | 原因 |
|------|------|------|
| - | 使用 C 语言而非 C++ | ESP-IDF 对 C++ 支持有限，且竞赛环境以 C 为主 |
| - | 跌倒检测使用 SVM + 姿态角双重验证 | 降低误报率，避免单阈值判断不可靠 |
| - | 语音确认倒计时 30s | 给老人足够时间回应，同时避免延误呼救 |

---

> **使用说明**：将此文件内容复制到 AI 对话的开头，或配置为 VS Code 智能体的项目上下文文件。
> **更新频率**：每次新增模块、修改引脚、变更协议后立即更新。
