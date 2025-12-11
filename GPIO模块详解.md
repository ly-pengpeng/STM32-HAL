# GPIO模块详解

## GPIO基本概念

GPIO（General Purpose Input/Output）通用输入输出是STM32最基本的外设，用于控制数字信号的输入和输出。

### GPIO工作模式

| 模式 | 描述 | 应用场景 |
|------|------|----------|
| 输入浮空 | 高阻态，电平不确定 | 按键检测 |
| 输入上拉 | 内部上拉电阻 | 按键检测 |
| 输入下拉 | 内部下拉电阻 | 按键检测 |
| 模拟输入 | ADC采样输入 | 模拟信号采集 |
| 开漏输出 | 只能输出低电平 | I2C总线 |
| 推挽输出 | 输出高低电平 | LED控制 |
| 复用功能 | 外设专用引脚 | UART、SPI等 |

## GPIO初始化配置

### 使用CubeMX配置GPIO

1. 在Pinout界面选择引脚
2. 设置GPIO模式
3. 配置上拉/下拉电阻
4. 设置输出速度
5. 设置用户标签

### 手动配置GPIO结构体

```c
GPIO_InitTypeDef GPIO_InitStruct = {0};

// 使能GPIO时钟
__HAL_RCC_GPIOA_CLK_ENABLE();

// 配置GPIO引脚
GPIO_InitStruct.Pin = GPIO_PIN_5;           // PA5引脚
GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP; // 推挽输出
GPIO_InitStruct.Pull = GPIO_NOPULL;         // 无上拉下拉
GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;// 低速

HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
```

## GPIO常用函数

### 基本操作函数

```c
// 设置引脚为高电平
HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_SET);

// 设置引脚为低电平
HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_RESET);

// 切换引脚电平
HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);

// 读取引脚状态
GPIO_PinState state = HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_5);
```

### 位操作函数（高效）

```c
// 直接操作寄存器（更高效）
GPIOA->BSRR = GPIO_PIN_5;      // 置位（设置高电平）
GPIOA->BSRR = (uint32_t)GPIO_PIN_5 << 16; // 复位（设置低电平）

// 读取引脚状态
uint8_t pin_state = (GPIOA->IDR & GPIO_PIN_5) ? 1 : 0;
```

## 实战示例

### 示例1：LED闪烁

```c
#include "main.h"

// 定义LED引脚
#define LED_PIN GPIO_PIN_5
#define LED_PORT GPIOA

int main(void)
{
  HAL_Init();
  SystemClock_Config();
  
  // GPIO初始化
  GPIO_InitTypeDef GPIO_InitStruct = {0};
  __HAL_RCC_GPIOA_CLK_ENABLE();
  
  GPIO_InitStruct.Pin = LED_PIN;
  GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
  HAL_GPIO_Init(LED_PORT, &GPIO_InitStruct);
  
  while (1)
  {
    HAL_GPIO_TogglePin(LED_PORT, LED_PIN);
    HAL_Delay(500); // 500ms延时
  }
}
```

### 示例2：按键检测

```c
#include "main.h"

// 定义按键和LED引脚
#define BUTTON_PIN GPIO_PIN_0
#define BUTTON_PORT GPIOA
#define LED_PIN GPIO_PIN_5
#define LED_PORT GPIOA

int main(void)
{
  HAL_Init();
  SystemClock_Config();
  
  // 初始化LED
  GPIO_InitTypeDef GPIO_InitStruct = {0};
  __HAL_RCC_GPIOA_CLK_ENABLE();
  
  // LED配置
  GPIO_InitStruct.Pin = LED_PIN;
  GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
  HAL_GPIO_Init(LED_PORT, &GPIO_InitStruct);
  
  // 按键配置（上拉输入）
  GPIO_InitStruct.Pin = BUTTON_PIN;
  GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
  GPIO_InitStruct.Pull = GPIO_PULLUP;
  HAL_GPIO_Init(BUTTON_PORT, &GPIO_InitStruct);
  
  while (1)
  {
    // 检测按键按下（低电平有效）
    if (HAL_GPIO_ReadPin(BUTTON_PORT, BUTTON_PIN) == GPIO_PIN_RESET)
    {
      HAL_GPIO_WritePin(LED_PORT, LED_PIN, GPIO_PIN_SET); // LED亮
    }
    else
    {
      HAL_GPIO_WritePin(LED_PORT, LED_PIN, GPIO_PIN_RESET); // LED灭
    }
    
    HAL_Delay(10); // 10ms检测间隔
  }
}
```

