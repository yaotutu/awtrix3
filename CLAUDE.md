# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

AWTRIX 3 是一个为 Ulanzi 智能像素时钟 (TC001) 或旧版 AWTRIX 2 主板升级版设计的开源自定义固件。它是一个基于 ESP32 的 LED 矩阵显示系统，主要用于智能家居集成（如 Home Assistant、IOBroker、FHEM、NodeRed）。

核心特点：
- 32x32 LED 矩阵显示器
- MQTT 和 HTTP API 控制
- 内置温湿度传感器支持
- 自定义 App 和通知系统
- RTTTL 旋律播放器
- Web 界面用于设置管理
- 无云、无遥测

## 构建和开发命令

### 构建固件

需要安装 PlatformIO。如果未安装：
```bash
pip install platformio
```

构建固件（支持两种硬件配置）：
```bash
# 为 Ulanzi TC001 构建
pio run --environment ulanzi

# 为 AWTRIX 2 升级版构建
pio run --environment awtrix2_upgrade

# 或构建所有环境
pio run
```

### 上传固件

```bash
# 上传到设备（自动检测端口）
pio run --target upload --environment ulanzi

# 指定串口上传
pio run --target upload --upload-port /dev/ttyUSB0 --environment ulanzi
```

### 监控串口输出

```bash
pio device monitor
# 或指定波特率
pio device monitor -b 115200
```

### 清理构建文件

```bash
pio run --target clean
```

### 格式化文件系统

在首次刷写时需要擦除设备：
```bash
pio run --target erase
```

## 代码架构

### 核心管理器（单例模式）

项目使用多个管理器类，每个都采用单例模式来处理特定功能域：

**DisplayManager** (`src/DisplayManager.h/cpp`)
- 核心显示控制，管理 LED 矩阵
- 处理 App 循环和切换
- 渲染文本、图标、GIF、特效
- 处理通知和自定义页面
- 管理亮度、颜色校正、屏幕方向

**ServerManager** (`src/ServerManager.h/cpp`)
- WiFi 连接管理
- Web 服务器（基于定制的 esp-fs-webserver）
- HTTP API 端点处理
- 文件系统管理（LittleFS）

**MQTTManager** (`src/MQTTManager.h/cpp`)
- MQTT 连接和消息处理
- Home Assistant 自动发现
- 发布状态更新到 MQTT 主题
- 订阅并处理控制命令

**PeripheryManager** (`src/PeripheryManager.h/cpp`)
- 硬件外设管理：按钮（左、中、右、复位）
- 电池监控和 LDR（光敏电阻）读取
- 温湿度传感器（BME280/BMP280/HTU21DF/SHT31）
- 蜂鸣器控制
- DFPlayer 音频播放器支持
- RTTTL 旋律播放

**UpdateManager** (`src/UpdateManager.h/cpp`)
- OTA 固件更新
- 检查 GitHub releases 更新

**MenuManager** (`src/MenuManager.h/cpp`)
- 设备屏幕菜单系统
- 按键导航和设置修改

**GameManager** (`src/Games/GameManager.h/cpp`)
- 内置游戏引擎
- 包括 Slot Machine 和 AwtrixSays 游戏

**PowerManager** (`src/PowerManager.h/cpp`)
- 电源状态管理
- 深度睡眠模式

### 应用系统

**Native Apps** (内置 App)
定义在 `src/Apps.h` 中的 `AppEnum` 枚举，包括：
- TIME, DATE, TEMP, HUM, BAT（基本显示）
- WEATHER, NEWS（通过 MQTT 获取数据）

**Custom Apps**
动态页面，由外部系统通过 MQTT/HTTP 推送内容：
- 定义在 `src/Apps.h` 的 `CustomApp` 结构体
- 不存储逻辑，只显示外部推送的内容
- 可以是文本、图标、GIF、图表、进度条等
- 支持动画、彩虹效果、渐变等特效

### 关键配置文件

**platformio.ini** - 构建配置
- `ulanzi`: 默认环境，用于 Ulanzi TC001
- `awtrix2_upgrade`: 用于 AWTRIX 2 主板升级
- 支持的传感器库：BME280, BMP280, HTU21DF, SHT31
- 构建标志：`-DULANZI` 或 `- Dawtrix2_upgrade`

**dev.json** (运行时配置)
存储在文件系统中，包含高级开发设置：
- 矩阵布局、屏幕旋转/镜像
- 温湿度偏移校准
- 颜色校正和温度
- LDR 和电池校准值
- Home Assistant 前缀
- 调试模式等
- 完整选项见 `docs/dev.md`

