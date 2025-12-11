# 中断和DMA配置

## 中断系统

### 中断基本概念

中断是CPU响应外部或内部事件的一种机制，允许CPU暂停当前任务，处理紧急事件，然后返回原任务。

#### STM32中断类型

| 中断类型 | 描述 | 应用场景 |
|----------|------|----------|
| 外部中断 | GPIO引脚电平变化 | 按键检测、外部事件 |
| 定时器中断 | 定时器溢出/比较匹配 | 定时任务、PWM生成 |
| 串口中断 | 数据收发完成 | 异步通信 |
| DMA中断 | DMA传输完成 | 大数据传输 |
| 系统异常 | 硬件错误、系统调用 | 错误处理 |

### NVIC（嵌套向量中断控制器）

NVIC负责管理所有中断的优先级和使能状态。

#### NVIC配置结构体

```c
// NVIC优先级分组配置
HAL_NVIC_SetPriorityGrouping(NVIC_PRIORITYGROUP_4);

// 配置具体中断
HAL_NVIC_SetPriority(EXTI0_IRQn, 0, 0);  // 设置优先级
HAL_NVIC_EnableIRQ(EXTI0_IRQn);          // 使能中断

// 禁用中断
HAL_NVIC_DisableIRQ(EXTI0_IRQn);

// 清除中断挂起标志
HAL_NVIC_ClearPendingIRQ(EXTI0_IRQn);
```

#### 中断优先级分组

| 分组 | 抢占优先级位数 | 子优先级位数 | 总优先级数 |
|------|---------------|-------------|-----------|
| NVIC_PRIORITYGROUP_0 | 0位 | 4位 | 16级 |
| NVIC_PRIORITYGROUP_1 | 1位 | 3位 | 8级 |
| NVIC_PRIORITYGROUP_2 | 2位 | 2位 | 4级 |
| NVIC_PRIORITYGROUP_3 | 3位 | 1位 | 2级 |
| NVIC_PRIORITYGROUP_4 | 4位 | 0位 | 16级 |

### 外部中断配置

#### GPIO外部中断配置

```c
// 配置GPIO为外部中断
GPIO_InitTypeDef GPIO_InitStruct = {0};

GPIO_InitStruct.Pin = GPIO_PIN_0;
GPIO_InitStruct.Mode = GPIO_MODE_IT_RISING;  // 上升沿触发
GPIO_InitStruct.Pull = GPIO_NOPULL;
HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

// 配置NVIC
HAL_NVIC_SetPriority(EXTI0_IRQn, 0, 0);
HAL_NVIC_EnableIRQ(EXTI0_IRQn);

// 外部中断处理函数（在stm32xx_it.c中）
void EXTI0_IRQHandler(void)
{
    HAL_GPIO_EXTI_IRQHandler(GPIO_PIN_0);
}

// 回调函数（在用户代码中）
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    if (GPIO_Pin == GPIO_PIN_0)
    {
        // 处理PA0引脚的中断
        HAL_GPIO_TogglePin(LED_GPIO_Port, LED_Pin);
    }
}
```

#### 多引脚外部中断

```c
// 配置多个GPIO中断
void GPIO_Interrupt_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStruct = {0};
    
    // 配置PA0（上升沿触发）
    GPIO_InitStruct.Pin = GPIO_PIN_0;
    GPIO_InitStruct.Mode = GPIO_MODE_IT_RISING;
    GPIO_InitStruct.Pull = GPIO_NOPULL;
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
    
    // 配置PA1（下降沿触发）
    GPIO_InitStruct.Pin = GPIO_PIN_1;
    GPIO_InitStruct.Mode = GPIO_MODE_IT_FALLING;
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
    
    // 配置NVIC
    HAL_NVIC_SetPriority(EXTI0_IRQn, 0, 0);
    HAL_NVIC_SetPriority(EXTI1_IRQn, 1, 0);
    HAL_NVIC_EnableIRQ(EXTI0_IRQn);
    HAL_NVIC_EnableIRQ(EXTI1_IRQn);
}

// 中断回调函数
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    switch (GPIO_Pin)
    {
        case GPIO_PIN_0:
            // PA0中断处理
            break;
        case GPIO_PIN_1:
            // PA1中断处理
            break;
        default:
            break;
    }
}
```

