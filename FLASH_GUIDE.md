# AWTRIX3 固件刷写指南

## 📋 概述

本指南适用于 **8行 × 32列** 的 WS2812B LED 矩阵屏幕，该屏幕采用**列优先扫描 + 顺序排列**方式。

本固件已添加 **matrix layout case 3** 配置，专门支持这种屏幕排列方式。

---

## 🎯 硬件规格

- **屏幕尺寸**：8 行 × 32 列
- **LED类型**：WS2812B
- **控制器**：ESP32
- **扫描方式**：列优先（Column-Major Order）
- **排列方式**：顺序排列（Progressive，非锯齿形）
- **数据流向**：
  - 从左上角 (0,0) 开始
  - 每列从上到下填充
  - 列之间从左到右进行
- **坐标公式**：`Index = (x × 8) + y`
  - x = 列号 (0-31)
  - y = 行号 (0-7)

---

## 🚀 快速刷写步骤

### 方法一：使用在线刷写器（推荐）

#### 1. 准备工作

**必需文件已包含在项目中：**
```
docs/ulanzi_flasher/firmware/
├── boot_app0.bin
├── bootloader.bin
├── partitions.bin
├── firmware.bin      ← 包含 case 3 的新固件
└── manifest.json
```

#### 2. 连接设备

1. 用 USB 线连接 ESP32 设备到电脑
2. 确认设备端口：
   - **macOS**: `/dev/cu.usbserial-xxxx`
   - **Linux**: `/dev/ttyUSB0` 或 `/dev/ttyACM0`
   - **Windows**: `COM3`、`COM4` 等

#### 3. 打开刷写器

在浏览器中打开：
```
file:///Users/yaotutu/Desktop/code/awtrix3/docs/ulanzi_flasher/index.html
```

或者从项目目录打开：
```
docs/ulanzi_flasher/index.html
```

#### 4. 开始刷写

1. 点击 **"INSTALL AWTRIX 3"** 按钮
2. 选择正确的串口
3. **首次刷写时勾选 "Erase"**（擦除旧固件）
4. 等待进度条完成（约 1-2 分钟）
5. 刷写完成后设备会自动重启

---

### 方法二：使用 esptool 命令行刷写

```bash
# 安装 esptool
pip install esptool

# 刷写命令
esptool.py --chip esp32 --port /dev/cu.usbserial-xxxx --baud 460800 \
  --before default_reset --after hard_reset \
  write_flash \
  --flash_mode dio --flash_freq 40m --flash_size 4MB \
  0x1000 docs/ulanzi_flasher/firmware/bootloader.bin \
  0x8000 docs/ulanzi_flasher/firmware/partitions.bin \
  0xe000 docs/ulanzi_flasher/firmware/boot_app0.bin \
  0x10000 docs/ulanzi_flasher/firmware/firmware.bin
```

---

## ⚙️ 刷写后配置

### 必须配置：矩阵布局

刷写完成后，**必须配置**使用 case 3 矩阵布局：

#### 方法 A：通过 Web 界面配置

1. 连接设备 WiFi：`AWTRIX3_xxxxx`
2. 浏览器访问：`http://192.168.4.1`
3. 进入 **File Manager**（文件管理）
4. 创建或编辑 `dev.json`：
   ```json
   {
     "matrix": 3
   }
   ```
5. 保存并重启设备

#### 方法 B：通过 API 配置

```bash
curl -X POST http://设备IP/api/settings \
  -H "Content-Type: application/json" \
  -d '{"matrix": 3}'
```

#### 方法 C：通过设备菜单配置

1. 按中间按钮进入菜单
2. 按右键找到 "Matrix" 选项
3. 按选择按钮切换到 "3"
4. 保存并重启

---

## ✅ 验证配置

重启后，检查以下几点：

1. **启动动画正常**：显示 "AWTRIX" 字样
2. **文字方向正确**：不是镜像或倒置
3. **内容清晰可读**：时间、日期等信息正确显示
4. **无乱码或错位**

如果显示异常，参考下方的"故障排除"部分。

---

## 🔧 故障排除

### 问题 1：屏幕完全黑屏

**原因**：固件未正确刷写或矩阵配置错误

**解决**：
1. 重新刷写固件（勾选 Erase）
2. 确认 `dev.json` 中配置了 `"matrix": 3`

---

### 问题 2：显示乱码或错位

**原因**：矩阵布局配置不匹配

**解决**：
1. 尝试其他 matrix 值（0, 1, 2）
2. 或检查你的屏幕是否真的是列优先扫描
3. 参考"硬件测试"部分确认屏幕排列方式

---

