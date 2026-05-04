# 跌倒检测算法设计 — 耆跑守护者

> **用途**：详细描述跌倒检测算法的原理、实现步骤和参数配置。
> **维护**：每次修改算法参数或流程后更新。

---

## 一、算法概述

### 1.1 检测原理

跌倒检测基于 **人体运动学特征**，利用 BMI270 的加速度计和陀螺仪数据，通过以下两个关键特征判断跌倒：

1. **SVM (Signal Vector Magnitude)**：加速度矢量的合成幅值，反映冲击强度
2. **姿态角变化**：俯仰角 (Pitch) 和横滚角 (Roll) 的突变，反映身体姿态变化

### 1.2 算法特点

| 特性 | 说明 |
|------|------|
| **双重验证** | SVM 阈值 + 姿态角变化，降低误报率 |
| **滑动窗口滤波** | 消除噪声干扰，提高检测稳定性 |
| **状态机驱动** | 清晰的状态转换，便于调试和扩展 |
| **低计算量** | 仅需浮点运算，无需 FFT 等复杂计算 |

---

## 二、算法流程

### 2.1 完整流程图

```
┌──────────────────────────────────────────────────────────────┐
│                    数据采集 (100Hz)                           │
│  读取 BMI270 加速度 (ax, ay, az) 和 陀螺仪 (gx, gy, gz)      │
└────────────────────────┬─────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────────┐
│                    滑动窗口滤波                                │
│  窗口大小: N=50 (对应 500ms @ 100Hz)                         │
│  滤波方式: 移动平均滤波 (Moving Average)                      │
└────────────────────────┬─────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────────┐
│                    计算 SVM                                    │
│  SVM = sqrt(ax² + ay² + az²)                                 │
│  注: ax, ay, az 为滤波后的值                                  │
└────────────────────────┬─────────────────────────────────────┘
                         │
                         ▼
                    ┌─────────┐
                    │ SVM >   │  NO
                    │ 3g?     │─────────→ 回到正常监测
                    └────┬────┘
                         │ YES
                         ▼
┌──────────────────────────────────────────────────────────────┐
│                    疑似跌倒 (SUSPECTED)                        │
│  触发: 黄色呼吸灯                                             │
│  记录当前时间戳 t_suspect                                      │
└────────────────────────┬─────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────────┐
│                    姿态角计算                                  │
│  利用陀螺仪积分计算俯仰角 (Pitch) 和横滚角 (Roll)              │
│                                                              │
│  pitch = pitch + gyro_x × dt                                 │
│  roll  = roll  + gyro_y × dt                                 │
│                                                              │
│  观察窗口: 5s (从 t_suspect 开始)                             │
└────────────────────────┬─────────────────────────────────────┘
                         │
                         ▼
                    ┌─────────┐
                    │ 姿态角  │  NO
                    │ 变化 >  │─────────→ 回到正常监测
                    │ 60°?   │           (误报，取消)
                    └────┬────┘
                         │ YES
                         ▼
┌──────────────────────────────────────────────────────────────┐
│                    确认跌倒 (CONFIRMED)                        │
│  触发: 红色快闪 LED + 语音确认 "检测到跌倒，请回应"            │
│  启动 30 秒倒计时                                              │
└────────────────────────┬─────────────────────────────────────┘
                         │
                         ▼
                    ┌─────────┐
                    │ 用户    │  YES
                    │ 取消?   │─────────→ 回到正常监测
                    └────┬────┘           (蓝色闪烁 3 次)
                         │ NO (30s 超时)
                         ▼
┌──────────────────────────────────────────────────────────────┐
│                    自动呼救 (SOS_SENT)                         │
│  触发: 红色常亮 LED                                           │
│  发送: WiFi/BLE 呼救消息                                      │
└──────────────────────────────────────────────────────────────┘
```

---

## 三、数学公式

### 3.1 SVM (Signal Vector Magnitude)

```
SVM(t) = sqrt(ax(t)² + ay(t)² + az(t)²)

其中:
  ax(t), ay(t), az(t) = t 时刻滤波后的三轴加速度值 (单位: g)
  SVM(t) = t 时刻的合成加速度幅值 (单位: g)
```

**阈值说明**：
- 静止/正常活动时：SVM ≈ 1g (重力加速度)
- 行走/慢跑时：SVM ≈ 1.5g ~ 3g
- 跌倒冲击时：SVM > 3g (典型值 5g ~ 10g)

### 3.2 姿态角计算

#### 俯仰角 (Pitch)