### 串口中断配置

#### 中断方式串口通信

```c
UART_HandleTypeDef huart1;
uint8_t rx_buffer[100];
uint8_t rx_index = 0;

// 初始化串口中断
void UART_Interrupt_Init(void)
{
    // 配置UART（略）
    
    // 启动中断接收
    HAL_UART_Receive_IT(&huart1, &rx_buffer[rx_index], 1);
    
    // 配置UART中断优先级
    HAL_NVIC_SetPriority(USART1_IRQn, 0, 0);
    HAL_NVIC_EnableIRQ(USART1_IRQn);
}

// 接收完成回调
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1)
    {
        // 处理接收到的数据
        if (rx_buffer[rx_index] == '\r' || rx_buffer[rx_index] == '\n')
        {
            // 收到完整命令
            if (rx_index > 0)
            {
                // 处理命令
                process_command(rx_buffer, rx_index);
                rx_index = 0;
            }
        }
        else
        {
            rx_index++;
            if (rx_index >= sizeof(rx_buffer))
            {
                rx_index = 0; // 防止缓冲区溢出
            }
        }
        
        // 重新启动接收
        HAL_UART_Receive_IT(&huart1, &rx_buffer[rx_index], 1);
    }
}

// 发送完成回调
void HAL_UART_TxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1)
    {
        // 发送完成处理
    }
}
```

### 定时器中断配置

#### 定时器更新中断

```c
TIM_HandleTypeDef htim2;

// 初始化定时器中断
void TIM_Interrupt_Init(void)
{
    // 配置定时器（1秒中断）
    htim2.Instance = TIM2;
    htim2.Init.Prescaler = 8399;
    htim2.Init.Period = 9999;
    // ... 其他配置
    HAL_TIM_Base_Init(&htim2);
    
    // 启动定时器中断
    HAL_TIM_Base_Start_IT(&htim2);
    
    // 配置定时器中断优先级
    HAL_NVIC_SetPriority(TIM2_IRQn, 1, 0);
    HAL_NVIC_EnableIRQ(TIM2_IRQn);
}

// 定时器中断回调
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
    if (htim->Instance == TIM2)
    {
        // 每秒执行的任务
        HAL_GPIO_TogglePin(LED_GPIO_Port, LED_Pin);
    }
}
```

#### 定时器比较中断

```c
// 配置定时器比较中断
void TIM_Compare_Init(void)
{
    TIM_OC_InitTypeDef sConfigOC = {0};
    
    // 配置比较通道
    sConfigOC.OCMode = TIM_OCMODE_TIMING;
    sConfigOC.Pulse = 5000; // 比较值
    sConfigOC.OCPolarity = TIM_OCPOLARITY_HIGH;
    sConfigOC.OCFastMode = TIM_OCFAST_DISABLE;
    HAL_TIM_OC_ConfigChannel(&htim2, &sConfigOC, TIM_CHANNEL_1);
    
    // 启动比较中断
    HAL_TIM_OC_Start_IT(&htim2, TIM_CHANNEL_1);
    
    // 配置中断优先级
    HAL_NVIC_SetPriority(TIM2_IRQn, 1, 0);
    HAL_NVIC_EnableIRQ(TIM2_IRQn);
}

// 比较中断回调
void HAL_TIM_OC_DelayElapsedCallback(TIM_HandleTypeDef *htim)
{
    if (htim->Instance == TIM2)
    {
        // 比较匹配处理
    }
}
```

## DMA（直接存储器访问）

### DMA基本概念

DMA允许外设直接与存储器进行数据传输，无需CPU介入，大大提高数据传输效率。

#### DMA传输模式

