# ADC模数转换

## ADC基本概念

ADC（Analog-to-Digital Converter）模数转换器是将模拟信号转换为数字信号的重要外设。

### ADC关键参数

| 参数 | 描述 | 影响 |
|------|------|------|
| 分辨率 | 转换精度（位数） | 精度越高，量化误差越小 |
| 采样率 | 每秒采样次数 | 决定信号带宽 |
| 转换时间 | 完成一次转换的时间 | 影响实时性 |
| 输入范围 | 可测量的电压范围 | 通常0-3.3V |

### STM32 ADC特性

- **分辨率**：12位（0-4095）
- **转换模式**：单次、连续、扫描、间断
- **触发源**：软件触发、定时器触发、外部触发
- **数据对齐**：右对齐、左对齐

## ADC初始化配置

### CubeMX配置ADC

1. 选择ADC外设和通道
2. 配置分辨率和采样时间
3. 设置转换模式
4. 配置触发源
5. 启用DMA/中断（可选）

### 手动配置ADC结构体

```c
ADC_HandleTypeDef hadc1;

void ADC1_Init(void)
{
    ADC_ChannelConfTypeDef sConfig = {0};

    hadc1.Instance = ADC1;
    hadc1.Init.ClockPrescaler = ADC_CLOCK_SYNC_PCLK_DIV4;
    hadc1.Init.Resolution = ADC_RESOLUTION_12B;
    hadc1.Init.DataAlign = ADC_DATAALIGN_RIGHT;
    hadc1.Init.ScanConvMode = DISABLE;
    hadc1.Init.EOCSelection = ADC_EOC_SINGLE_CONV;
    hadc1.Init.LowPowerAutoWait = DISABLE;
    hadc1.Init.ContinuousConvMode = DISABLE;
    hadc1.Init.NbrOfConversion = 1;
    hadc1.Init.DiscontinuousConvMode = DISABLE;
    hadc1.Init.ExternalTrigConv = ADC_SOFTWARE_START;
    hadc1.Init.ExternalTrigConvEdge = ADC_EXTERNALTRIGCONVEDGE_NONE;
    hadc1.Init.DMAContinuousRequests = DISABLE;
    hadc1.Init.Overrun = ADC_OVR_DATA_PRESERVED;
    hadc1.Init.SamplingTimeCommon1 = ADC_SAMPLETIME_1CYCLE_5;
    hadc1.Init.SamplingTimeCommon2 = ADC_SAMPLETIME_1CYCLE_5;
    hadc1.Init.OversamplingMode = DISABLE;
    hadc1.Init.TriggerFrequencyMode = ADC_TRIGGER_FREQ_HIGH;
    
    if (HAL_ADC_Init(&hadc1) != HAL_OK)
    {
        Error_Handler();
    }
    
    // 配置ADC通道
    sConfig.Channel = ADC_CHANNEL_0;      // PA0引脚
    sConfig.Rank = ADC_REGULAR_RANK_1;
    sConfig.SamplingTime = ADC_SAMPLINGTIME_COMMON_1;
    
    if (HAL_ADC_ConfigChannel(&hadc1, &sConfig) != HAL_OK)
    {
        Error_Handler();
    }
}
```

## ADC常用函数

### 基本转换函数

```c
// 启动ADC转换
HAL_ADC_Start(&hadc1);

// 轮询等待转换完成
if (HAL_ADC_PollForConversion(&hadc1, 100) == HAL_OK)
{
    // 读取转换结果
    uint32_t adc_value = HAL_ADC_GetValue(&hadc1);
}

// 停止ADC转换
HAL_ADC_Stop(&hadc1);

// 单次转换（启动+轮询+停止）
HAL_ADC_Start(&hadc1);
HAL_ADC_PollForConversion(&hadc1, 100);
uint32_t value = HAL_ADC_GetValue(&hadc1);
HAL_ADC_Stop(&hadc1);
```

### 中断方式转换

```c
// 启动ADC转换（中断）
HAL_ADC_Start_IT(&hadc1);

// 转换完成回调函数
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef* hadc)
{
    if (hadc->Instance == ADC1)
    {
        uint32_t adc_value = HAL_ADC_GetValue(hadc);
        // 处理转换结果
    }
}

// 转换半完成回调
void HAL_ADC_ConvHalfCpltCallback(ADC_HandleTypeDef* hadc)
{
    // 双缓冲区模式下，一半数据转换完成
}
```

