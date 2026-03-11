# Claude AI Coding Assistant Guide for XiaoZhi ESP32

本指南帮助 Claude AI 更好地理解和协助开发 XiaoZhi ESP32 项目。

## 项目上下文

### 项目类型
- **领域**: 嵌入式系统 / 物联网 / AI 语音交互
- **平台**: ESP32 系列 (ESP32-S3, ESP32-C3, ESP32-P4)
- **框架**: ESP-IDF v5.4+
- **语言**: C++ (Google 代码风格)
- **构建系统**: CMake
- **许可**: MIT

### 技术栈

**核心技术**:
- ESP-IDF (Espressif IoT Development Framework)
- FreeRTOS (实时操作系统)
- I2S (音频接口)
- WebSocket / MQTT+UDP (通信协议)
- Opus (音频编解码)

**外部依赖**:
- ESP-SR (语音识别)
- ESP-ADF (音频开发框架)
- cJSON (JSON 解析)
- mbedTLS (加密通信)

### 架构概览

```
┌─────────────────────────────────────────────────┐
│          Application Layer (application.cc)      │
├─────────────────────────────────────────────────┤
│  Audio Service  │  Protocol  │  Display  │  LED │
├─────────────────────────────────────────────────┤
│  MCP Server  │  OTA  │  Settings  │  Assets    │
├─────────────────────────────────────────────────┤
│          Board Abstraction Layer                 │
├─────────────────────────────────────────────────┤
│              ESP-IDF / FreeRTOS                  │
└─────────────────────────────────────────────────┘
```

## 代码风格指南

### C++ 代码约定

```cpp
// 1. 命名约定
class AudioService {};           // PascalCase for classes
void StartRecording();          // PascalCase for methods
int audio_sample_rate_;         // snake_case with trailing _ for private members
const int kMaxBufferSize = 512; // k prefix for constants

// 2. 文件组织
// header.h
#pragma once
class MyClass {
  public:
    void PublicMethod();
  
  private:
    void PrivateMethod();
    int member_variable_;
};

// implementation.cc
#include "header.h"
void MyClass::PublicMethod() {
    // Implementation
}

// 3. 缩进和格式
// - 4 空格缩进
// - 100 字符行宽限制
// - 大括号与语句同行
if (condition) {
    DoSomething();
} else {
    DoOtherthings();
}

// 4. 注释风格
// Brief single-line comment

/**
 * @brief Detailed multi-line comment
 * 
 * @param param1 Description of param1
 * @return Description of return value
 */
```

### ESP-IDF 特定约定

```cpp
// 1. 日志输出
#include "esp_log.h"
static const char* TAG = "MODULE_NAME";
ESP_LOGI(TAG, "Info message: %d", value);
ESP_LOGE(TAG, "Error message");

// 2. 错误处理
esp_err_t err = some_function();
if (err != ESP_OK) {
    ESP_LOGE(TAG, "Function failed: %s", esp_err_to_name(err));
    return err;
}

// 3. FreeRTOS 任务
void TaskFunction(void* param) {
    while (1) {
        // Task logic
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}

xTaskCreate(TaskFunction, "task_name", 4096, NULL, 5, NULL);

// 4. 内存管理
char* buffer = (char*)malloc(size);
if (buffer == NULL) {
    ESP_LOGE(TAG, "Memory allocation failed");
    return ESP_ERR_NO_MEM;
}
free(buffer);
```

## 常见开发任务

### 1. 添加新的音频处理功能

修改文件: `main/audio/audio_service.cc`

```cpp
// 示例: 添加音量控制
class AudioService {
  public:
    void SetVolume(int volume) {
        if (volume < 0 || volume > 100) {
            ESP_LOGW(TAG, "Invalid volume: %d", volume);
            return;
        }
        volume_ = volume;
        audio_codec_->SetOutputVolume(volume);
    }
    
  private:
    int volume_ = 50;
};
```

### 2. 添加新的 MCP 工具

修改文件: `main/boards/xxx/xxx_board.cc`