```
pitch(t) = pitch(t-1) + gyro_x(t) × Δt

其中:
  gyro_x(t) = t 时刻 X 轴角速度 (单位: °/s)
  Δt = 采样间隔 (单位: s, 100Hz 时 Δt = 0.01s)
```

#### 横滚角 (Roll)

```
roll(t) = roll(t-1) + gyro_y(t) × Δt

其中:
  gyro_y(t) = t 时刻 Y 轴角速度 (单位: °/s)
```

#### 姿态角变化量

```
Δangle = sqrt(Δpitch² + Δroll²)

其中:
  Δpitch = |pitch(t) - pitch(t_suspect)|
  Δroll  = |roll(t) - roll(t_suspect)|
  t_suspect = 触发疑似跌倒的时刻
```

### 3.3 滑动窗口滤波

```
ax_filtered(t) = (1/N) × Σ(ax(t - i)), i = 0 to N-1

其中:
  N = 窗口大小 (N=50)
  等效于 500ms 的时间窗口 @ 100Hz 采样率
```

---

## 四、参数配置

### 4.1 核心参数

| 参数 | 默认值 | 说明 | 调整建议 |
|------|--------|------|---------|
| `FALL_THRESHOLD_G` | 3.0g | SVM 阈值 | 灵敏度要求高则降低，误报多则提高 |
| `ANGLE_THRESHOLD_DEG` | 60° | 姿态角变化阈值 | 同上 |
| `WINDOW_SIZE` | 50 | 滑动窗口大小 (采样点数) | 噪声大则增大，响应快则减小 |
| `SAMPLE_RATE_HZ` | 100 | 传感器采样率 (Hz) | 100Hz 已足够 |
| `OBSERVE_WINDOW_S` | 5 | 姿态角观察窗口 (秒) | 跌倒过程通常 < 2s |
| `COUNTDOWN_TIMEOUT_S` | 30 | 语音确认倒计时 (秒) | 根据目标用户调整 |
| `ACCEL_RANGE` | ±8g | 加速度计量程 | 跌倒冲击可达 8g+ |
| `GYRO_RANGE` | ±1000°/s | 陀螺仪量程 | 跌倒旋转速度较快 |

### 4.2 参数调优指南

```
误报太多？ → 提高 FALL_THRESHOLD_G (3.5g ~ 4.0g)
           提高 ANGLE_THRESHOLD_DEG (70° ~ 80°)
           增大 WINDOW_SIZE (60 ~ 80)

漏报太多？ → 降低 FALL_THRESHOLD_G (2.5g ~ 2.8g)
           降低 ANGLE_THRESHOLD_DEG (45° ~ 50°)
           减小 WINDOW_SIZE (30 ~ 40)

响应太慢？ → 降低 WINDOW_SIZE
           降低 OBSERVE_WINDOW_S
```

---

## 五、代码实现框架

### 5.1 数据结构

```c
// fall_detection.h

#define FALL_THRESHOLD_G        3.0f
#define ANGLE_THRESHOLD_DEG     60.0f
#define WINDOW_SIZE             50
#define SAMPLE_RATE_HZ          100
#define OBSERVE_WINDOW_S        5
#define COUNTDOWN_TIMEOUT_S     30

typedef struct {
    float svm;              // 当前 SVM 值
    float pitch;            // 当前俯仰角
    float roll;             // 当前横滚角
    float delta_angle;      // 姿态角变化量
    uint32_t timestamp;     // 时间戳 (ms)
} fall_detection_data_t;

typedef enum {
    FALL_STATE_IDLE = 0,
    FALL_STATE_SUSPECTED,
    FALL_STATE_CONFIRMED,
    FALL_STATE_COUNTDOWN,
    FALL_STATE_SOS_SENT,
} fall_state_t;

typedef struct {
    fall_state_t state;                     // 当前状态
    float svm_buffer[WINDOW_SIZE];          // SVM 滑动窗口缓冲区
    uint8_t buffer_index;                   // 缓冲区索引
    float pitch;                            // 累计俯仰角
    float roll;                             // 累计横滚角
    uint32_t suspect_time;                  // 疑似跌倒触发时间
    uint32_t confirm_time;                  // 确认跌倒触发时间
    fall_event_callback_t callback;         // 事件回调
    void *user_data;                        // 用户数据指针
} fall_detection_ctx_t;
```

### 5.2 核心函数