### DMA方式转换

```c
uint16_t adc_buffer[100]; // DMA缓冲区

// 启动ADC转换（DMA）
HAL_ADC_Start_DMA(&hadc1, (uint32_t*)adc_buffer, 100);

// DMA转换完成回调
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef* hadc)
{
    // 所有数据转换完成
    for (int i = 0; i < 100; i++)
    {
        // 处理adc_buffer中的数据
    }
}
```

## 实战示例

### 示例1：单通道ADC采样

```c
#include "main.h"

ADC_HandleTypeDef hadc1;

// 将ADC值转换为电压（12位分辨率，3.3V参考电压）
float adc_to_voltage(uint32_t adc_value)
{
    return (adc_value * 3.3f) / 4095.0f;
}

int main(void)
{
    HAL_Init();
    SystemClock_Config();
    
    // 初始化ADC
    MX_ADC1_Init();
    
    // 初始化串口用于输出结果
    MX_USART1_UART_Init();
    
    while (1)
    {
        // 启动单次转换
        HAL_ADC_Start(&hadc1);
        
        // 等待转换完成
        if (HAL_ADC_PollForConversion(&hadc1, 100) == HAL_OK)
        {
            // 读取ADC值
            uint32_t adc_value = HAL_ADC_GetValue(&hadc1);
            
            // 转换为电压
            float voltage = adc_to_voltage(adc_value);
            
            // 通过串口输出（需要实现printf）
            // printf("ADC Value: %lu, Voltage: %.3f V\r\n", adc_value, voltage);
        }
        
        HAL_ADC_Stop(&hadc1);
        HAL_Delay(500); // 每500ms采样一次
    }
}
```

### 示例2：多通道ADC扫描

```c
#include "main.h"

ADC_HandleTypeDef hadc1;

// 定义多个ADC通道
#define ADC_CHANNEL_COUNT 3
const uint32_t adc_channels[ADC_CHANNEL_COUNT] = {
    ADC_CHANNEL_0,  // PA0
    ADC_CHANNEL_1,  // PA1
    ADC_CHANNEL_2   // PA2
};

uint16_t adc_results[ADC_CHANNEL_COUNT];

int main(void)
{
    HAL_Init();
    SystemClock_Config();
    
    // 配置ADC为多通道扫描模式
    hadc1.Instance = ADC1;
    hadc1.Init.ScanConvMode = ENABLE;           // 启用扫描模式
    hadc1.Init.ContinuousConvMode = ENABLE;     // 连续转换
    hadc1.Init.NbrOfConversion = ADC_CHANNEL_COUNT; // 转换通道数
    // ... 其他配置
    HAL_ADC_Init(&hadc1);
    
    // 配置所有通道
    ADC_ChannelConfTypeDef sConfig = {0};
    for (int i = 0; i < ADC_CHANNEL_COUNT; i++)
    {
        sConfig.Channel = adc_channels[i];
        sConfig.Rank = i + 1; // 通道排名
        sConfig.SamplingTime = ADC_SAMPLINGTIME_COMMON_1;
        HAL_ADC_ConfigChannel(&hadc1, &sConfig);
    }
    
    // 启动DMA连续转换
    HAL_ADC_Start_DMA(&hadc1, (uint32_t*)adc_results, ADC_CHANNEL_COUNT);
    
    while (1)
    {
        // 处理ADC数据（在DMA回调中处理更佳）
        for (int i = 0; i < ADC_CHANNEL_COUNT; i++)
        {
            float voltage = (adc_results[i] * 3.3f) / 4095.0f;
            // printf("Channel %d: %lu (%.3f V)\r\n", i, adc_results[i], voltage);
        }
        
        HAL_Delay(1000);
    }
}
```

### 示例3：定时器触发ADC采样

