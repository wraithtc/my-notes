**背景
1. 低功耗4G定位器，与[[4G模组低功耗技术调研]]背景相同
2. MQTT心跳由MQTT库发送，报文不可控，业务报文（定位信息，电量信息等）由业务代码控制可控
3. 目的是怎么尽量减少mcu从休眠中唤醒，一次唤醒完成所有上报任务
4. 要求设计算法做到极致低功耗

---

## 一、问题分析

### 1.1 核心矛盾

```
┌──────────────────────────────────────────────────────────────┐
│  MCU休眠中...                                                │
│  ████████████████████████████████████████████████████████████ │
│  ↑ heartbeat       ↑ heartbeat       ↑ biz_report  ↑ hb     │
│  第1次唤醒         第2次唤醒          第3次唤醒      第4次唤醒 │
│  (仅心跳)          (仅心跳)          (仅业务)       (仅心跳)  │
│                                                                │
│  ❌ 朴素方案：4次唤醒，每次仅做一件事 → 功耗翻倍               │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│  MCU休眠中...                                                │
│  ████████████████████████████████████████████████████████████ │
│  ↑ heartbeat       ↑ heartbeat+biz_report                    │
│  第1次唤醒         第2次唤醒                                  │
│  (心跳)            (心跳 + 业务合并)                          │
│                                                                │
│  ✅ 优化方案：2次唤醒，业务合并到心跳时刻 → 功耗减半           │
└──────────────────────────────────────────────────────────────┘
```

### 1.2 约束条件

| 约束                | 说明                           |
| ----------------- | ---------------------------- |
| MQTT心跳报文不可控       | MQTT库自动发送PINGREQ，内容和时机由库控制   |
| 业务报文可控            | 定位、电量等PUBLISH报文，时机和内容由业务代码控制 |
| MQTT Keep-Alive可调 | 可设置较大的Keep-Alive值降低心跳频率      |
| MCU使用Stop模式       | RAM保持，唤醒后恢复时钟即可继续执行          |

### 1.3 优化目标

$$
\min \sum_{i=1}^{N} E_{wakeup\_i}
$$

其中 $E_{wakeup}$ 为单次唤醒能耗，$N$ 为统计周期内总唤醒次数。核心策略是 **最小化 N**。

---

## 二、核心算法：心跳对齐统一调度（HBUS）

### 2.1 算法思想

> **HBUS = Heartbeat-Bound Unified Scheduling**
>
> 将 MQTT Keep-Alive 周期 $T_{hb}$ 作为**基准时间槽**，所有业务报文周期对齐到 $T_{hb}$ 的整数倍，实现"一次唤醒，完成所有任务"。

**三条原则：**

1. **心跳驱动唤醒**：MCU 唤醒时机仅由心跳周期 $T_{hb}$ 决定
2. **业务对齐心跳**：业务报文周期强制为 $T_{hb}$ 的整数倍，在心跳唤醒时顺带发送
3. **零额外唤醒**：除事件驱动（如SOS报警），不存在"仅为业务数据而唤醒"的情况

### 2.2 时间槽模型

```
时间轴 (以 T_hb 为一个slot)
├── slot 0 ──┤── slot 1 ──┤── slot 2 ──┤── slot 3 ──┤── slot 4 ──┤── slot 5 ──┤
│    HB       │    HB       │  HB + LOC   │    HB       │    HB       │HB+LOC+BAT │
│             │             │             │             │             │           │
│ 唤醒发心跳  │ 唤醒发心跳  │心跳+位置上报 │ 唤醒发心跳  │ 唤醒发心跳  │心跳+位置+电量│

T_hb = 5min,  Location = 3 × T_hb = 15min,  Battery = 6 × T_hb = 30min
```

### 2.3 业务数据结构

```c
typedef struct {
    uint8_t   type;          // 业务类型标识
    uint8_t   multiplier;    // 对齐系数 = 实际周期 / T_hb
    uint8_t   priority;      // 优先级 (事件驱动 > 周期性)
    bool      (*prepare)(void);   // 数据采集函数
    int       (*publish)(void);   // 数据发送函数
} BizTask_t;

// 业务注册表
static BizTask_t biz_registry[] = {
    // type              multiplier  priority  prepare_fn       publish_fn
    { BIZ_LOCATION,      3,          2,        gps_read,        loc_publish  },  // 每15min
    { BIZ_BATTERY,       12,         3,        adc_read_bat,    bat_publish  },  // 每60min
    { BIZ_ALARM_STATUS,  6,          2,        alarm_check,     alarm_publish},  // 每30min
};
#define BIZ_COUNT  (sizeof(biz_registry) / sizeof(biz_registry[0]))
```

