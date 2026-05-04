# 系统架构设计 — 耆跑守护者

> **用途**：描述系统整体架构、模块划分、任务调度和数据流。
> **维护**：每次新增模块或修改架构后更新。

---

## 一、系统架构总览

```
┌─────────────────────────────────────────────────────────────┐
│                    应用层 (Application)                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ 跌倒检测  │  │ 语音交互  │  │ 状态指示  │  │ 呼救上报  │   │
│  │  状态机   │  │  模块    │  │  LED控制  │  │  模块    │   │
│  └─────┬────┘  └─────┬────┘  └─────┬────┘  └─────┬────┘   │
└────────┼──────────────┼──────────────┼──────────────┼───────┘
         │              │              │              │
┌────────┼──────────────┼──────────────┼──────────────┼───────┐
│        ▼              ▼              ▼              ▼       │
│                    驱动层 (Drivers)                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ BMI270   │  │ BME690   │  │ WS2812   │  │ WiFi/BLE │   │
│  │ 驱动     │  │ 驱动     │  │ RMT驱动  │  │ 驱动     │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
│  ┌──────────┐  ┌──────────┐                                 │
│  │ I2C总线  │  │ 音频驱动  │                                 │
│  │ 管理     │  │ (I2S/PDM)│                                 │
│  └──────────┘  └──────────┘                                 │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────┼───────────────────────────────┐
│                             ▼                               │
│                    硬件层 (Hardware)                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ BMI270   │  │ BME690   │  │ WS2812B  │  │ ESP32-C5 │   │
│  │ (Shuttle)│  │ (Shuttle)│  │ 灯带     │  │ 无线外设  │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、FreeRTOS 任务设计

### 2.1 任务总览

| 任务名称 | 优先级 | 栈大小 | 周期 | 功能 |
|---------|--------|--------|------|------|
| `sensor_read_task` | 5 (高) | 4096 | 10ms (100Hz) | 定时读取 BMI270 加速度/陀螺仪数据 |
| `fall_detect_task` | 4 (中高) | 4096 | 10ms (100Hz) | 跌倒检测算法处理 |
| `voice_task` | 3 (中) | 4096 | 事件触发 | 语音录制/播放/倒计时 |
| `led_task` | 2 (中低) | 2048 | 50ms (20Hz) | WS2812B 灯带状态更新 |
| `sos_report_task` | 1 (低) | 4096 | 事件触发 | WiFi 连接与呼救上报 |
| `env_sensor_task` | 1 (低) | 2048 | 1000ms (1Hz) | BME690 环境数据读取 |

### 2.2 任务间通信

```
sensor_read_task ──[Queue: accel_data]──→ fall_detect_task
                     [Queue: gyro_data]        │
                                               │
                                          [Event Group: fall_event]
                                               │
                    ┌──────────────────────────┼──────────────────┐
                    ▼                          ▼                  ▼
              voice_task                 led_task          sos_report_task
         (播放语音确认)              (更新灯带状态)       (发送呼救消息)
```

#### 通信方式

| 方式 | 用途 | 数据结构 |
|------|------|---------|
| **Queue (队列)** | 传感器数据传递 | `sensor_data_t` |
| **Event Group (事件组)** | 跌倒事件通知 | `fall_event_bits_t` |
| **Semaphore (信号量)** | 资源互斥（I2C 总线） | 二值信号量 |

### 2.3 数据流

```
BMI270 (100Hz)
    │
    ▼
sensor_read_task
    │ 读取原始数据 (ax, ay, az, gx, gy, gz)
    │ 滑动窗口滤波 (窗口大小=50)
    │ 计算 SVM = sqrt(ax² + ay² + az²)
    │
    ├──→ [Queue] → fall_detect_task
    │               │ SVM > 3g? → 疑似跌倒
    │               │ 姿态角突变 > 60°? → 确认跌倒
    │               │ 设置 Event Group 位
    │
    ├──→ [Queue] → env_sensor_task (1Hz)
    │               │ 读取温湿气压
    │               │ 可选：空气质量 IAQ
    │
    ▼
fall_detect_task
    │ FALL_EVENT_SUSPECTED  → led_task (黄色呼吸灯)
    │ FALL_EVENT_CONFIRMED  → voice_task (播放确认语音)
    │                       → led_task (红色快闪)
    │                       → 启动 30s 倒计时
    │ FALL_EVENT_CANCELLED  → led_task (蓝色闪烁3次)
    │                       → 恢复正常
    │ FALL_EVENT_TIMEOUT    → sos_report_task (发送呼救)
    │                       → led_task (红色常亮)
```

---

## 三、模块接口定义

### 3.1 BMI270 传感器模块

```c
// sensor_bmi270.h

typedef struct {
    float accel_x;      // 加速度 X 轴 (g)
    float accel_y;      // 加速度 Y 轴 (g)
    float accel_z;      // 加速度 Z 轴 (g)
    float gyro_x;       // 陀螺仪 X 轴 (°/s)
    float gyro_y;       // 陀螺仪 Y 轴 (°/s)
    float gyro_z;       // 陀螺仪 Z 轴 (°/s)
    float temperature;  // 芯片温度 (°C)
    uint32_t timestamp; // 时间戳 (us)
} bmi270_data_t;