```c
#include "main.h"

ADC_HandleTypeDef hadc1;
TIM_HandleTypeDef htim2;

uint16_t adc_buffer[100];
uint32_t sample_count = 0;

// ADC转换完成回调
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef* hadc)
{
    if (hadc->Instance == ADC1)
    {
        sample_count++;
        
        // 每100次采样处理一次数据
        if (sample_count >= 100)
        {
            // 计算平均值
            uint32_t sum = 0;
            for (int i = 0; i < 100; i++)
            {
                sum += adc_buffer[i];
            }
            uint32_t average = sum / 100;
            
            // 输出平均值
            // printf("Average ADC: %lu\r\n", average);
            
            sample_count = 0;
        }
    }
}

int main(void)
{
    HAL_Init();
    SystemClock_Config();
    
    // 初始化定时器（1kHz采样率）
    MX_TIM2_Init();
    
    // 配置ADC为定时器触发
    hadc1.Instance = ADC1;
    hadc1.Init.ExternalTrigConv = ADC_EXTERNALTRIGCONV_T2_TRGO; // TIM2触发
    hadc1.Init.ExternalTrigConvEdge = ADC_EXTERNALTRIGCONVEDGE_RISING;
    hadc1.Init.ContinuousConvMode = DISABLE; // 外部触发时禁用连续模式
    // ... 其他配置
    HAL_ADC_Init(&hadc1);
    
    // 配置ADC通道
    ADC_ChannelConfTypeDef sConfig = {0};
    sConfig.Channel = ADC_CHANNEL_0;
    sConfig.Rank = ADC_REGULAR_RANK_1;
    sConfig.SamplingTime = ADC_SAMPLINGTIME_COMMON_1;
    HAL_ADC_ConfigChannel(&hadc1, &sConfig);
    
    // 启动DMA转换
    HAL_ADC_Start_DMA(&hadc1, (uint32_t*)adc_buffer, 100);
    
    // 启动定时器（触发ADC采样）
    HAL_TIM_Base_Start(&htim2);
    
    while (1)
    {
        // 主循环处理其他任务
        HAL_Delay(100);
    }
}
```

### 示例4：温度传感器读取

```c
#include "main.h"
#include <math.h>

ADC_HandleTypeDef hadc1;

// STM32内部温度传感器相关参数
#define V25        0.76f    // 25°C时的电压（V）
#define AVG_SLOPE  2.5f     // 平均斜率（mV/°C）
#define VREF       3.3f     // 参考电压（V）
#define ADC_MAX    4095.0f  // 12位ADC最大值

// 读取内部温度传感器
float read_internal_temperature(void)
{
    // 使能温度传感器
    __HAL_ADC_ENABLE(&hadc1);
    
    // 配置温度传感器通道
    ADC_ChannelConfTypeDef sConfig = {0};
    sConfig.Channel = ADC_CHANNEL_TEMPSENSOR;
    sConfig.Rank = ADC_REGULAR_RANK_1;
    sConfig.SamplingTime = ADC_SAMPLINGTIME_COMMON_1;
    HAL_ADC_ConfigChannel(&hadc1, &sConfig);
    
    // 启动转换并读取
    HAL_ADC_Start(&hadc1);
    HAL_ADC_PollForConversion(&hadc1, 100);
    uint32_t adc_value = HAL_ADC_GetValue(&hadc1);
    HAL_ADC_Stop(&hadc1);
    
    // 转换为电压
    float voltage = (adc_value * VREF) / ADC_MAX;
    
    // 计算温度
    float temperature = ((V25 - voltage) * 1000.0f / AVG_SLOPE) + 25.0f;
    
    return temperature;
}

int main(void)
{
    HAL_Init();
    SystemClock_Config();
    
    // 初始化ADC
    MX_ADC1_Init();
    
    while (1)
    {
        // 读取温度
        float temp = read_internal_temperature();
        
        // 输出温度值
        // printf("Temperature: %.1f°C\r\n", temp);
        
        HAL_Delay(2000); // 每2秒读取一次
    }
}
```

## 高级功能

### 1. 过采样提高分辨率

```c
// 配置ADC过采样（提高有效分辨率）
hadc1.Init.OversamplingMode = ENABLE;
hadc1.Init.Oversampling.Ratio = ADC_OVERSAMPLING_RATIO_8;    // 8倍过采样
hadc1.Init.Oversampling.RightBitShift = ADC_RIGHTBITSHIFT_3; // 右移3位
hadc1.Init.Oversampling.TriggeredMode = ADC_TRIGGEREDMODE_SINGLE_TRIGGER;

// 过采样后有效分辨率从12位提高到15位
```

### 2. 窗口看门狗比较

