# XiaoZhi ESP32 Project Usage Guide

## 项目概述

XiaoZhi ESP32 是一个基于 MCP (Model Context Protocol) 的 AI 语音聊天机器人固件项目,运行在 ESP32 系列芯片上,支持 70+ 种开发板。

## 快速开始

### 环境要求

- **操作系统**: Linux (推荐 WSL Ubuntu) / macOS
- **IDE**: VSCode / Cursor
- **ESP-IDF**: v5.4+
- **Python**: 3.8+
- **工具链**: clang-format (代码格式化)

### 安装 ESP-IDF

```bash
# 克隆 ESP-IDF
cd ~
git clone --recursive https://github.com/espressif/esp-idf.git
cd esp-idf
git checkout v5.4

# 安装工具链
./install.sh esp32,esp32s3,esp32c3

# 设置环境变量
. ~/esp-idf/export.sh
echo '. ~/esp-idf/export.sh' >> ~/.bashrc
```

### 克隆项目

```bash
git clone https://github.com/adlink8/xiaozhi-esp32.git
cd xiaozhi-esp32
```

### 构建和烧录

```bash
# 配置目标芯片 (ESP32-S3 示例)
idf.py set-target esp32s3

# 配置项目
idf.py menuconfig

# 编译
idf.py build

# 烧录 (USB 串口 /dev/ttyUSB0)
idf.py -p /dev/ttyUSB0 flash

# 查看日志
idf.py -p /dev/ttyUSB0 monitor
```

### 使用发布脚本编译指定开发板

```bash
# 编译立创 ESP32-S3 开发板固件
python scripts/release.py lichuang-s3

# 查看支持的开发板列表
ls main/boards/
```

## 项目结构

```
xiaozhi-esp32/
├── main/                      # 主源代码目录
│   ├── application.cc/h       # 应用程序主逻辑
│   ├── audio/                 # 音频服务 (录音/播放/编解码)
│   ├── boards/                # 70+ 开发板配置
│   ├── display/               # 显示驱动 (OLED/LCD)
│   ├── led/                   # LED 控制
│   ├── protocols/             # 通信协议 (WebSocket/MQTT+UDP)
│   ├── mcp_server.cc/h        # MCP 协议服务器
│   ├── ota.cc/h               # OTA 升级
│   └── main.cc                # 程序入口
├── docs/                      # 文档
│   ├── custom-board.md        # 自定义开发板指南
│   ├── mcp-usage.md           # MCP 协议物联网控制
│   ├── websocket.md           # WebSocket 协议文档
│   └── mqtt-udp.md            # MQTT+UDP 协议文档
├── partitions/                # Flash 分区表
│   └── v2/                    # v2 版本分区 (带 assets 分区)
├── scripts/                   # 构建脚本
│   └── release.py             # 发布脚本
├── sdkconfig.defaults         # SDK 默认配置
└── CMakeLists.txt             # CMake 构建文件
```

## 核心模块说明

### 1. Audio Service (音频服务)

位置: `main/audio/`

- **AudioCodec**: 音频编解码器硬件抽象层 (I2S)
- **AudioProcessor**: 音频处理 (AEC/降噪/VAD)
- **WakeWord**: 离线唤醒词检测
- **OpusEncoder/Decoder**: Opus 音频编解码

详见: [main/audio/README.md](main/audio/README.md)

### 2. Protocols (通信协议)

位置: `main/protocols/`

- **WebSocket**: 实时双向通信
- **MQTT+UDP**: MQTT 控制通道 + UDP 音频数据
- **MCP**: 设备控制协议 (JSON-RPC 2.0)

详见:
- [docs/websocket.md](docs/websocket.md)
- [docs/mqtt-udp.md](docs/mqtt-udp.md)
- [docs/mcp-protocol.md](docs/mcp-protocol.md)

### 3. Display (显示驱动)

位置: `main/display/`

支持多种显示屏:
- OLED (SSD1306, SH1106)
- LCD (ST7789, ILI9341, GC9A01)
- AMOLED

### 4. Board Support (开发板支持)

位置: `main/boards/`

每个开发板包含:
- `config.h`: 硬件引脚配置
- `config.json`: 编译配置
- `xxx_board.cc`: 板级初始化代码

详见: [docs/custom-board.md](docs/custom-board.md)

## 常用开发任务

### 添加新的开发板

```bash
# 1. 创建开发板目录
mkdir main/boards/my-board

# 2. 创建配置文件
touch main/boards/my-board/config.h
touch main/boards/my-board/config.json
touch main/boards/my-board/my_board.cc

# 3. 参考现有开发板配置
cat main/boards/lichuang-s3/config.h

# 4. 编译测试
python scripts/release.py my-board
```