```c
/**
 * @brief 初始化跌倒检测模块
 * @param callback 事件回调函数
 * @param user_data 用户数据指针
 * @return esp_err_t
 */
esp_err_t fall_detection_init(fall_event_callback_t callback, void *user_data);

/**
 * @brief 输入传感器数据，执行检测
 * @param data BMI270 传感器数据
 * @return esp_err_t
 */
esp_err_t fall_detection_feed(bmi270_data_t *data);

/**
 * @brief 用户取消呼救
 */
void fall_detection_cancel(void);

/**
 * @brief 获取当前检测状态
 * @return fall_state_t
 */
fall_state_t fall_detection_get_state(void);

/**
 * @brief 重置检测器（回到 IDLE 状态）
 */
void fall_detection_reset(void);
```

### 5.3 状态机实现伪代码

```c
esp_err_t fall_detection_feed(bmi270_data_t *data)
{
    // 1. 滑动窗口滤波
    float svm = calculate_svm(data);
    svm = moving_average_filter(svm, ctx);

    // 2. 更新姿态角（陀螺仪积分）
    ctx->pitch += data->gyro_x * (1.0f / SAMPLE_RATE_HZ);
    ctx->roll  += data->gyro_y * (1.0f / SAMPLE_RATE_HZ);

    // 3. 状态机处理
    switch (ctx->state) {
        case FALL_STATE_IDLE:
            if (svm > FALL_THRESHOLD_G) {
                ctx->state = FALL_STATE_SUSPECTED;
                ctx->suspect_time = get_timestamp_ms();
                ctx->pitch = 0;  // 重置姿态角
                ctx->roll = 0;
                ctx->callback(FALL_EVENT_SUSPECTED, ctx->user_data);
            }
            break;

        case FALL_STATE_SUSPECTED:
            // 在观察窗口内检查姿态角变化
            uint32_t elapsed = get_timestamp_ms() - ctx->suspect_time;
            if (elapsed > OBSERVE_WINDOW_S * 1000) {
                // 超时未确认，回到 IDLE
                ctx->state = FALL_STATE_IDLE;
                ctx->callback(FALL_EVENT_CANCELLED, ctx->user_data);
            } else {
                float delta = sqrt(ctx->pitch * ctx->pitch + ctx->roll * ctx->roll);
                if (delta > ANGLE_THRESHOLD_DEG) {
                    ctx->state = FALL_STATE_CONFIRMED;
                    ctx->confirm_time = get_timestamp_ms();
                    ctx->callback(FALL_EVENT_CONFIRMED, ctx->user_data);
                }
            }
            break;

        case FALL_STATE_CONFIRMED:
            // 进入倒计时状态
            ctx->state = FALL_STATE_COUNTDOWN;
            break;

        case FALL_STATE_COUNTDOWN:
            // 倒计时由外部定时器管理
            // 超时时调用 fall_detection_timeout()
            break;

        case FALL_STATE_SOS_SENT:
            // 已发送呼救，等待系统重置
            break;
    }

    return ESP_OK;
}
```

---

## 六、测试方案

### 6.1 单元测试

| 测试用例 | 输入 | 期望输出 |
|---------|------|---------|
| 静止状态 | ax=0, ay=0, az=1g | SVM=1g, 无事件 |
| 正常行走 | SVM 波动 1.5~2.5g | 无事件 |
| 快速坐下 | SVM 短暂 2.8g | 无事件 (低于阈值) |
| 跌倒冲击 | SVM=6g, 姿态角变化 80° | FALL_EVENT_CONFIRMED |
| 跌倒后站起 | SVM=6g, 姿态角变化 30° | FALL_EVENT_SUSPECTED → CANCELLED |

### 6.2 集成测试

| 测试场景 | 步骤 | 预期结果 |
|---------|------|---------|
| 正常监测 | 系统运行 5 分钟，正常活动 | 无误报 |
| 跌倒检测 | 模拟跌倒 (快速冲击 + 姿态变化) | 触发语音确认 |
| 用户取消 | 跌倒后 10s 内按下取消按钮 | 回到正常监测 |
| 超时呼救 | 跌倒后等待 30s 不操作 | 自动发送呼救 |

---

## 七、参考资料

- [BMI270 数据手册](https://www.bosch-sensortec.com/products/motion-sensors/imus/bmi270/)
- [基于 IMU 的跌倒检测算法综述](https://ieeexplore.ieee.org/document/1234567) (示例)
- ESP-IDF I2C 驱动示例：`examples/peripherals/i2c`
- ESP-IDF RMT 驱动示例：`examples/peripherals/rmt/led_strip`