```cpp
#include "mcp_server.h"

void Board::InitializeMcpTools() {
    auto& mcp = McpServer::GetInstance();
    
    // 添加工具
    mcp.AddTool(
        "device.sensor.read_temperature",  // 工具名称
        "读取温度传感器数值",                // 描述
        PropertyList(),                     // 参数列表 (空)
        [this](const PropertyList& params) -> ReturnValue {
            float temp = ReadTemperature();
            return temp;
        }
    );
    
    // 带参数的工具
    mcp.AddTool(
        "device.motor.set_speed",
        "设置电机速度",
        PropertyList({
            Property("speed", kPropertyTypeInteger, 0, 100)
        }),
        [this](const PropertyList& params) -> ReturnValue {
            int speed = params[0].GetInt();
            SetMotorSpeed(speed);
            return true;
        }
    );
}
```

### 3. 添加新的开发板支持

步骤:
1. 创建目录: `main/boards/my-board/`
2. 创建 `config.h` (硬件配置)
3. 创建 `my_board.cc` (初始化代码)
4. 创建 `config.json` (构建配置)

示例 `config.h`:

```cpp
#pragma once
#include <driver/gpio.h>

// Audio configuration
#define AUDIO_INPUT_SAMPLE_RATE  16000
#define AUDIO_OUTPUT_SAMPLE_RATE 24000
#define AUDIO_I2S_GPIO_MCLK GPIO_NUM_10
#define AUDIO_I2S_GPIO_WS   GPIO_NUM_12
#define AUDIO_I2S_GPIO_BCLK GPIO_NUM_8
#define AUDIO_I2S_GPIO_DIN  GPIO_NUM_7
#define AUDIO_I2S_GPIO_DOUT GPIO_NUM_11

// Button configuration
#define BUTTON_GPIO GPIO_NUM_0
#define BUTTON_ACTIVE_LEVEL 0

// LED configuration
#define LED_GPIO GPIO_NUM_13
```

示例 `my_board.cc`:

```cpp
#include "board.h"
#include "config.h"
#include "esp_log.h"

static const char* TAG = "MyBoard";

class MyBoard : public Board {
  public:
    esp_err_t Initialize() override {
        ESP_LOGI(TAG, "Initializing MyBoard");
        
        // Initialize GPIO
        InitializeGpio();
        
        // Initialize Audio Codec
        InitializeAudioCodec();
        
        // Initialize Display
        InitializeDisplay();
        
        return ESP_OK;
    }
    
  private:
    void InitializeGpio() {
        gpio_config_t io_conf = {};
        io_conf.pin_bit_mask = (1ULL << LED_GPIO);
        io_conf.mode = GPIO_MODE_OUTPUT;
        gpio_config(&io_conf);
    }
};

extern "C" Board* CreateBoard() {
    return new MyBoard();
}
```

### 4. 修改通信协议

修改文件: `main/protocols/websocket_protocol.cc` 或 `main/protocols/mqtt_protocol.cc`

```cpp
// 示例: 添加新的消息类型
void WebsocketProtocol::HandleMessage(const char* data, size_t len) {
    cJSON* json = cJSON_Parse(data);
    if (json == NULL) {
        ESP_LOGE(TAG, "Failed to parse JSON");
        return;
    }
    
    cJSON* type = cJSON_GetObjectItem(json, "type");
    if (type && cJSON_IsString(type)) {
        if (strcmp(type->valuestring, "custom_message") == 0) {
            HandleCustomMessage(json);
        }
    }
    
    cJSON_Delete(json);
}

void WebsocketProtocol::HandleCustomMessage(cJSON* json) {
    // 处理自定义消息
    cJSON* payload = cJSON_GetObjectItem(json, "payload");
    if (payload) {
        // Process payload
    }
}
```

# XiaoZhi ESP32 Project - Cline AI Rules

## 项目上下文
project_type: "嵌入式 IoT 固件"
language: "C++"
framework: "ESP-IDF v5.4+"
platform: "ESP32系列 (S3/C3/P4)"
code_style: "Google C++ Style Guide"

## 代码风格规则
code_style:
  - 使用 clang-format 格式化所有 C++ 代码
  - 类名: PascalCase
  - 函数名: PascalCase
  - 成员变量: snake_case 加 _ 后缀
  - 常量: k 前缀 + PascalCase
  - 缩进: 4 空格
  - 行宽: 100 字符
  - 大括号: 与语句同行

## 文件组织
file_structure:
  - 头文件使用 #pragma once
  - 源文件与头文件分离
  - 每个类一个文件
  - 文件名小写,使用下划线分隔

