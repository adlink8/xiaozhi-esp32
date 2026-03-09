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