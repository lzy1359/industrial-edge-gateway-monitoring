# 工业边缘网关监控系统

基于 `STM32F411CEU6`、`FreeRTOS`、`LVGL`、`ESP8266`、`Modbus RTU` 和 `W25Q64 SPI Flash` 实现的工业物联网边缘网关项目。系统面向温湿度、光照等现场传感器采集场景，完成了从 RS485 设备轮询、本地触摸屏显示、MQTT 云端上报，到断网缓存、重连补传和运行日志查看的一整套边缘侧链路。


## 项目功能

- 通过 RS485 总线轮询 Modbus 从机，采集温度、湿度、光照数据。
- 使用 ESP8266 AT 固件接入 Wi-Fi，并通过 TCP 透传连接巴法云 MQTT 服务。
- 手动封装 MQTT 连接、订阅、发布、心跳等关键报文，不依赖完整 MQTT SDK。
- 基于 FreeRTOS 拆分 UI、Modbus 采集、MQTT 上报、网络心跳、看门狗等任务。
- 使用 LVGL 构建 320x240 本地监控界面，显示实时数据、连接状态和运行时长。
- 使用 W25Q64 外部 Flash 实现日志区和离线缓存区分区管理。
- 网络断开时将采集数据写入 Flash，重连后分批补传缓存数据。
- 通过独立看门狗 IWDG 提升长期运行时的故障自恢复能力。
- 支持 LCD 空闲自动息屏，数据变化或状态变化时自动亮屏。
- 提供运行日志页面，支持从 Flash 倒序读取并展示最近日志。

## 系统架构

```text
Modbus 传感器
  | RS485
  v
STM32F411CEU6
  |-- FreeRTOS 任务调度
  |-- Modbus 轮询采集
  |-- LVGL 本地显示
  |-- W25Q64 日志/缓存
  |-- IWDG 看门狗
  |
  | USART1
  v
ESP8266 AT 模块
  | Wi-Fi / TCP 透传
  v
巴法云 MQTT 服务
```

核心数据链路：

```text
传感器数据
  -> Modbus_Poll_AllDev()
  -> FreeRTOS Queue
  -> LVGL 页面刷新
  -> MQTT 发布
  -> 网络异常时写入 FlashCache
  -> 网络恢复后批量补传
```

## 硬件组成

| 模块 | 型号/说明 | 用途 |
| --- | --- | --- |
| 主控 | STM32F411CEU6 | 主控 MCU，运行 FreeRTOS 和业务逻辑 |
| 显示 | SPI TFT 触摸屏，320x240 | LVGL 数据展示和日志查看 |
| 联网 | ESP8266 AT 固件 | Wi-Fi 连接和 TCP 透传 |
| 总线 | SP3485 或同类 RS485 收发器 | STM32 与 Modbus 从机通信 |
| 传感器 | Modbus 温湿度、光照传感器 | 现场环境数据采集 |
| 存储 | W25Q64，8MB SPI Flash | 运行日志和离线数据缓存 |
| 调试 | ST-Link | 下载和调试固件 |

## 外设与引脚

| 外设 | 引脚 | 说明 |
| --- | --- | --- |
| USART1 | PA9 TX / PA10 RX | ESP8266，115200-8-N-1，DMA 接收空闲中断 |
| USART2 | PA2 TX / PA3 RX | RS485/Modbus，9600-8-N-1 |
| RS485 DE | PB0 | RS485 发送/接收方向控制 |
| SPI1 | PA5 SCK / PA6 MISO / PA7 MOSI | LCD 通信 |
| SPI2 | PB13 SCK / PB14 MISO / PB15 MOSI | W25Q64 Flash 通信 |
| Flash CS | PB12 | W25Q64 片选 |
| LCD 背光 | PA8 | 背光开关控制 |
| 状态指示 | PB3 / PB4 / PA15 等 | 在线、离线、补传等状态指示 |

## 软件架构

### FreeRTOS 任务

| 任务 | 文件 | 职责 |
| --- | --- | --- |
| `UI` | `code/Taskhandel.c` | 初始化 LCD、触摸和 LVGL，刷新主页面和日志页面 |
| `Modbus` | `code/Taskhandel.c`、`model/modbus.c` | 轮询多个 Modbus 从机并将数据写入队列 |
| `MQTT` | `code/Taskhandel.c`、`SPI_Flash/flash.c` | 发送实时数据，离线时写缓存，在线后补传 |
| `AT` | `code/Taskhandel.c`、`ESP8266/pal.c` | 初始化 MQTT 连接、心跳检测、断线重连 |
| `Dog_task` | `code/Taskhandel.c` | 定时刷新 IWDG，避免系统卡死后无法恢复 |
| `defaultTask` | `Core/Src/freertos.c` | 创建业务任务后删除自身 |

### 模块划分

| 目录 | 说明 |
| --- | --- |
| `Core/` | STM32CubeMX 生成的 HAL 初始化、中断、外设配置代码 |
| `code/` | 业务任务、LCD、触摸、引脚相关代码 |
| `model/` | Modbus RTU 协议、CRC16、从机轮询和数据组包 |
| `ESP8266/` | AT 指令、环形缓冲区、MQTT/TCP 透传相关代码 |
| `SPI_Flash/` | W25Q64 底层驱动、日志分区、离线缓存和补传逻辑 |
| `Lvgl/` | LVGL 源码、移植层和自定义监控页面 |
| `Middlewares/` | FreeRTOS 等中间件 |
| `Drivers/` | CMSIS 和 STM32 HAL 驱动 |
| `MDK-ARM/` | Keil MDK 工程文件和编译产物 |
| `reports/` | 项目审计、面试讲解和简历材料草稿 |