| 传输模式 | 描述 | 应用场景 |
|----------|------|----------|
| 存储器到外设 | 内存数据发送到外设 | UART发送、SPI发送 |
| 外设到存储器 | 外设数据读取到内存 | ADC采样、UART接收 |
| 存储器到存储器 | 内存块之间传输 | 数据搬移、缓冲区操作 |

### DMA初始化配置

#### DMA结构体配置

```c
DMA_HandleTypeDef hdma_usart1_tx;
DMA_HandleTypeDef hdma_usart1_rx;

// 配置DMA发送
void DMA_TX_Init(void)
{
    // DMA时钟使能
    __HAL_RCC_DMA1_CLK_ENABLE();
    
    // 配置DMA发送
    hdma_usart1_tx.Instance = DMA1_Channel4;
    hdma_usart1_tx.Init.Direction = DMA_MEMORY_TO_PERIPH;
    hdma_usart1_tx.Init.PeriphInc = DMA_PINC_DISABLE;
    hdma_usart1_tx.Init.MemInc = DMA_MINC_ENABLE;
    hdma_usart1_tx.Init.PeriphDataAlignment = DMA_PDATAALIGN_BYTE;
    hdma_usart1_tx.Init.MemDataAlignment = DMA_MDATAALIGN_BYTE;
    hdma_usart1_tx.Init.Mode = DMA_NORMAL;
    hdma_usart1_tx.Init.Priority = DMA_PRIORITY_LOW;
    
    HAL_DMA_Init(&hdma_usart1_tx);
    
    // 关联DMA到UART
    __HAL_LINKDMA(&huart1, hdmatx, hdma_usart1_tx);
    
    // 配置DMA中断
    HAL_NVIC_SetPriority(DMA1_Channel4_IRQn, 0, 0);
    HAL_NVIC_EnableIRQ(DMA1_Channel4_IRQn);
}

// 配置DMA接收
void DMA_RX_Init(void)
{
    hdma_usart1_rx.Instance = DMA1_Channel5;
    hdma_usart1_rx.Init.Direction = DMA_PERIPH_TO_MEMORY;
    hdma_usart1_rx.Init.PeriphInc = DMA_PINC_DISABLE;
    hdma_usart1_rx.Init.MemInc = DMA_MINC_ENABLE;
    hdma_usart1_rx.Init.PeriphDataAlignment = DMA_PDATAALIGN_BYTE;
    hdma_usart1_rx.Init.MemDataAlignment = DMA_MDATAALIGN_BYTE;
    hdma_usart1_rx.Init.Mode = DMA_CIRCULAR; // 循环模式
    hdma_usart1_rx.Init.Priority = DMA_PRIORITY_HIGH;
    
    HAL_DMA_Init(&hdma_usart1_rx);
    
    // 关联DMA到UART
    __HAL_LINKDMA(&huart1, hdmarx, hdma_usart1_rx);
    
    // 配置DMA中断
    HAL_NVIC_SetPriority(DMA1_Channel5_IRQn, 0, 0);
    HAL_NVIC_EnableIRQ(DMA1_Channel5_IRQn);
}
```

### DMA实战示例

#### 示例1：UART DMA通信

