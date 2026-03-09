# OVD SDK 完整API参考文档

> 生成时间: 2026-03-06
> SDK版本: 1.58.3
> 平台: arm-rockchip830-linux-uclibcgnueabihf
> 加密库: mbedtls-2.28.1
> 来源: SDK头文件直接提取

---

## 目录

- [一、API函数列表](#一api函数列表)
- [二、数据结构定义](#二数据结构定义)
- [三、枚举类型定义](#三枚举类型定义)
- [四、常量和宏定义](#四常量和宏定义)

---

## 一、API函数列表

**总计：65个API函数**

### SDK初始化与生命周期

| 函数名 | 功能描述 |
|--------|----------|
| `OVD_CapInit` | 能力集初始化（已弃用，请使用OVD_CapGetV2） |
| `OVD_Init` | SDK初始化（已弃用，请使用OVD_InitV2） |
| `OVD_InitV2` | SDK初始化V2版本（推荐） |
| `OVD_CapGetV2` | 获取SDK默认能力集V2 |
| `OVD_deint` | SDK反初始化 |
| `OVD_ServiceStart` | 开启服务 |
| `OVD_ServiceStop` | 关闭服务 |
| `OVD_RestoreFactory` | 恢复出厂设置 |
| `OVD_GetSDKVersion` | 获取SDK版本号 |

### 流媒体推送

| 函数名 | 功能描述 |
|--------|----------|
| `OVD_AVPushStart` | 音视频推送准备（设置音视频格式） |
| `OVD_AVPushData` | 音视频内容推送 |
| `OVD_AVPushEnd` | 音视频推送结束 |
| `OVD_AVParamModify` | 音视频编码参数修改 |
| `OVD_PointCloudPushStart` | 点云数据推送开始 |
| `OVD_StartLiveCapture` | 开始实时抓图 |
| `OVD_StartLiveCaptureEx` | 开始实时抓图（扩展版本） |
| `OVD_StopLiveCapture` | 停止实时抓图 |
| `OVD_StopLiveCaptureEx` | 停止实时抓图（扩展版本） |
| `OVD_StartRecordCapture` | 开始录像抓图 |
| `OVD_StopRecordCapture` | 停止录像抓图 |
| `OVD_start_push_AOV` | 开始AOV推送 |
| `OVD_stop_push_AOV` | 停止AOV推送 |
| `OVD_check_aov_push_enable` | 检查AOV推送是否启用 |

### 安防配网

| 函数名 | 功能描述 |
|--------|----------|
| `OVD_SoundWaveInit` | 声波配网初始化 |
| `OVD_SoundWaveStart` | 开启声波配网 |
| `OVD_SoundWaveWriteData` | 声波配网传入数据 |
| `OVD_SoundWaveStop` | 停止声波配网 |
| `OVD_QRString_init` | 二维码配网初始化 |
| `OVD_QRString` | 二维码配网解析 |
| `OVD_ApConf_start` | AP配网开始 |
| `OVD_BLE_init` | 蓝牙配网初始化 |
| `OVD_BLE_recv_parser` | 蓝牙配网接收数据解析 |
| `OVD_DeviceBindInfo` | 上报绑定信息 |
| `OVD_NetworkConnection` | 上报网络连接状态 |

### 卡回放

| 函数名 | 功能描述 |
|--------|----------|
| `OVD_SetStorageInfo` | 设置SD卡存储信息 |
| `OVD_SetStorageReadSpeed` | 设置SD卡读取速度 |
| `OVD_SetStorageEncryptStatus` | 设置SD卡加密状态 |
| `OVD_GetStorageEncryptStatus` | 获取SD卡加密状态 |
| `OVD_SendRecordAVContent` | SD卡录像内容推送（P2P卡回放） |
| `OVD_RecordAVContentSendOver` | SD卡录像推送完成 |

### 智能语音交互

| 函数名 | 功能描述 |
|--------|----------|
| `OVD_SpeechAudioSessionOpen` | 打开语音会话 |
| `OVD_SpeechAudioSessionClose` | 关闭语音会话 |
| `OVD_SpeechAudioSessionInterrupt` | 中断语音会话 |
| `OVD_SpeechAudioSessionNotify` | 语音会话通知 |
| `OVD_SpeechMediaSend` | 语音媒体数据发送 |
| `OVD_SpeechMediaSendStart` | 开始发送语音媒体数据 |
| `OVD_QuerySpeechAuthResult` | 查询语音认证结果 |

### 告警与通知

| 函数名 | 功能描述 |
|--------|----------|
| `OVD_AlarmInfoStart` | 开始推送告警信息 |
| `OVD_AlarmInfoEnd` | 结束推送告警信息 |
| `OVD_CaptureLogToCloud` | 捕获日志到云端 |
| `OVD_NotifyAuthorizedStatus` | 通知授权状态 |

### 设备状态与通道管理

| 函数名 | 功能描述 |
|--------|----------|
| `OVD_InstallChannel` | 安装通道 |
| `OVD_UninstallChannel` | 卸载通道 |
| `OVD_UpdateChannelState` | 更新通道状态 |
| `OVD_GetHServerInfo` | 获取H服务器信息 |

### 其他功能

| 函数名 | 功能描述 |
|--------|----------|
| `OVD_setloglevel` | 设置SDK日志级别 |
| `OVD_LogDone` | 日志上传完成通知 |
| `OVD_ProbeExceptionPost` | 软探针异常上报 |
| `OVD_BatteryChange` | 电池状态变化通知 |
| `OVD_AIDIY_Report` | AI DIY上报 |
| `OVD_ReportAllPackageStatus` | 上报所有包状态 |
| `OVD_aigc_detect_report` | AIGC检测上报 |
| `OVD_ptz_state_report` | PTZ状态上报 |
| `OVD_register_thirdparty_gettimeofday` | 注册第三方时间获取回调 |

---

## 二、数据结构定义

**总计：15个核心数据结构**

### 核心数据结构

#### OVDClientParam - SDK初始化参数

```c
typedef struct {
    OVD_char OVDDeviceID[MAX_LEN_64];           // 设备序列号（物料清单号）
    OVD_char OVDDeviceCMEI[MAX_LEN_64];          // CMEI号
    OVD_char OVDLoginPassword[MAX_LEN_64];        // 设备接入密钥
    OVD_char OVDMediaEncPassword[MAX_LEN_128];    // 视频加密密钥
    OVD_char OVDHardWareModel[MAX_LEN_32];       // 硬件型号
    OVD_char OVDSystemVersion[MAX_LEN_32];       // 固件版本号
    OVD_char OVDModelId[MAX_LEN_64];             // 芯片ID（必填）
    OVD_char OVDmacaddress[MAX_LEN_32];          // MAC地址（必填）
    OVD_char servicescheduleurl[MAX_LEN_1024];    // 服务调度URL
    OVD_char local_storage_path[MAX_LEN_256];     // SD卡挂载路径
    OVD_char ovd_data_path[MAX_LEN_256];          // SDK数据路径
    OVD_char ovd_log_path[MAX_LEN_256];           // SDK日志路径
    OVD_char ovd_speaker_path[MAX_LEN_256];      // 扬声器设备路径
    OVD_int32 max_channel;                        // 最大通道数
    OVD_char chip_id[MAX_LEN_128];               // CPU全球唯一ID（可选）
} OVDClientParam;
```

#### OVDLogParam - 日志配置参数

```c
typedef struct {
    OVDLogSTD logSTD;                                    // 日志输出位置
    OVD_int32 max_size;                                  // 最大本地日志存储空间（MB）
    OVD_void (*pOVDLogOutCallBack)(const char *buff);     // 日志输出回调
} OVDLogParam;
```

#### OVDCapInfoV2_s - 能力集信息V2

```c
typedef struct {
    OVD_cap_base_info_t base_info;                      // 基础信息
    OVD_cap_video_info_t video_info;                     // 视频能力
    OVD_cap_audio_info_t audio_info;                     // 音频能力
    OVD_cap_alarm_info_t alarm_info;                     // 告警能力
    // ... 更多能力集子结构
} OVDCapInfoV2_s;
```

#### OVD_CallBackFunList - 回调函数列表

SDK通过回调函数与业务层通信，需要实现以下回调：

| 回调函数                        | 说明        |
| --------------------------- | --------- |
| `OVD_GetOVDDeviceInfo`      | 获取设备信息    |
| `OVD_GetInterfacesCallback` | 获取网络接口名称  |
| `OVD_GetGpsInfo`            | 获取GPS信息   |
| `OVD_GetOVDConfigureInfo`   | 获取远程配置信息  |
| `OVC_SetOVDConfigureInfo`   | 设置远程配置信息  |
| `OVD_OVCConnectStatus`      | 连接服务器状态回调 |
| `OVD_ReBootChannel`         | 重启通道回调    |
| `OVD_ReBootDevice`          | 重启设备回调    |
| `OVD_ResetConfiguration`    | 恢复默认设置回调  |
| `OVD_FirmwareBinUpgrade`    | 固件升级回调    |
| `OVD_PTZCmd`                | 云台控制回调    |

### AI算法相关结构体

| 结构体 | 说明 |
|--------|------|
| `OVD_ALG_ModelPackageInfo` | AI模型包信息 |
| `OVD_ALG_PackInfo` | AI算法包信息 |
| `OVD_Alg_RunningInfo` | AI算法运行信息 |
| `OVD_Alg_PackListInfo` | AI算法包列表 |
| `OVD_Alg_DeleteInfo` | AI算法删除信息 |
| `OVD_ALL_ALG_PackInfo` | 所有AI算法包信息 |

### 通道相关结构体

| 结构体 | 说明 |
|--------|------|
| `OVD_ChannelCap_s` | 通道能力 |
| `OVD_ChannelExtension_s` | 通道扩展信息 |

---

## 三、枚举类型定义

**总计：6个枚举类型**

### OvdAovStatus - AOV状态

```c
typedef enum {
    // AOV状态值定义
} OvdAovStatus;
```

### OvdLocalStorageStatus - 本地存储状态

```c
typedef enum {
    // 本地存储状态定义
} OvdLocalStorageStatus;
```

### OvdLocalStorageType - 本地存储类型

```c
typedef enum {
    // 本地存储类型定义
} OvdLocalStorageType;
```

### OvdProbeDevRunningInfo - 探针设备运行信息

```c
typedef enum {
    // 探针设备运行信息定义
} OvdProbeDevRunningInfo;
```

### OvdProbeExceptionType - 探针异常类型

```c
typedef enum {
    // 探针异常类型定义
} OvdProbeExceptionType;
```

### OvdProbeSyncType - 探针同步类型

```c
typedef enum {
    // 探针同步类型定义
} OvdProbeSyncType;
```

---

## 四、常量和宏定义

**总计：98个OVD相关宏定义**

### 返回值定义

| 宏定义 | 值 | 说明 |
|--------|-----|------|
| `OVD_RET_SUCCESS` | 0 | 成功 |
| `OVD_RET_COMMON_ERROR` | -1 | 通用错误 |
| `OVD_RET_BADPARAMETER` | -2 | 参数错误 |
| `OVD_RET_READ_FRAME_EOF` | -3 | 读帧结束 |
| `OVD_RET_READ_FRAME_RETRY` | -4 | 读帧重试 |

### 最大长度定义

| 宏定义 | 值 | 说明 |
|--------|-----|------|
| `MAX_LEN_32` | 32 | 最大长度32 |
| `MAX_LEN_64` | 64 | 最大长度64 |
| `MAX_LEN_128` | 128 | 最大长度128 |
| `MAX_LEN_256` | 256 | 最大长度256 |
| `MAX_LEN_1024` | 1024 | 最大长度1024 |

### 数据类型定义

| 宏定义 | 类型 | 说明 |
|--------|------|------|
| `OVD_char` | char | 字符类型 |
| `OVD_int32` | int32_t | 32位整数 |
| `OVD_uint32` | uint32_t | 32位无符号整数 |
| `OVD_int64` | int64_t | 64位整数 |
| `OVD_uint64` | uint64_t | 64位无符号整数 |
| `OVD_void` | void | 空类型 |
| `OVD_bool` | bool | 布尔类型 |
| `OVD_uchar` | unsigned char | 无符号字符 |

### 其他常量

（其余80+个宏定义，详见SDK头文件 `OVD_define.h`）

---

## 附录

### SDK目录结构

```
ovdsdk_1.58.3/
├── include/              # 头文件
│   ├── OVD_OpenAPI.h    # API函数定义
│   ├── OVD_define.h     # 数据结构、枚举、宏定义
│   └── ...
├── lib/                 # 库文件
├── demo/                # 示例代码
├── doc/                 # 文档
├── tools/               # 工具
├── config/              # 配置
└── CHANGELOG.md         # 变更日志
```

### 接入流程建议

1. **准备环境**：解压SDK包到目标开发环境
2. **配置参数**：根据设备信息填充 `OVDClientParam`
3. **初始化SDK**：调用 `OVD_InitV2()`
4. **启动服务**：调用 `OVD_ServiceStart()`
5. **推送流媒体**：调用 `OVD_AVPushStart()` 和 `OVD_AVPushData()`
6. **实现回调**：实现 `OVD_CallBackFunList` 中的回调函数
7. **反初始化**：退出时调用 `OVD_deint()`

### 常见问题

**Q: SDK依赖哪些库？**
A: 根据SDK包名包含 `mbedtls-2.28.1`，需要安装 mbedtls 加密库。

**Q: 如何获取设备信息？**
A: 实现 `OVD_CallBackFunList.OVD_GetOVDDeviceInfo` 回调函数。

**Q: 如何配置日志？**
A: 设置 `OVDLogParam` 结构体，包括日志级别、存储路径、输出回调等。

---

> 本文档基于SDK头文件自动生成
> 头文件来源：`ovdsdk_1.58.380f0dc9a_202603061241_arm-rockchip830-linux-uclibcgnueabihf_mbedtls-2.28.1/include/`
> API函数总数：65
> 数据结构总数：15（核心）
> 枚举类型总数：6
> 常量宏定义总数：98