## 关键实现

### 1. Modbus 多从机轮询

`model/modbus.c` 中维护了 `dev_list` 从机表，通过通用的 `Modbus_Read_Dev()` 读取指定地址、功能码和寄存器范围。CRC16 同时保留了按位计算和查表计算两种实现，接收侧使用帧间超时和绝对超时避免总线异常时阻塞整个系统。

当前默认从机：

| 地址 | 功能码 | 数据 |
| --- | --- | --- |
| `0x01` | `0x03` | 温度、湿度 |
| `0x02` | `0x03` | 光照 |

### 2. MQTT 透传通信

ESP8266 使用 AT 指令连接 Wi-Fi，并通过 `AT+CIPSTART` 建立到 `bemfa.com:9501` 的 TCP 连接。进入透传模式后，系统手动发送 MQTT CONNECT、SUBSCRIBE、PUBLISH、PINGREQ 报文，并根据 CONNACK、PINGRESP 判断连接状态。

### 3. 离线缓存与自动补传

W25Q64 被划分为两个逻辑区域：

| 分区 | 地址范围 | 用途 |
| --- | --- | --- |
| 日志区 | `0x000000` 起始，约 4MB | 保存运行日志，日志页面倒序读取 |
| 缓存区 | `0x400000` 到 `0x800000` | 网络离线时保存待上传 JSON 数据 |

离线时，MQTT 任务会将采集数据写入 `FlashCache_Write()`。网络恢复后，系统通过 `FlashCache_Read()` 读取缓存数据，并调用 `MQTT_PublishJson()` 分批补传，发送完成后通过魔术字标记为已处理。

### 4. LVGL 本地界面

`Lvgl/LVGL_myGUI/page_main.c` 实现了主监控页和日志页：

- 主页面显示温度、湿度、光照、连接状态、运行时长。
- 数据变化时刷新标签和进度条。
- 右上角日志按钮进入日志页面。
- 日志页面从 Flash 中倒序读取最近日志，新日志显示在上方。
- 30 秒无动作时关闭 LCD 背光，数据或连接状态变化时重新亮屏。

### 5. 看门狗与自恢复

工程启用了 IWDG 独立看门狗，`Dog_task` 周期性喂狗。若系统长时间卡死或任务调度异常，硬件看门狗会触发复位，提高无人值守运行场景下的恢复能力。

## 编译与烧录

### 环境要求

- Keil MDK-ARM 5.x
- STM32CubeMX 6.x，可选，用于查看或重新生成 `F4project.ioc`
- STM32Cube FW_F4 V1.28.x
- ST-Link 驱动和下载工具
- ESP8266 标准 AT 固件

### 编译步骤

1. 使用 Keil 打开：

   ```text
   MDK-ARM/F4project.uvprojx
   ```

2. 检查目标芯片为 `STM32F411CEUx`。
3. 编译工程，确认无错误。
4. 使用 ST-Link 下载到开发板。
5. 接好 ESP8266、RS485 模块、传感器、LCD 和 W25Q64 后上电测试。

仓库中也包含已生成的 `F4project.hex`，路径为：

```text
MDK-ARM/F4project/F4project.hex
```

## 配置说明

先检查并替换源码中的敏感配置：

- Wi-Fi SSID 和密码
- 巴法云 UID / Client ID
- MQTT 主题名
相关配置和硬编码连接参数主要分布在：
```text
ESP8266/pal.h
ESP8266/pal.c
```

## 使用流程

1. 烧录固件并完成硬件接线。
2. ESP8266 上电后通过 AT 指令连接 Wi-Fi。
3. STM32 初始化 Flash、LCD、触摸、串口、定时器、RTC、IWDG。
4. FreeRTOS 创建各业务任务。
5. Modbus 任务周期采集传感器数据。
6. UI 任务刷新本地页面。
7. MQTT 任务在线时上传数据，离线时写入缓存。
8. 网络任务定时发送心跳，异常时进入重连流程。
9. 网络恢复后自动补传缓存数据。

## 源码阅读入口

| 目标 | 推荐文件 |
| --- | --- |
| 程序入口和外设初始化 | `Core/Src/main.c` |
| FreeRTOS 任务创建 | `Core/Src/freertos.c` |
| 业务任务主逻辑 | `code/Taskhandel.c` |
| Modbus 协议和传感器采集 | `model/modbus.c` |
| ESP8266 AT 指令 | `ESP8266/ESP_at.c` |
| MQTT 初始化和心跳 | `ESP8266/pal.c` |
| 环形缓冲区和串口接收 | `ESP8266/command.c` |
| W25Q64 底层驱动 | `SPI_Flash/bsp_flash.c` |
| 日志和离线缓存 | `SPI_Flash/flash.c` |
| LVGL 监控页面 | `Lvgl/LVGL_myGUI/page_main.c` |

## 技术栈

`STM32F4` `STM32F411` `HAL` `FreeRTOS` `CMSIS-RTOS2` `LVGL` `Modbus RTU` `RS485` `ESP8266` `AT 指令` `MQTT` `SPI Flash` `W25Q64` `IWDG` `DMA` `CRC16` `环形缓冲区` `断网缓存` `自动重连`

## License

本项目使用 MIT License，详见 [LICENSE](LICENSE)。