### 问题 3：屏幕镜像或倒置

**原因**：屏幕方向设置问题

**解决**：
在 `dev.json` 中添加：
```json
{
  "matrix": 3,
  "rotate_screen": true,   // 旋转180度
  "mirror_screen": false   // 镜像显示
}
```

---

### 问题 4：WiFi 连接不上

**原因**：WiFi 信号差或配置错误

**解决**：
1. 确认设备在路由器覆盖范围内
2. 使用 2.4GHz 频段（ESP32 不支持 5GHz）
3. 在 Web 界面重新配置 WiFi

---

### 问题 5：Web 界面很卡

**原因**：WiFi 信号弱或 LiveView 占用带宽

**解决**：
1. 靠近路由器
2. 关闭 LiveView（实时预览）
3. 清除浏览器缓存
4. 使用 Chrome 浏览器

---

## 📊 矩阵布局参考

### Case 3 配置详解

**代码定义**（`src/DisplayManager.cpp:1690`）：
```cpp
case 3:
  // 列优先 + 顺序排列 (Column-Major + Progressive)
  // 适用于: 8行 x 32列，每列从上到下，列之间从左到右
  // Index = (x * 8) + y, 其中 x 是列号(0-31)，y 是行号(0-7)
  matrix = new FastLED_NeoMatrix(
    leds, 32, 8,
    NEO_MATRIX_TOP + NEO_MATRIX_LEFT +
    NEO_MATRIX_COLUMNS +           // 列优先
    NEO_MATRIX_PROGRESSIVE         // 顺序排列
  );
  break;
```

### 其他矩阵布局对比

| Case | 扫描方式 | 走向 | 适用屏幕 |
|------|---------|------|---------|
| 0 | 行优先 | 锯齿形 | Ulanzi TC001 原装 |
| 1 | 行优先（4个8x8模块）| 顺序 | 4个8x8模块横向 |
| 2 | 列优先 | 锯齿形 | 列优先+锯齿 |
| **3** | **列优先** | **顺序** | **本屏幕** ✅ |

---

## 🛠️ 重新编译固件

如果需要修改代码后重新编译：

### 1. 安装编译环境

```bash
# 创建虚拟环境
python3 -m venv .venv

# 安装 PlatformIO
.venv/bin/pip install platformio
```

### 2. 编译固件

```bash
# 编译 Ulanzi 版本
.venv/bin/pio run --environment ulanzi

# 编译 AWTRIX 2 升级版
.venv/bin/pio run --environment awtrix2_upgrade
```

### 3. 复制到刷写器目录

```bash
cp .pio/build/ulanzi/firmware.bin docs/ulanzi_flasher/firmware/
```

### 4. 更新版本号（可选）

编辑 `src/Globals.cpp` 中的 `VERSION` 常量。

---

## 📝 开发笔记

### 代码修改记录

**修改文件**：`src/DisplayManager.cpp`

**修改位置**：`setMatrixLayout()` 函数，第 1690-1695 行

**修改内容**：添加 case 3 配置

**Commit ID**：`2ebdc29`

**修改量**：+6 行代码

---

### 本地编译信息

- **编译时间**：约 2 分 10 秒
- **Flash 使用率**：96.8%
- **RAM 使用率**：31.0%
- **固件大小**：1.2 MB

---

## 📞 技术支持

### 项目地址

- **原项目**：https://github.com/Blueforcer/awtrix3
- **你的 Fork**：https://github.com/yaotutu/awtrix3

### 参考文档

- **官方文档**：https://blueforcer.github.io/awtrix3/
- **API 文档**：`docs/api.md`
- **开发文档**：`docs/dev.md`
- **硬件说明**：`docs/hardware.md`

---

## 📌 重要提示

1. **首次刷写必须勾选 "Erase"**
2. **刷写后必须配置 `matrix: 3`**
3. **使用 2.4GHz WiFi（不支持 5GHz）**
4. **保持设备与路由器距离 < 10 米**
5. **定期检查 `/api/stats` 监控设备状态**

---

## ✅ 检查清单

刷写前：
- [ ] 备份原固件（如需要）
- [ ] 确认屏幕规格（8x32）
- [ ] 确认屏幕排列方式（列优先+顺序）
- [ ] 准备好 USB 数据线

刷写后：
- [ ] 设备正常启动
- [ ] 配置 `matrix: 3`
- [ ] 验证显示正常
- [ ] 连接 WiFi 成功
- [ ] Web 界面可访问
- [ ] 时间/日期显示正确

---

**文档版本**：1.0
**最后更新**：2025-01-04
**适用固件版本**：0.98（含 case 3 支持）
