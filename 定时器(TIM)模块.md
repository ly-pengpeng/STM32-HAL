# 定时器(TIM)模块

## 定时器基本概念

STM32的定时器是功能强大的外设，可用于定时、计数、PWM生成、输入捕获等多种应用。

### 定时器分类

| 定时器类型 | 主要功能 | 典型应用 |
|-----------|---------|---------|
| 基本定时器 | 基本定时 | 系统滴答、延时 |
| 通用定时器 | 定时、PWM、输入捕获 | 电机控制、信号测量 |
| 高级定时器 | 复杂PWM、死区控制 | 电机驱动、电源管理 |

### 定时器关键参数

- **时钟源**：内部时钟、外部时钟等
- **预分频器**：降低计数频率
- **自动重装载值**：定时周期
- **计数模式**：向上、向下、中央对齐

## 定时器初始化配置

### CubeMX配置定时器

1. 选择TIM外设
2. 配置时钟源和分频
3. 设置计数模式和周期
4. 配置通道功能（PWM/输入捕获等）
5. 启用中断（可选）

### 手动配置定时器结构体

```c
TIM_HandleTypeDef htim2;

void TIM2_Init(void)
{
    TIM_ClockConfigTypeDef sClockSourceConfig = {0};
    TIM_MasterConfigTypeDef sMasterConfig = {0};

    htim2.Instance = TIM2;
    htim2.Init.Prescaler = 8399;          // 预分频值
    htim2.Init.CounterMode = TIM_COUNTERMODE_UP; // 向上计数
    htim2.Init.Period = 9999;             // 自动重装载值
    htim2.Init.ClockDivision = TIM_CLOCKDIVISION_DIV1;
    htim2.Init.AutoReloadPreload = TIM_AUTORELOAD_PRELOAD_DISABLE;
    
    if (HAL_TIM_Base_Init(&htim2) != HAL_OK)
    {
        Error_Handler();
    }
    
    // 配置时钟源
    sClockSourceConfig.ClockSource = TIM_CLOCKSOURCE_INTERNAL;
    HAL_TIM_ConfigClockSource(&htim2, &sClockSourceConfig);
    
    // 配置主模式
    sMasterConfig.MasterOutputTrigger = TIM_TRGO_RESET;
    sMasterConfig.MasterSlaveMode = TIM_MASTERSLAVEMODE_DISABLE;
    HAL_TIMEx_MasterConfigSynchronization(&htim2, &sMasterConfig);
}
```

## 定时器常用函数

### 基础定时功能

```c
// 启动定时器
HAL_TIM_Base_Start(&htim2);

// 停止定时器
HAL_TIM_Base_Stop(&htim2);

// 启动定时器中断
HAL_TIM_Base_Start_IT(&htim2);

// 启动定时器DMA
HAL_TIM_Base_Start_DMA(&htim2);

// 获取计数器值
uint32_t count = __HAL_TIM_GET_COUNTER(&htim2);

// 设置计数器值
__HAL_TIM_SET_COUNTER(&htim2, 0);
```

### PWM功能函数

```c
// 启动PWM输出
HAL_TIM_PWM_Start(&htim2, TIM_CHANNEL_1);

// 停止PWM输出
HAL_TIM_PWM_Stop(&htim2, TIM_CHANNEL_1);

// 设置PWM占空比
__HAL_TIM_SET_COMPARE(&htim2, TIM_CHANNEL_1, 500); // 设置比较值

// 获取当前占空比
uint32_t duty = __HAL_TIM_GET_COMPARE(&htim2, TIM_CHANNEL_1);
```

### 输入捕获函数

```c
// 启动输入捕获
HAL_TIM_IC_Start(&htim2, TIM_CHANNEL_1);

// 启动输入捕获中断
HAL_TIM_IC_Start_IT(&htim2, TIM_CHANNEL_1);

// 获取捕获值
uint32_t capture_value = HAL_TIM_ReadCapturedValue(&htim2, TIM_CHANNEL_1);
```