```c
UART_HandleTypeDef huart1;
DMA_HandleTypeDef hdma_usart1_rx;
DMA_HandleTypeDef hdma_usart1_tx;

uint8_t dma_rx_buffer[100];
uint8_t dma_tx_buffer[100];

// DMA接收完成回调
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1)
    {
        // 处理接收到的数据
        process_received_data(dma_rx_buffer, 100);
        
        // 重新启动DMA接收（循环模式会自动重启）
        // 对于普通模式需要手动重启
        // HAL_UART_Receive_DMA(&huart1, dma_rx_buffer, 100);
    }
}

// DMA发送完成回调
void HAL_UART_TxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1)
    {
        // 发送完成处理
    }
}

// DMA错误回调
void HAL_UART_ErrorCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1)
    {
        // 处理DMA错误
        uint32_t error = HAL_UART_GetError(huart);
        
        if (error & HAL_UART_ERROR_DMA)
        {
            // DMA传输错误
            // 重新初始化DMA
            HAL_UART_DMAStop(&huart1);
            HAL_UART_Receive_DMA(&huart1, dma_rx_buffer, 100);
        }
    }
}

int main(void)
{
    HAL_Init();
    SystemClock_Config();
    
    // 初始化UART和DMA
    MX_USART1_UART_Init();
    MX_DMA_Init();
    
    // 启动DMA接收（循环模式）
    HAL_UART_Receive_DMA(&huart1, dma_rx_buffer, 100);
    
    while (1)
    {
        // 主循环处理其他任务
        
        // 定时发送数据
        static uint32_t last_send_time = 0;
        if (HAL_GetTick() - last_send_time > 1000)
        {
            last_send_time = HAL_GetTick();
            
            // 准备发送数据
            int len = sprintf((char*)dma_tx_buffer, "Time: %lu ms\r\n", last_send_time);
            
            // 使用DMA发送
            HAL_UART_Transmit_DMA(&huart1, dma_tx_buffer, len);
        }
        
        HAL_Delay(10);
    }
}
```

#### 示例2：ADC DMA采样

```c
ADC_HandleTypeDef hadc1;
DMA_HandleTypeDef hdma_adc1;

uint16_t adc_dma_buffer[100];

// ADC DMA配置
void ADC_DMA_Init(void)
{
    // 配置ADC
    hadc1.Instance = ADC1;
    hadc1.Init.ScanConvMode = ENABLE;
    hadc1.Init.ContinuousConvMode = ENABLE;
    hadc1.Init.DMAContinuousRequests = ENABLE; // 连续DMA请求
    hadc1.Init.NbrOfConversion = 1;
    // ... 其他配置
    HAL_ADC_Init(&hadc1);
    
    // 配置DMA
    hdma_adc1.Instance = DMA2_Channel0;
    hdma_adc1.Init.Direction = DMA_PERIPH_TO_MEMORY;
    hdma_adc1.Init.PeriphInc = DMA_PINC_DISABLE;
    hdma_adc1.Init.MemInc = DMA_MINC_ENABLE;
    hdma_adc1.Init.PeriphDataAlignment = DMA_PDATAALIGN_HALFWORD;
    hdma_adc1.Init.MemDataAlignment = DMA_MDATAALIGN_HALFWORD;
    hdma_adc1.Init.Mode = DMA_CIRCULAR;
    hdma_adc1.Init.Priority = DMA_PRIORITY_HIGH;
    
    HAL_DMA_Init(&hdma_adc1);
    
    // 关联DMA到ADC
    __HAL_LINKDMA(&hadc1, DMA_Handle, hdma_adc1);
    
    // 配置ADC通道
    ADC_ChannelConfTypeDef sConfig = {0};
    sConfig.Channel = ADC_CHANNEL_0;
    sConfig.Rank = ADC_REGULAR_RANK_1;
    sConfig.SamplingTime = ADC_SAMPLINGTIME_COMMON_1;
    HAL_ADC_ConfigChannel(&hadc1, &sConfig);
}

// ADC转换完成回调（半传输）
void HAL_ADC_ConvHalfCpltCallback(ADC_HandleTypeDef* hadc)
{
    if (hadc->Instance == ADC1)
    {
        // 处理前一半数据（adc_dma_buffer[0]到adc_dma_buffer[49]）
        process_adc_data(adc_dma_buffer, 50);
    }
}

// ADC转换完成回调（全传输）
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef* hadc)
{
    if (hadc->Instance == ADC1)
    {
        // 处理后一半数据（adc_dma_buffer[50]到adc_dma_buffer[99]）
        process_adc_data(&adc_dma_buffer[50], 50);
    }
}

int main(void)
{
    HAL_Init();
    SystemClock_Config();
    
    // 初始化ADC和DMA
    ADC_DMA_Init();
    
    // 启动ADC DMA转换
    HAL_ADC_Start_DMA(&hadc1, (uint32_t*)adc_dma_buffer, 100);
    
    while (1)
    {
        // 主循环可以处理其他任务
        // ADC数据在DMA回调函数中处理
        
        HAL_Delay(100);
    }
}
```