详见: [docs/custom-board.md](docs/custom-board.md)

### 添加 MCP 工具 (设备控制)

```cpp
// 在开发板初始化代码中注册 MCP 工具
#include "mcp_server.h"

void InitializeTools() {
    auto& mcp = McpServer::GetInstance();
    
    // 添加无参数工具
    mcp.AddTool("device.led.on", "打开 LED", PropertyList(), 
        [](const PropertyList&) -> ReturnValue {
            gpio_set_level(LED_PIN, 1);
            return true;
        });
    
    // 添加带参数工具
    mcp.AddTool("device.led.brightness", "设置 LED 亮度", 
        PropertyList({Property("level", kPropertyTypeInteger, 0, 100)}),
        [](const PropertyList& params) -> ReturnValue {
            int level = params[0].GetInt();
            set_led_brightness(level);
            return level;
        });
}
```

详见: [docs/mcp-usage.md](docs/mcp-usage.md)

### 修改服务器地址

```bash
# 方法1: 通过 menuconfig
idf.py menuconfig
# 导航到: XiaoZhi Configuration -> Server Configuration

# 方法2: 直接修改 sdkconfig.defaults
vim sdkconfig.defaults
# 修改: CONFIG_SERVER_URL="ws://your-server-ip:8000/xiaozhi/v1/"
```

### 代码格式化

项目使用 Google C++ 代码风格:

```bash
# 格式化单个文件
clang-format -i main/application.cc

# 格式化所有 C++ 文件
find main -iname "*.h" -o -iname "*.cc" | xargs clang-format -i

# 检查格式 (不修改文件)
clang-format --dry-run -Werror main/application.cc
```

详见: [docs/code_style.md](docs/code_style.md)

## 调试技巧

### 查看日志

```bash
# 实时查看串口日志
idf.py monitor

# 过滤日志级别
idf.py monitor --print-filter "AUDIO:I"

# 保存日志到文件
idf.py monitor | tee debug.log
```

### 使用 JTAG 调试

```bash
# 启动 OpenOCD
openocd -f board/esp32s3-builtin.cfg

# 启动 GDB
xtensa-esp32s3-elf-gdb build/xiaozhi-esp32.elf
(gdb) target remote :3333
(gdb) monitor reset halt
(gdb) break application.cc:100
(gdb) continue
```

### 分析内存使用

```bash
# 查看编译后的大小
idf.py size

# 详细内存分析
idf.py size-components
idf.py size-files
```

## 配网方式

### BluFi (蓝牙配网)

```bash
# 1. 启用 BluFi
idf.py menuconfig
# WiFi Configuration Method -> Esp Blufi

# 2. 下载 EspBlufi App
# Android: https://github.com/EspressifApp/EspBlufiForAndroid/releases
# iOS: App Store 搜索 "EspBlufi"

# 3. 使用 App 连接设备并配置 WiFi
```

详见: [docs/blufi.md](docs/blufi.md)

### Hotspot (热点配网)

设备会创建热点 `XiaoZhi-XXXX`,连接后访问 `http://192.168.4.1` 配置 WiFi。

## 常见问题

### 编译错误

```bash
# 清理构建
idf.py fullclean

# 更新依赖
idf.py reconfigure

# 检查 ESP-IDF 版本
idf.py --version
```

### 烧录失败

```bash
# 进入下载模式: 按住 BOOT 键,再按 RESET 键

# 降低波特率
idf.py -p /dev/ttyUSB0 -b 115200 flash

# 擦除 Flash
esptool.py --port /dev/ttyUSB0 erase_flash
```

### 音频问题

- 检查 I2S 引脚配置是否正确
- 检查麦克风和扬声器接线
- 查看日志中的音频初始化信息
- 确认音频编解码器型号匹配

## 贡献指南

1. Fork 项目
2. 创建特性分支: `git checkout -b feature/new-feature`
3. 提交更改: `git commit -am 'Add new feature'`
4. 推送分支: `git push origin feature/new-feature`
5. 提交 Pull Request

**注意**:
- 代码必须通过 clang-format 格式化
- 遵循 Google C++ 代码风格
- 添加必要的注释和文档

## 资源链接

- [项目主页](https://github.com/78/xiaozhi-esp32)
- [在线文档](https://ccnphfhqs21z.feishu.cn/wiki/F5krwD16viZoF0kKkvDcrZNYnhb)
- [Discord 社区](https://discord.gg/bXqgAfRm)
- QQ 群: 994694848