## 实战示例

### 示例1：基本定时器中断

```c
#include "main.h"

TIM_HandleTypeDef htim2;

// 定时器中断回调函数
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
    if (htim->Instance == TIM2)
    {
        // 每1秒执行一次
        HAL_GPIO_TogglePin(LED_GPIO_Port, LED_Pin);
    }
}

int main(void)
{
    HAL_Init();
    SystemClock_Config();
    
    // 初始化GPIO和定时器
    MX_GPIO_Init();
    MX_TIM2_Init();
    
    // 计算定时器参数（1秒中断）
    // 系统时钟84MHz，预分频8399，自动重装载9999
    // 定时频率 = 84MHz / (8399+1) = 10kHz
    // 中断周期 = 10000 / 10kHz = 1秒
    
    // 启动定时器中断
    HAL_TIM_Base_Start_IT(&htim2);
    
    while (1)
    {
        // 主循环处理其他任务
        HAL_Delay(100);
    }
}
```

### 示例2：PWM呼吸灯

```c
#include "main.h"

TIM_HandleTypeDef htim3;

int main(void)
{
    HAL_Init();
    SystemClock_Config();
    
    // 初始化GPIO和定时器
    MX_GPIO_Init();
    MX_TIM3_Init();
    
    // 启动PWM输出
    HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_1);
    
    uint16_t pwm_value = 0;
    int8_t direction = 1; // 1:增加, -1:减少
    
    while (1)
    {
        // 设置PWM占空比
        __HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, pwm_value);
        
        // 更新PWM值
        pwm_value += direction * 10;
        
        // 改变方向
        if (pwm_value >= 1000)
        {
            direction = -1;
        }
        else if (pwm_value <= 0)
        {
            direction = 1;
        }
        
        HAL_Delay(10); // 10ms延时
    }
}
```

### 示例3：输入捕获测量脉冲宽度

```c
#include "main.h"

TIM_HandleTypeDef htim4;

uint32_t capture_start = 0;
uint32_t capture_end = 0;
uint32_t pulse_width = 0;
uint8_t capture_state = 0; // 0:等待上升沿, 1:等待下降沿

// 输入捕获回调函数
void HAL_TIM_IC_CaptureCallback(TIM_HandleTypeDef *htim)
{
    if (htim->Instance == TIM4)
    {
        if (capture_state == 0)
        {
            // 捕获到上升沿
            capture_start = HAL_TIM_ReadCapturedValue(htim, TIM_CHANNEL_1);
            
            // 切换为下降沿捕获
            __HAL_TIM_SET_CAPTUREPOLARITY(htim, TIM_CHANNEL_1, TIM_INPUTCHANNELPOLARITY_FALLING);
            capture_state = 1;
        }
        else
        {
            // 捕获到下降沿
            capture_end = HAL_TIM_ReadCapturedValue(htim, TIM_CHANNEL_1);
            
            // 计算脉冲宽度
            if (capture_end > capture_start)
            {
                pulse_width = capture_end - capture_start;
            }
            else
            {
                pulse_width = (0xFFFF - capture_start) + capture_end + 1;
            }
            
            // 切换回上升沿捕获
            __HAL_TIM_SET_CAPTUREPOLARITY(htim, TIM_CHANNEL_1, TIM_INPUTCHANNELPOLARITY_RISING);
            capture_state = 0;
        }
    }
}

int main(void)
{
    HAL_Init();
    SystemClock_Config();
    
    // 初始化GPIO和定时器
    MX_GPIO_Init();
    MX_TIM4_Init();
    
    // 启动输入捕获中断
    HAL_TIM_IC_Start_IT(&htim4, TIM_CHANNEL_1);
    
    while (1)
    {
        // 显示脉冲宽度（通过串口）
        if (pulse_width > 0)
        {
            // 转换为微秒（假设定时器时钟为1MHz）
            uint32_t width_us = pulse_width;
            
            // 通过串口输出（需要实现串口功能）
            // printf("Pulse Width: %lu us\r\n", width_us);
            
            pulse_width = 0; // 重置
        }
        
        HAL_Delay(100);
    }
}
```