### 2.4 核心调度算法

```c
// ========== 全局变量 ==========
static volatile uint32_t slot_counter = 0;    // 当前slot计数
static uint32_t last_hb_timestamp = 0;        // 上次心跳时间(RTC)
static uint32_t T_hb = 300;                   // 心跳周期(秒)，如5分钟

// ========== 最小公倍数计算 ==========
// LCM决定完整周期长度，用于slot_counter归零
static uint32_t calc_schedule_lcm(void) {
    uint32_t lcm = 1;
    for (int i = 0; i < BIZ_COUNT; i++) {
        uint32_t m = biz_registry[i].multiplier;
        lcm = lcm * m / gcd(lcm, m);
    }
    return lcm;
}

// ========== 进入休眠 ==========
void enter_sleep(void) {
    uint32_t lcm = calc_schedule_lcm();

    // 计算下一个slot的时间
    slot_counter = (slot_counter + 1) % lcm;
    uint32_t next_wake = last_hb_timestamp + T_hb;

    // 设置RTC闹钟
    RTC_SetAlarm(next_wake - MARGIN_SECONDS);

    // 通知4G模组进入PSM
    send_at_command("AT+CPSMS=1");

    // MCU进入Stop模式
    HAL_PWR_EnterSTOPMode(PWR_LOWPOWERREGULATOR_ON, PWR_STOPENTRY_WFI);
}

// ========== 唤醒后处理 ==========
void on_wakeup(void) {
    // Step 1: 恢复硬件
    SystemClock_Config();          // 恢复系统时钟
    restore_peripherals();         // 恢复外设

    uint32_t now = RTC_GetTime();

    // Step 2: 检查唤醒源
    WakeupSource_t src = get_wakeup_source();

    if (src == WAKEUP_RTC_ALARM) {
        // ---- 定时唤醒（心跳驱动） ----

        // 2a: MQTT库发送心跳 (库自动检测keep-alive超时)
        //     Tickless Idle恢复后，FreeRTOS补偿tick，MQTT任务自动触发
        //     无需手动调用，等MQTT任务执行即可
        vTaskResume(mqtt_task_handle);
        vTaskDelay(pdMS_TO_TICKS(500));  // 等待心跳发送完成

        // 2b: 执行到期业务任务
        for (int i = 0; i < BIZ_COUNT; i++) {
            if (slot_counter % biz_registry[i].multiplier == 0) {
                biz_registry[i].prepare();   // 采集数据
                biz_registry[i].publish();   // 发送数据
            }
        }

        last_hb_timestamp = now;

    } else if (src == WAKEUP_GPIO || src == WAKEUP_UART) {
        // ---- 事件驱动唤醒（下行消息/外部中断） ----
        process_event(src);
    }

    // Step 3: 处理完毕，重新进入休眠
    enter_sleep();
}
```

### 2.5 调度示例

假设配置：
- $T_{hb}$ = 5分钟，Location = 3×$T_{hb}$ = 15min，Battery = 12×$T_{hb}$ = 60min

| slot | counter | Heartbeat | Location | Battery | 操作 |
|------|---------|-----------|----------|---------|------|
| 0    | 0       | ✅        | ✅       | ✅      | 心跳+位置+电量 |
| 1    | 1       | ✅        |          |         | 仅心跳 |
| 2    | 2       | ✅        |          |         | 仅心跳 |
| 3    | 3       | ✅        | ✅       |         | 心跳+位置 |
| 4    | 4       | ✅        |          |         | 仅心跳 |
| 5    | 5       | ✅        |          |         | 仅心跳 |
| 6    | 6       | ✅        | ✅       |         | 心跳+位置 |
| ...  | ...     | ...       | ...      | ...     | ... |
| 11   | 11      | ✅        | ✅       |         | 心跳+位置 |
| 12   | 12→0    | ✅        | ✅       | ✅      | 心跳+位置+电量 (周期重启) |

**1小时内唤醒次数 = 12次**（仅心跳频率，无额外唤醒）

---

## 三、进阶优化策略

### 3.1 Keep-Alive最大化