### 全局状态

所有全局变量定义在 `src/Globals.h`，包括：
- 网络设置（WiFi、MQTT）
- 显示设置（亮度、颜色、特效）
- 传感器读数（温度、湿度、电池、光照）
- App 循环配置
- 时间和日期格式

### API 接口

**MQTT API**
- 主题前缀：默认为 `awtrix3_[uniqueID]`
- 主要端点：`stats`, `power`, `sleep`, `sound`, `rtttl`, `custom`, `settings`
- 状态广播：`stats`, `loop`, `screen`

**HTTP API**
- 基础 URL: `http://[IP]/api/`
- 端点：`stats`, `power`, `sleep`, `sound`, `rtttl`, `screen`, `effects`, `transitions`, `loop`
- Liveview: `http://[IP]/screen` 或 `http://[IP]/fullscreen`

详见 `docs/api.md`

### 图标和特效系统

**Icons** (`src/icons.h/cpp`)
- 存储在 `/icons/` 目录（运行时）
- 支持单色和动画图标（GIF）
- 可以通过 Web 界面上传

**Effects** (`src/effects.h/cpp`)
- 内置特效：Fade、Pixel、Move、Instant
- 背景特效层
- 可设置为全局背景

### MelodyPlayer

RTTTL 铃声播放器（`src/MelodyPlayer/`）：
- 支持标准 RTTTL 格式
- 蜂鸣器播放（PWM）
- 可选 DFPlayer 硬件支持

### 硬件引脚定义

关键引脚（Ulanzi TC001）：
- GPIO32: LED 矩阵数据线
- GPIO34 (ADC6): 电池传感器
- GPIO35 (ADC7): LDR 光敏传感器
- GPIO26: 左按钮
- GPIO27: 中按钮
- GPIO14: 右按钮
- GPIO15: 蜂鸣器
- GPIO21/22 (SDA/SCL): I2C 传感器

完整引脚定义见 `docs/hardware.md`

## 开发注意事项

### 单例模式访问
所有管理器使用单例模式，访问方式：
```cpp
DisplayManager.setup();
ServerManager.getInstance().someMethod();
```

### 文件系统
使用 LittleFS，配置文件存储在根目录：
- `/settings.json`: 主设置
- `/dev.json`: 开发者设置
- `/custom/`: 自定义 App 配置
- `/icons/`: 图标文件
- `/MELODIES/`: RTTTL 旋律文件

### 调试
使用 `DEBUG_PRINTLN()` 和 `DEBUG_PRINTF()` 宏（在 `src/Globals.h` 中定义）
只有在 `DEBUG` 被定义时才输出到串口

### 矩阵布局问题
如果矩阵显示乱码，需要在 `dev.json` 中调整 `matrix` 参数（0, 1, 或 2）

### 代码风格
- 类名使用 PascalCase
- 方法名使用 camelCase
- 全局变量使用 UPPER_CASE
- 单例模式：`getInstance()` 静态方法

### 依赖库
主要第三方库（都在 `lib/` 目录中定制）：
- FastLED + FastLED_NeoMatrix: LED 控制
- ArduinoJson: JSON 处理
- PubSubClient: MQTT 客户端
- TJpg_Decoder: JPEG 解码（定制）
- GifPlayer: GIF 播放（高度定制）
- esp-fs-webserver: Web 服务器（高度定制）

### Web 界面
HTML/CSS/JS 代码嵌入在 `src/htmls.h` 中，使用 Progmem 存储

### 内存限制
ESP32 内存有限，注意：
- JSON 文档使用 StaticJsonDocument 或固定大小的 DynamicJsonDocument
- 避免大栈分配
- 使用 `DEBUG_PRINTLN` 监控内存使用

## 版本和发布

版本号定义在 `src/Globals.cpp` 中的 `VERSION` 常量。
GitHub Actions 在推送到 main 分支时自动构建固件。
构建的固件被复制到 `docs/ulanzi_flasher/firmware/firmware.bin` 用于在线刷写工具。

## 测试和调试

1. 使用串口监控器查看调试输出
2. Web 界面的 Liveview 功能实时查看显示内容
3. `/api/stats` 端点获取设备状态
4. `dev.json` 中的 `debug_mode: true` 启用详细调试输出

## 在线刷写

项目提供基于浏览器的在线刷写工具：
- Ulanzi TC001: `docs/ulanzi_flasher/index.html`
- AWTRIX 2 升级版: `docs/awtrix2_flasher/index.html`

首次刷写时需要勾选"erase"选项。