### 示例4：多通道PWM控制

```c
#include "main.h"

TIM_HandleTypeDef htim1;

// PWM通道定义
#define PWM_CHANNELS 4
const uint32_t pwm_channels[PWM_CHANNELS] = {
    TIM_CHANNEL_1, TIM_CHANNEL_2, TIM_CHANNEL_3, TIM_CHANNEL_4
};

uint16_t pwm_values[PWM_CHANNELS] = {0, 250, 500, 750};
int8_t directions[PWM_CHANNELS] = {1, 1, 1, 1};

int main(void)
{
    HAL_Init();
    SystemClock_Config();
    
    // 初始化GPIO和高级定时器
    MX_GPIO_Init();
    MX_TIM1_Init();
    
    // 启动所有PWM通道
    for (int i = 0; i < PWM_CHANNELS; i++)
    {
        HAL_TIM_PWM_Start(&htim1, pwm_channels[i]);
    }
    
    while (1)
    {
        // 更新所有通道的PWM值
        for (int i = 0; i < PWM_CHANNELS; i++)
        {
            // 设置PWM值
            __HAL_TIM_SET_COMPARE(&htim1, pwm_channels[i], pwm_values[i]);
            
            // 更新PWM值
            pwm_values[i] += directions[i] * 5;
            
            // 改变方向
            if (pwm_values[i] >= 1000)
            {
                directions[i] = -1;
            }
            else if (pwm_values[i] <= 0)
            {
                directions[i] = 1;
            }
        }
        
        HAL_Delay(20); // 20ms延时
    }
}
```

## 高级定时器功能

### 1. 互补输出和死区控制

```c
// 高级定时器互补PWM配置
TIM_OC_InitTypeDef sConfigOC = {0};
TIM_BreakDeadTimeConfigTypeDef sBreakDeadTimeConfig = {0};

// 配置输出比较
sConfigOC.OCMode = TIM_OCMODE_PWM1;
sConfigOC.Pulse = 500;
sConfigOC.OCPolarity = TIM_OCPOLARITY_HIGH;
sConfigOC.OCNPolarity = TIM_OCNPOLARITY_HIGH;
sConfigOC.OCFastMode = TIM_OCFAST_DISABLE;
sConfigOC.OCIdleState = TIM_OCIDLESTATE_RESET;
sConfigOC.OCNIdleState = TIM_OCNIDLESTATE_RESET;
HAL_TIM_PWM_ConfigChannel(&htim1, &sConfigOC, TIM_CHANNEL_1);

// 配置死区时间
sBreakDeadTimeConfig.OffStateRunMode = TIM_OSSR_DISABLE;
sBreakDeadTimeConfig.OffStateIDLEMode = TIM_OSSI_DISABLE;
sBreakDeadTimeConfig.LockLevel = TIM_LOCKLEVEL_OFF;
sBreakDeadTimeConfig.DeadTime = 100; // 死区时间
sBreakDeadTimeConfig.BreakState = TIM_BREAK_DISABLE;
sBreakDeadTimeConfig.BreakPolarity = TIM_BREAKPOLARITY_HIGH;
sBreakDeadTimeConfig.AutomaticOutput = TIM_AUTOMATICOUTPUT_DISABLE;
HAL_TIMEx_ConfigBreakDeadTime(&htim1, &sBreakDeadTimeConfig);
```

### 2. 编码器模式