esp_err_t bmi270_init(i2c_bus_handle_t i2c_bus);
esp_err_t bmi270_read_data(bmi270_data_t *data);
esp_err_t bmi270_get_fifo_data(bmi270_data_t *buffer, size_t *count);
```

### 3.2 跌倒检测模块

```c
// fall_detection.h

typedef enum {
    FALL_EVENT_NONE = 0,
    FALL_EVENT_SUSPECTED,       // 疑似跌倒
    FALL_EVENT_CONFIRMED,       // 确认跌倒
    FALL_EVENT_CANCELLED,       // 用户取消
    FALL_EVENT_TIMEOUT,         // 倒计时结束
} fall_event_type_t;

typedef void (*fall_event_callback_t)(fall_event_type_t event, void *user_data);

esp_err_t fall_detection_init(fall_event_callback_t callback, void *user_data);
esp_err_t fall_detection_feed(bmi270_data_t *data);
void fall_detection_cancel(void);   // 用户取消呼救
```

### 3.3 呼救上报模块

```c
// sos_reporter.h

typedef struct {
    char device_id[32];
    uint32_t timestamp;
    float latitude;
    float longitude;
    uint8_t fall_confidence;
    float temperature;
    float humidity;
} sos_message_t;

esp_err_t sos_reporter_init(void);
esp_err_t sos_reporter_send(sos_message_t *msg);
esp_err_t sos_reporter_set_wifi_credentials(const char *ssid, const char *password);
```

---

## 四、状态机设计

### 4.1 系统主状态机

```
                    ┌──────────────┐
                    │   INIT       │
                    │   (初始化)    │
                    └──────┬───────┘
                           │ 初始化完成
                           ▼
                    ┌──────────────┐
              ┌────│   NORMAL     │◄──────────────┐
              │    │   (正常监测)   │               │
              │    └──────┬───────┘               │
              │           │ SVM > 3g              │ 用户取消
              │           ▼                       │
              │    ┌──────────────┐               │
              │    │  SUSPECTED   │               │
              │    │  (疑似跌倒)   │               │
              │    └──────┬───────┘               │
              │           │ 姿态角验证通过          │
              │           ▼                       │
              │    ┌──────────────┐               │
              │    │  CONFIRMED   │               │
              │    │  (确认跌倒)   │               │
              │    └──────┬───────┘               │
              │           │ 启动 30s 倒计时        │
              │           ▼                       │
              │    ┌──────────────┐               │
              │    │  COUNTDOWN   │               │
              │    │  (倒计时中)   │───────────────┘
              │    └──────┬───────┘
              │           │ 倒计时结束
              │           ▼
              │    ┌──────────────┐
              │    │   SOS_SENT   │
              │    │  (已发送呼救)  │
              │    └──────────────┘
              │
              └──── 系统重置后回到 NORMAL
```

### 4.2 跌倒检测状态机

```
                    ┌─────────────────┐
                    │   IDLE          │
                    │  等待数据        │
                    └────────┬────────┘
                             │ 收到新数据
                             ▼
                    ┌─────────────────┐
                    │  CALC_SVM       │
                    │  计算 SVM 幅值   │
                    └────────┬────────┘
                             │
                    ┌────────┴────────┐
                    │                 │
               SVM > 3g          SVM ≤ 3g
                    │                 │
                    ▼                 ▼
            ┌──────────────┐  ┌──────────────┐
            │ CHECK_POSTURE │  │ 回到 IDLE    │
            │ 检查姿态角     │  │              │
            └────────┬─────┘  └──────────────┘
                     │
            ┌────────┴────────┐
            │                 │
       姿态角突变 > 60°    姿态角正常
            │                 │
            ▼                 ▼
    ┌──────────────┐  ┌──────────────┐
    │  FALL_EVENT  │  │ 回到 IDLE    │
    │  触发回调     │  │              │
    └──────────────┘  └──────────────┘
```

---

## 五、错误处理策略

| 错误场景 | 处理方式 |
|---------|---------|
| BMI270 初始化失败 | 重试 3 次，每次间隔 100ms；失败后系统报错重启 |
| BMI270 读取失败 | 重试 3 次，间隔 10ms；连续失败 10 次触发系统警告 |
| I2C 总线超时 | 重置 I2C 总线，重新初始化 |
| WiFi 连接失败 | 重试 5 次，每次间隔 2s；失败后使用 BLE 备选方案 |
| 呼救发送失败 | 本地存储消息，每 30s 重试一次，最多 10 次 |

---

## 六、电源管理

| 状态 | 说明 | 功耗优化措施 |
|------|------|-------------|
| **正常监测** | 系统全速运行 | BMI270 100Hz 采样 |
| **疑似跌倒** | 提高处理频率 | 启用陀螺仪，姿态角计算 |
| **倒计时中** | 等待用户确认 | 降低传感器采样率至 50Hz |
| **待机** | 未来扩展 | 待实现：深度睡眠，RTC 唤醒 |

---

## 七、模块依赖关系

```
main.c
  ├── sensor_bmi270    (依赖: I2C 驱动)
  ├── sensor_bme690    (依赖: I2C 驱动)
  ├── fall_detection   (依赖: sensor_bmi270)
  ├── voice_interaction (依赖: I2S/PDM 驱动)
  ├── led_indicator    (依赖: RMT 驱动)
  └── sos_reporter     (依赖: WiFi/BLE 驱动)
```

> **注意**：各模块之间通过事件组和队列解耦，不直接调用对方函数。