```c
// 配置ADC看门狗
ADC_AnalogWDGConfTypeDef AnalogWDGConfig = {0};

AnalogWDGConfig.WatchdogNumber = ADC_ANALOGWATCHDOG_1;
AnalogWDGConfig.WatchdogMode = ADC_ANALOGWATCHDOG_ALL_REG;
AnalogWDGConfig.HighThreshold = 3000;  // 高阈值
AnalogWDGConfig.LowThreshold = 1000;   // 低阈值
AnalogWDGConfig.Channel = ADC_CHANNEL_0;
AnalogWDGConfig.ITMode = ENABLE;

HAL_ADC_AnalogWDGConfig(&hadc1, &AnalogWDGConfig);

// 看门狗中断回调
void HAL_ADC_LevelOutOfWindowCallback(ADC_HandleTypeDef* hadc)
{
    // ADC值超出窗口范围
}
```

### 3. 差分输入模式

```c
// 配置差分输入（某些STM32型号支持）
ADC_ChannelConfTypeDef sConfig = {0};

sConfig.Channel = ADC_CHANNEL_0;
// sConfig.DifferentialMode = ENABLE;  // 启用差分模式
// sConfig.OffsetNumber = ADC_OFFSET_1; // 偏移校准
// sConfig.Offset = 0;                 // 偏移值

HAL_ADC_ConfigChannel(&hadc1, &sConfig);
```

## 校准和精度优化

### 1. ADC校准

```c
// 执行ADC校准
HAL_ADCEx_Calibration_Start(&hadc1, ADC_SINGLE_ENDED);

// 等待校准完成
while (HAL_ADCEx_Calibration_GetState(&hadc1) != HAL_ADC_CALIB_STATE_READY)
{
    // 等待校准完成
}
```

### 2. 软件滤波

```c
// 移动平均滤波
#define FILTER_SIZE 10
uint16_t adc_filter_buffer[FILTER_SIZE];
uint8_t filter_index = 0;

uint16_t moving_average_filter(uint16_t new_value)
{
    static uint32_t sum = 0;
    
    // 减去最旧的值，加上最新的值
    sum = sum - adc_filter_buffer[filter_index] + new_value;
    adc_filter_buffer[filter_index] = new_value;
    
    filter_index = (filter_index + 1) % FILTER_SIZE;
    
    return sum / FILTER_SIZE;
}
```

## 调试技巧

### 1. ADC性能测试

```c
// 测试ADC转换时间
void test_adc_speed(void)
{
    uint32_t start_time = HAL_GetTick();
    uint32_t sample_count = 0;
    
    while (HAL_GetTick() - start_time < 1000) // 测试1秒
    {
        HAL_ADC_Start(&hadc1);
        HAL_ADC_PollForConversion(&hadc1, 10);
        HAL_ADC_GetValue(&hadc1);
        HAL_ADC_Stop(&hadc1);
        sample_count++;
    }
    
    // printf("ADC采样率: %lu Hz\r\n", sample_count);
}
```

### 2. 线性度测试

```c
// 测试ADC线性度
void test_adc_linearity(void)
{
    uint32_t adc_values[10];
    
    for (int i = 0; i < 10; i++)
    {
        // 多次采样取平均
        uint32_t sum = 0;
        for (int j = 0; j < 10; j++)
        {
            HAL_ADC_Start(&hadc1);
            HAL_ADC_PollForConversion(&hadc1, 10);
            sum += HAL_ADC_GetValue(&hadc1);
            HAL_ADC_Stop(&hadc1);
        }
        adc_values[i] = sum / 10;
        
        HAL_Delay(100);
    }
    
    // 分析线性度
    for (int i = 0; i < 10; i++)
    {
        // printf("Sample %d: %lu\r\n", i, adc_values[i]);
    }
}
```

## 常见问题与解决方案

### 问题1：ADC采样值不稳定

**原因：** 电源噪声、接地问题、采样时间不足
**解决：** 增加采样时间、添加硬件滤波、改善电源质量

### 问题2：转换结果偏差大

**原因：** 参考电压不稳定、未进行校准
**解决：** 使用稳定的参考电压、执行ADC校准

### 问题3：DMA传输数据错误

**原因：** 内存对齐问题、缓冲区大小错误
**解决：** 确保缓冲区地址对齐、检查DMA配置

## 性能优化建议

1. **合理设置采样时间**：根据信号源阻抗设置合适的采样时间
2. **使用DMA**：连续采样时使用DMA减少CPU负担
3. **过采样技术**：需要更高分辨率时使用过采样
4. **硬件滤波**：在ADC输入端添加RC滤波电路

## 总结

ADC是STM32中重要的模拟信号采集外设，通过HAL库可以方便地实现各种ADC应用。掌握单通道、多通道、定时器触发等不同模式的使用方法，能够满足各种模拟信号采集需求。