```c
// 配置定时器为编码器模式
TIM_Encoder_InitTypeDef sConfig = {0};
TIM_MasterConfigTypeDef sMasterConfig = {0};

sConfig.EncoderMode = TIM_ENCODERMODE_TI12;
sConfig.IC1Polarity = TIM_ICPOLARITY_RISING;
sConfig.IC1Selection = TIM_ICSELECTION_DIRECTTI;
sConfig.IC1Prescaler = TIM_ICPSC_DIV1;
sConfig.IC1Filter = 0;
sConfig.IC2Polarity = TIM_ICPOLARITY_RISING;
sConfig.IC2Selection = TIM_ICSELECTION_DIRECTTI;
sConfig.IC2Prescaler = TIM_ICPSC_DIV1;
sConfig.IC2Filter = 0;
HAL_TIM_Encoder_Init(&htim2, &sConfig);

// 启动编码器接口
HAL_TIM_Encoder_Start(&htim2, TIM_CHANNEL_ALL);

// 读取编码器值
int32_t encoder_value = (int32_t)__HAL_TIM_GET_COUNTER(&htim2);
```

### 3. 定时器级联

```c
// 配置主从定时器
TIM_SlaveConfigTypeDef sSlaveConfig = {0};
TIM_MasterConfigTypeDef sMasterConfig = {0};

// 主定时器配置
sMasterConfig.MasterOutputTrigger = TIM_TRGO_UPDATE;
sMasterConfig.MasterSlaveMode = TIM_MASTERSLAVEMODE_ENABLE;
HAL_TIMEx_MasterConfigSynchronization(&htim1, &sMasterConfig);

// 从定时器配置
sSlaveConfig.SlaveMode = TIM_SLAVEMODE_EXTERNAL1;
sSlaveConfig.InputTrigger = TIM_TS_ITR0; // 连接到TIM1
HAL_TIM_SlaveConfigSynchronization(&htim2, &sSlaveConfig);
```

## 调试技巧

### 1. 定时器频率计算

```c
// 计算定时器频率和周期
uint32_t timer_freq = HAL_RCC_GetPCLK1Freq() * 2; // APB1定时器时钟
uint32_t prescaler = 8399;
uint32_t period = 9999;

uint32_t counter_freq = timer_freq / (prescaler + 1);
uint32_t interrupt_freq = counter_freq / (period + 1);
float interrupt_period = 1.0f / interrupt_freq;

printf("Timer Frequency: %lu Hz\r\n", counter_freq);
printf("Interrupt Frequency: %lu Hz\r\n", interrupt_freq);
printf("Interrupt Period: %.3f s\r\n", interrupt_period);
```

### 2. PWM占空比计算

```c
// 计算PWM占空比
uint32_t period_value = __HAL_TIM_GET_AUTORELOAD(&htim3);
uint32_t compare_value = __HAL_TIM_GET_COMPARE(&htim3, TIM_CHANNEL_1);
float duty_cycle = (float)compare_value / (period_value + 1) * 100.0f;

printf("PWM Duty Cycle: %.1f%%\r\n", duty_cycle);
```

## 常见问题与解决方案

### 问题1：定时器不工作

**原因：** 时钟未使能、配置错误
**解决：** 检查时钟配置，验证预分频和周期值

### 问题2：PWM输出异常

**原因：** GPIO模式配置错误、定时器通道未启用
**解决：** 检查GPIO复用功能，确认通道配置正确

### 问题3：中断不触发

**原因：** NVIC配置错误、中断优先级过低
**解决：** 检查NVIC配置，提高中断优先级

## 性能优化建议

1. **合理选择预分频**：根据需求平衡精度和范围
2. **使用DMA**：大数据传输时使用DMA减少CPU负担
3. **优化中断处理**：中断函数尽量简短高效
4. **利用高级功能**：根据应用需求使用编码器、互补输出等高级功能

## 总结

STM32的定时器功能非常强大，通过HAL库可以方便地实现各种定时、PWM、输入捕获等应用。掌握定时器的配置和使用方法，对于嵌入式系统开发至关重要。