MQTT Keep-Alive 越大，唤醒越少。需在**连接可靠性**与**功耗**之间权衡：

| Keep-Alive | 1小时唤醒次数 | 连接断开检测延迟 | 适用场景 |
|------------|-------------|----------------|---------|
| 60s        | 60          | ~2min          | 实时监控 |
| 300s (5min)| 12          | ~10min         | **推荐：定位器** |
| 600s (10min)| 6          | ~20min         | 低频上报 |
| 1800s (30min)| 2         | ~60min         | 极低功耗 |

```c
// 推荐配置：与MQTT Broker协商最大Keep-Alive
#define MQTT_KEEPALIVE_DEFAULT   300   // 5分钟
#define MQTT_KEEPALIVE_MAX       1800  // 30分钟（需broker支持）

// MQTT连接时设置
mqtt_client_config.keepalive = MQTT_KEEPALIVE_DEFAULT;
```

### 3.2 动态周期调整

根据场景动态调整业务上报频率，进一步降低功耗：

```c
typedef struct {
    uint8_t mode;               // 当前模式
    uint8_t loc_multiplier;     // 位置上报倍数
    uint8_t bat_multiplier;     // 电量上报倍数
} PowerMode_t;

static const PowerMode_t power_modes[] = {
    // mode               loc  bat   说明
    { MODE_NORMAL,         3,   12 }, // 正常：15min定位, 60min电量
    { MODE_SAVE,           6,   24 }, // 省电：30min定位, 120min电量
    { MODE_ULTRA_SAVE,    12,   36 }, // 超省电：60min定位, 180min电量
    { MODE_ALARM,          1,    6 }, // 报警：5min定位, 30min电量
};

// 根据条件切换模式
void update_power_mode(void) {
    PowerMode_t *cfg;
    uint8_t bat_level = get_battery_level();

    if (alarm_is_active()) {
        cfg = &power_modes[MODE_ALARM];
    } else if (bat_level < 10) {
        cfg = &power_modes[MODE_ULTRA_SAVE];  // 电量<10%进入超省电
    } else if (is_night_time()) {
        cfg = &power_modes[MODE_SAVE];         // 夜间省电
    } else {
        cfg = &power_modes[MODE_NORMAL];
    }

    // 更新业务注册表的multiplier
    biz_registry[BIZ_LOCATION].multiplier = cfg->loc_multiplier;
    biz_registry[BIZ_BATTERY].multiplier  = cfg->bat_multiplier;
}
```

### 3.3 事件驱动快速上报

对于SOS、围栏报警等紧急事件，需要**立即唤醒**，不等待心跳对齐：

```c
// 事件驱动的立即唤醒（中断上下文）
void EXTI_SOS_Alarm_Handler(void) {
    // 标记紧急事件
    emergency_flag = EMERGENCY_SOS;
    // 唤醒MCU（如果已休眠则触发中断退出Stop模式）
    // Stop模式下EXTI中断自动唤醒
}

// 唤醒后优先处理紧急事件
void on_wakeup(void) {
    SystemClock_Config();

    // 优先处理紧急事件（不等心跳）
    if (emergency_flag != EMERGENCY_NONE) {
        wakeup_4g_module();          // 唤醒模组
        handle_emergency(emergency_flag);
        emergency_flag = EMERGENCY_NONE;
    }

    // 然后走正常心跳+业务调度流程
    // ...
}
```

### 3.4 业务数据预采集

在心跳唤醒瞬间先采集传感器数据，与心跳报文"流水线"发送：

```c
void on_wakeup_optimized(void) {
    SystemClock_Config();

    // Phase 1: 立即启动传感器采集（与MQTT恢复并行）
    bool biz_due = check_biz_due(slot_counter);
    if (biz_due) {
        // 启动GPS定位/ADC采样（利用MQTT重连的等待时间）
        start_sensor_sampling();
    }

    // Phase 2: MQTT心跳发送（库自动执行）
    vTaskResume(mqtt_task_handle);
    // 等待心跳发送 + 同时等传感器就绪
    wait_mqtt_and_sensors_ready();

    // Phase 3: 发送业务数据（传感器数据已就绪）
    if (biz_due) {
        publish_biz_data();
    }

    enter_sleep();
}
```

**时序对比：**