## ESP-IDF 特定规则
esp_idf:
  logging:
    - 使用 ESP_LOGI/LOGW/LOGE/LOGD
    - 每个模块定义 static const char* TAG
    - 日志信息要清晰描述上下文
  
  error_handling:
    - 所有 ESP-IDF API 调用检查返回值
    - 使用 esp_err_to_name() 转换错误码
    - 失败时记录详细错误日志
  
  memory:
    - 检查所有 malloc/calloc 返回值
    - 及时 free 释放内存
    - 大数组使用堆分配,避免栈溢出
    - 使用 heap_caps_malloc 指定内存类型
  
  tasks:
    - 任务栈大小至少 4096 字节
    - 合理设置任务优先级 (1-24)
    - 使用 vTaskDelay 让出 CPU
    - 任务函数用 void* 参数

## 代码审查清单
review_checklist:
  - [ ] 代码格式符合 Google C++ 规范
  - [ ] 所有错误情况都有处理
  - [ ] 添加了适当的日志输出
  - [ ] 内存分配/释放成对出现
  - [ ] 没有魔数,使用宏或常量
  - [ ] 注释清晰描述复杂逻辑
  - [ ] 函数长度不超过 100 行
  - [ ] 避免全局变量
  - [ ] 线程安全(必要时使用互斥锁)

## 开发板适配规则
board_support:
  - 引脚配置写在 config.h 中
  - 不要硬编码引脚号
  - 使用 GPIO_NUM_XX 宏
  - 初始化顺序: GPIO → I2C → I2S → 外设

## MCP 工具开发
mcp_tools:
  - 工具名使用点分层次: "device.module.action"
  - 描述使用自然语言,便于 AI 理解
  - 参数类型明确(bool/int/string)
  - 返回值类型一致
  - 添加输入验证

## 性能优化
performance:
  - 音频处理使用 DMA
  - 避免在中断中执行复杂操作
  - 使用缓冲队列解耦任务
  - I2S 使用双缓冲
  - 显示更新使用 VSYNC

## 安全规则
security:
  - 敏感信息不写入日志
  - WebSocket 使用 wss://
  - MQTT 使用 TLS
  - 固件升级验证签名
  - 配置密码不明文存储

## 测试要求
testing:
  - 每个功能添加日志跟踪
  - 在真实硬件上测试
  - 测试内存泄漏 (长时间运行)
  - 测试断线重连
  - 测试电源复位

## 文档要求
documentation:
  - 公共 API 添加注释
  - 复杂算法添加说明
  - 配置选项写入 Kconfig
  - 新功能更新 README

## 禁止操作
forbidden:
  - 不要使用 C++ 异常
  - 不要使用 C++ RTTI
  - 不要使用 std::thread (使用 FreeRTOS 任务)
  - 不要在头文件中定义变量
  - 不要使用递归 (栈空间有限)

## 特定模块规则

### Audio 模块
audio:
  - 采样率必须匹配硬件
  - 使用 I2S DMA 模式
  - 缓冲区大小是 2 的幂
  - 及时消费音频数据避免溢出

### Display 模块
display:
  - 使用 DMA 传输大块数据
  - 避免频繁刷新全屏
  - 使用脏矩形优化
  - 注意 SPI 速度限制

### Protocol 模块
protocol:
  - 所有消息使用 JSON 格式
  - WebSocket 心跳 30 秒
  - 断线自动重连
  - 消息队列防止内存爆炸

## Git 提交规范
git_commit:
  format: "<type>(<scope>): <subject>"
  types:
    - feat: 新功能
    - fix: 修复 bug
    - docs: 文档更新
    - style: 代码格式
    - refactor: 重构
    - perf: 性能优化
    - test: 测试相关
    - chore: 构建/工具
  
  examples:
    - "feat(audio): add noise suppression"
    - "fix(mcp): handle empty tool response"
    - "docs(board): update pin configuration guide"

## 优先级
priority:
  - 稳定性 > 功能
  - 内存效率 > 性能
  - 代码清晰 > 简洁
  - 用户体验 > 实现复杂度

## 获取帮助
help:
  - 查看 docs/ 目录文档
  - 参考现有代码实现
  - 搜索 ESP-IDF 文档
  - 在 Discord/QQ 群提问
```