#### 示例3：存储器到存储器DMA

```c
DMA_HandleTypeDef hdma_memtomem;

uint32_t src_buffer[100];
uint32_t dst_buffer[100];

// 存储器到存储器DMA配置
void MEM2MEM_DMA_Init(void)
{
    // DMA时钟使能
    __HAL_RCC_DMA2_CLK_ENABLE();
    
    // 配置DMA
    hdma_memtomem.Instance = DMA2_Channel1;
    hdma_memtomem.Init.Direction = DMA_MEMORY_TO_MEMORY;
    hdma_memtomem.Init.PeriphInc = DMA_MINC_ENABLE;
    hdma_memtomem.Init.MemInc = DMA_MINC_ENABLE;
    hdma_memtomem.Init.PeriphDataAlignment = DMA_PDATAALIGN_WORD;
    hdma_memtomem.Init.MemDataAlignment = DMA_MDATAALIGN_WORD;
    hdma_memtomem.Init.Mode = DMA_NORMAL;
    hdma_memtomem.Init.Priority = DMA_PRIORITY_HIGH;
    
    HAL_DMA_Init(&hdma_memtomem);
    
    // 配置DMA中断
    HAL_NVIC_SetPriority(DMA2_Channel1_IRQn, 0, 0);
    HAL_NVIC_EnableIRQ(DMA2_Channel1_IRQn);
}

// DMA传输完成回调
void HAL_DMA_XferCpltCallback(DMA_HandleTypeDef *hdma)
{
    if (hdma->Instance == DMA2_Channel1)
    {
        // 存储器到存储器传输完成
        // printf("Memory copy completed!\r\n");
    }
}

// 执行存储器拷贝
void mem_copy_dma(uint32_t* src, uint32_t* dst, uint32_t size)
{
    // 启动DMA传输
    HAL_DMA_Start_IT(&hdma_memtomem, (uint32_t)src, (uint32_t)dst, size);
    
    // 等待传输完成（或使用回调函数）
    // HAL_DMA_PollForTransfer(&hdma_memtomem, HAL_DMA_FULL_TRANSFER, 1000);
}

int main(void)
{
    HAL_Init();
    SystemClock_Config();
    
    // 初始化存储器到存储器DMA
    MEM2MEM_DMA_Init();
    
    // 填充源缓冲区
    for (int i = 0; i < 100; i++)
    {
        src_buffer[i] = i;
    }
    
    // 使用DMA拷贝数据
    mem_copy_dma(src_buffer, dst_buffer, 100);
    
    // 验证拷贝结果
    if (memcmp(src_buffer, dst_buffer, 100 * sizeof(uint32_t)) == 0)
    {
        // printf("DMA memory copy successful!\r\n");
    }
    
    while (1)
    {
        HAL_Delay(1000);
    }
}
```

## 中断和DMA的协同工作

### 中断优先级管理

```c
// 合理设置中断优先级
void Interrupt_Priority_Config(void)
{
    // 设置优先级分组（4位抢占优先级，0位子优先级）
    HAL_NVIC_SetPriorityGrouping(NVIC_PRIORITYGROUP_4);
    
    // 高优先级中断（紧急任务）
    HAL_NVIC_SetPriority(SysTick_IRQn, 0, 0);     // 系统滴答定时器
    HAL_NVIC_SetPriority(EXTI0_IRQn, 1, 0);       // 紧急外部事件
    
    // 中等优先级中断
    HAL_NVIC_SetPriority(USART1_IRQn, 2, 0);      // 串口通信
    HAL_NVIC_SetPriority(TIM2_IRQn, 2, 0);        // 定时器
    
    // 低优先级中断
    HAL_NVIC_SetPriority(DMA1_Channel4_IRQn, 3, 0); // DMA传输
    
    // 使能所有中断
    HAL_NVIC_EnableIRQ(SysTick_IRQn);
    HAL_NVIC_EnableIRQ(EXTI0_IRQn);
    HAL_NVIC_EnableIRQ(USART1_IRQn);
    HAL_NVIC_EnableIRQ(TIM2_IRQn);
    HAL_NVIC_EnableIRQ(DMA1_Channel4_IRQn);
}
```