```
朴素串行：  wake → [MQTT重连+心跳 2s] → [GPS采集 3s] → [发送业务 1s] = 6s
并行优化：  wake → [MQTT重连+心跳 2s | GPS采集 3s] → [发送业务 1s] = 4s
                                                                    ↑ 节省2s活跃时间
```

---

## 四、功耗分析

### 4.1 唤醒能耗模型

单次唤醒能耗：

$$
E_{wakeup} = I_{active} \times V \times T_{active}
$$

| 阶段 | 电流 | 时长 | 说明 |
|------|------|------|------|
| 时钟恢复 | ~10mA | ~5ms | SystemClock_Config |
| 4G模组唤醒 | ~50mA | ~500ms | 从PSM恢复 |
| MQTT心跳发送 | ~150mA | ~500ms | PINGREQ/PINGRESP |
| 业务数据采集+发送 | ~150mA | ~1000ms | 仅在业务slot |
| **仅心跳唤醒** | ~100mA avg | **~1s** | |
| **心跳+业务唤醒** | ~150mA avg | **~2s** | |

### 4.2 方案对比（24小时）

配置：$T_{hb}$ = 5min, Location = 15min, Battery = 60min

| 方案 | 心跳唤醒 | 业务额外唤醒 | 总唤醒次数 | 日耗电(mAh) | 续航(天) |
|------|---------|------------|-----------|------------|---------|
| ❌ 朴素（不对齐） | 288 | 96+24=120 | **408** | ~136 | **5.9** |
| ✅ HBUS对齐 | 288 | 0 | **288** | ~96 | **8.3** |
| ✅ HBUS + T_hb=10min | 144 | 0 | **144** | ~48 | **16.7** |
| ✅ HBUS + T_hb=10min + 动态省电 | ~96 | 0 | **~96** | ~32 | **25** |
| ✅ HBUS + T_hb=30min + 夜间省电 | ~72 | 0 | **~72** | ~24 | **33** |

### 4.3 极致功耗优化路径

```
基础方案                    →  HBUS对齐     →  增大T_hb    →  动态模式    →  并行采集
  408次/天                     288次/天        144次/天       ~96次/天      每次节省30%时间
  5.9天续航                    8.3天续航       16.7天续航     25天续航      33天续航
  ─────────────────────────────────────────────────────────────────────────────────→
                                                     功耗优化方向
```

---

## 五、异常处理

### 5.1 MQTT连接断开

```c
void on_wakeup(void) {
    // ...
    if (mqtt_is_connected()) {
        // 正常流程
    } else {
        // 连接已断开，需重连
        mqtt_reconnect();
        // 重连成功后立即发送业务数据（不等slot对齐）
        // 因为断连期间可能丢失了数据
        publish_all_pending_biz();
    }
}
```

### 5.2 心跳发送失败

```c
// 在MQTT库的回调中处理心跳失败
void mqtt_ping_callback(int result) {
    if (result != MQTT_SUCCESS) {
        fail_count++;
        if (fail_count >= 3) {
            // 连续3次失败，标记需要重连
            need_reconnect = true;
        }
    } else {
        fail_count = 0;
    }
}
```

### 5.3 RTC时间漂移补偿

```c
// 每次唤醒时用NTP校准RTC
void compensate_rtc_drift(void) {
    if (slot_counter % 12 == 0) {  // 每小时校准一次
        uint32_t ntp_time = ntp_get_time();
        if (ntp_time > 0) {
            uint32_t drift = abs(ntp_time - RTC_GetTime());
            if (drift > RTC_DRIFT_THRESHOLD) {
                RTC_SetTime(ntp_time);
            }
        }
    }
}
```

---

## 六、实施清单

- [ ] 确定MQTT Broker支持的最大Keep-Alive值
- [ ] 定义业务数据类型及其上报周期
- [ ] 确认所有业务周期可被 $T_{hb}$ 整除（或调整到可整除）
- [ ] 实现 `slot_counter` 调度逻辑
- [ ] 实现动态模式切换
- [ ] 实现传感器与MQTT并行流水线
- [ ] 实现事件驱动唤醒的紧急通道
- [ ] 搭建功耗测试环境，测量实际唤醒能耗
- [ ] 压测：连续运行72小时验证稳定性
- [ ] 边界测试：断网重连、低电量、RTC漂移场景

---

## 更新记录

- 2026-04-07: 设计HBUS心跳对齐统一调度算法，含功耗分析和进阶优化策略