### 示例3：多LED跑马灯

```c
#include "main.h"

// 定义LED引脚数组
#define LED_NUM 4
const uint16_t LED_PINS[LED_NUM] = {GPIO_PIN_5, GPIO_PIN_6, GPIO_PIN_7, GPIO_PIN_8};
#define LED_PORT GPIOA

int main(void)
{
  HAL_Init();
  SystemClock_Config();
  
  // GPIO初始化
  GPIO_InitTypeDef GPIO_InitStruct = {0};
  __HAL_RCC_GPIOA_CLK_ENABLE();
  
  GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
  
  // 初始化所有LED引脚
  for (int i = 0; i < LED_NUM; i++)
  {
    GPIO_InitStruct.Pin = LED_PINS[i];
    HAL_GPIO_Init(LED_PORT, &GPIO_InitStruct);
  }
  
  uint8_t current_led = 0;
  
  while (1)
  {
    // 关闭所有LED
    for (int i = 0; i < LED_NUM; i++)
    {
      HAL_GPIO_WritePin(LED_PORT, LED_PINS[i], GPIO_PIN_RESET);
    }
    
    // 点亮当前LED
    HAL_GPIO_WritePin(LED_PORT, LED_PINS[current_led], GPIO_PIN_SET);
    
    // 移动到下一个LED
    current_led = (current_led + 1) % LED_NUM;
    
    HAL_Delay(200); // 200ms间隔
  }
}
```

## GPIO高级应用

### 1. 引脚重映射

某些外设功能可以映射到不同的GPIO引脚：

```c
// 使能重映射时钟
__HAL_RCC_AFIO_CLK_ENABLE();

// 进行引脚重映射
__HAL_AFIO_REMAP_SWJ_DISABLE(); // 禁用SWD调试接口
```

### 2. 外部中断配置

GPIO引脚可以配置为外部中断源：

```c
// 配置GPIO为外部中断
GPIO_InitStruct.Pin = GPIO_PIN_0;
GPIO_InitStruct.Mode = GPIO_MODE_IT_RISING; // 上升沿触发
GPIO_InitStruct.Pull = GPIO_NOPULL;
HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

// 配置NVIC（嵌套向量中断控制器）
HAL_NVIC_SetPriority(EXTI0_IRQn, 0, 0);
HAL_NVIC_EnableIRQ(EXTI0_IRQn);
```

### 3. 模拟输入配置（ADC）

```c
// 配置GPIO为模拟输入
GPIO_InitStruct.Pin = GPIO_PIN_0;
GPIO_InitStruct.Mode = GPIO_MODE_ANALOG;
GPIO_InitStruct.Pull = GPIO_NOPULL;
HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
```

## 常见问题与调试

### 问题1：GPIO无法正常工作

**可能原因：**
- 时钟未使能
- 引脚模式配置错误
- 硬件连接问题

**解决方法：**
```c
// 检查时钟是否使能
if (__HAL_RCC_GPIOA_IS_CLK_ENABLED())
{
    // 时钟已使能
}
```

### 问题2：电平读取不稳定

**解决方法：**
- 添加软件消抖
- 使用硬件滤波
- 检查上拉/下拉电阻配置

```c
// 软件消抖示例
uint8_t debounce_read(GPIO_TypeDef* port, uint16_t pin)
{
    uint8_t stable_count = 0;
    GPIO_PinState last_state = HAL_GPIO_ReadPin(port, pin);
    
    for (int i = 0; i < 10; i++)
    {
        HAL_Delay(1);
        GPIO_PinState current_state = HAL_GPIO_ReadPin(port, pin);
        
        if (current_state == last_state)
        {
            stable_count++;
        }
        else
        {
            stable_count = 0;
            last_state = current_state;
        }
        
        if (stable_count >= 5) // 连续5次稳定
        {
            break;
        }
    }
    
    return (last_state == GPIO_PIN_SET) ? 1 : 0;
}
```

## 性能优化建议

1. **使用位带操作**：对于频繁操作的GPIO，使用位带区域提高效率
2. **批量操作**：同时操作多个引脚时使用BSRR寄存器
3. **合理设置速度**：根据实际需求选择GPIO速度等级
4. **避免频繁模式切换**：减少GPIO模式切换次数

## 总结

GPIO是STM32最基础也是最重要的外设，掌握GPIO的各种工作模式和配置方法对于嵌入式开发至关重要。通过HAL库提供的统一接口，可以简化GPIO的操作，提高代码的可移植性和可维护性。