### 中断嵌套处理

```c
// 中断服务函数示例
void EXTI0_IRQHandler(void)
{
    // 高优先级中断
    HAL_GPIO_EXTI_IRQHandler(GPIO_PIN_0);
}

void USART1_IRQHandler(void)
{
    // 中等优先级中断
    HAL_UART_IRQHandler(&huart1);
}

// 在回调函数中避免长时间操作
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    if (GPIO_Pin == GPIO_PIN_0)
    {
        // 快速处理，设置标志位
        exti0_flag = 1;
        
        // 避免在中断中执行复杂操作
        // 复杂操作放到主循环中处理
    }
}

// 主循环中处理标志位
void main_loop(void)
{
    while (1)
    {
        if (exti0_flag)
        {
            exti0_flag = 0;
            // 执行复杂操作
            process_complex_task();
        }
        
        HAL_Delay(1);
    }
}
```

## 调试技巧

### 1. 中断性能分析

```c
// 测量中断响应时间
uint32_t interrupt_start_time;
uint32_t interrupt_response_time;

void EXTI0_IRQHandler(void)
{
    interrupt_start_time = DWT->CYCCNT; // 使用DWT计数器
    
    HAL_GPIO_EXTI_IRQHandler(GPIO_PIN_0);
    
    interrupt_response_time = DWT->CYCCNT - interrupt_start_time;
    
    // 转换为微秒（假设系统时钟84MHz）
    // float time_us = (float)interrupt_response_time / 84.0f;
}
```

### 2. DMA传输状态监控

```c
// 检查DMA传输状态
void check_dma_status(void)
{
    DMA_HandleTypeDef* hdma = &hdma_usart1_tx;
    
    // 检查传输是否完成
    if (__HAL_DMA_GET_FLAG(hdma, __HAL_DMA_GET_TC_FLAG_INDEX(hdma)))
    {
        // 传输完成
        __HAL_DMA_CLEAR_FLAG(hdma, __HAL_DMA_GET_TC_FLAG_INDEX(hdma));
    }
    
    // 检查传输错误
    if (__HAL_DMA_GET_FLAG(hdma, __HAL_DMA_GET_TE_FLAG_INDEX(hdma)))
    {
        // 传输错误
        __HAL_DMA_CLEAR_FLAG(hdma, __HAL_DMA_GET_TE_FLAG_INDEX(hdma));
        // 处理错误
    }
}
```

## 常见问题与解决方案

### 中断相关问题

**问题1：中断不触发**
**原因：** NVIC未使能、中断优先级配置错误
**解决：** 检查NVIC配置，确认中断使能和优先级设置正确

**问题2：中断嵌套异常**
**原因：** 中断优先级设置不合理
**解决：** 合理设置中断优先级，避免高优先级中断被低优先级中断阻塞

### DMA相关问题

**问题1：DMA传输数据错误**
**原因：** 内存对齐问题、缓冲区溢出
**解决：** 确保缓冲区地址对齐，检查缓冲区大小

**问题2：DMA传输不启动**
**原因：** DMA通道配置错误、外设未正确关联
**解决：** 检查DMA配置，确认外设与DMA正确关联

## 性能优化建议

1. **合理设置中断优先级**：确保关键任务有足够高的优先级
2. **使用DMA减少CPU负担**：大数据传输时优先使用DMA
3. **优化中断处理函数**：中断函数尽量简短，复杂操作放到主循环
4. **使用循环DMA模式**：连续数据流处理时使用循环模式
5. **合理分配内存**：DMA缓冲区使用对齐的内存地址

## 总结

中断和DMA是STM32中重要的系统特性，合理使用可以大大提高系统性能和响应速度。中断用于处理异步事件，DMA用于高效数据传输。通过HAL库提供的统一接口，可以方便地配置和使用这些功能。