# UART串口通信

## UART基本概念

UART（Universal Asynchronous Receiver/Transmitter）通用异步收发器是嵌入式系统中最常用的通信接口之一。

### UART通信原理

- **异步通信**：不需要时钟信号同步
- **起始位**：标志数据传输开始
- **数据位**：5-9位数据
- **校验位**：奇偶校验（可选）
- **停止位**：1-2位，标志传输结束

### 波特率计算

波特率 = 系统时钟 / (波特率分频系数)

## UART初始化配置

### CubeMX配置UART

1. 选择USART/UART外设
2. 配置模式（异步/同步）
3. 设置波特率（常用115200）
4. 配置数据位、停止位、校验位
5. 启用中断/DMA（可选）

### 手动配置UART结构体

```c
UART_HandleTypeDef huart1;

void UART1_Init(void)
{
    huart1.Instance = USART1;
    huart1.Init.BaudRate = 115200;
    huart1.Init.WordLength = UART_WORDLENGTH_8B;
    huart1.Init.StopBits = UART_STOPBITS_1;
    huart1.Init.Parity = UART_PARITY_NONE;
    huart1.Init.Mode = UART_MODE_TX_RX;
    huart1.Init.HwFlowCtl = UART_HWCONTROL_NONE;
    huart1.Init.OverSampling = UART_OVERSAMPLING_16;
    
    if (HAL_UART_Init(&huart1) != HAL_OK)
    {
        Error_Handler();
    }
}
```

## UART常用函数

### 阻塞式传输

```c
// 发送数据（阻塞）
uint8_t tx_data[] = "Hello World\r\n";
HAL_UART_Transmit(&huart1, tx_data, sizeof(tx_data)-1, HAL_MAX_DELAY);

// 接收数据（阻塞）
uint8_t rx_data[10];
HAL_UART_Receive(&huart1, rx_data, 10, HAL_MAX_DELAY);
```

### 中断方式传输

```c
// 发送数据（中断）
uint8_t tx_data[] = "Hello Interrupt\r\n";
HAL_UART_Transmit_IT(&huart1, tx_data, sizeof(tx_data)-1);

// 接收数据（中断）
uint8_t rx_buffer[100];
HAL_UART_Receive_IT(&huart1, rx_buffer, 1); // 每次接收1字节

// 发送完成回调函数
void HAL_UART_TxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1)
    {
        // 发送完成处理
    }
}

// 接收完成回调函数
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1)
    {
        // 处理接收到的数据
        // 重新启动接收
        HAL_UART_Receive_IT(&huart1, rx_buffer, 1);
    }
}
```

### DMA方式传输

```c
// 发送数据（DMA）
uint8_t dma_tx_data[] = "Hello DMA\r\n";
HAL_UART_Transmit_DMA(&huart1, dma_tx_data, sizeof(dma_tx_data)-1);

// 接收数据（DMA）
uint8_t dma_rx_buffer[100];
HAL_UART_Receive_DMA(&huart1, dma_rx_buffer, 100);

// DMA传输完成回调
void HAL_UART_TxHalfCpltCallback(UART_HandleTypeDef *huart)
{
    // 发送一半完成
}

void HAL_UART_TxCpltCallback(UART_HandleTypeDef *huart)
{
    // 发送完成
}

void HAL_UART_RxHalfCpltCallback(UART_HandleTypeDef *huart)
{
    // 接收一半完成
}

void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    // 接收完成
}
```

## 实战示例

### 示例1：基础串口通信

```c
#include "main.h"

UART_HandleTypeDef huart1;

int main(void)
{
    HAL_Init();
    SystemClock_Config();
    
    // UART初始化
    MX_USART1_UART_Init();
    
    uint8_t welcome_msg[] = "STM32 UART Demo Started\r\n";
    HAL_UART_Transmit(&huart1, welcome_msg, sizeof(welcome_msg)-1, 1000);
    
    uint8_t rx_data;
    uint8_t counter = 0;
    
    while (1)
    {
        // 等待接收数据
        if (HAL_UART_Receive(&huart1, &rx_data, 1, 100) == HAL_OK)
        {
            // 回显接收到的数据
            HAL_UART_Transmit(&huart1, &rx_data, 1, 100);
            
            // 发送计数信息
            uint8_t msg[50];
            int len = sprintf((char*)msg, "Received: %c, Count: %d\r\n", rx_data, counter++);
            HAL_UART_Transmit(&huart1, msg, len, 1000);
        }
        
        HAL_Delay(10);
    }
}
```

### 示例2：中断方式接收数据

```c
#include "main.h"

UART_HandleTypeDef huart1;
uint8_t rx_buffer[100];
uint8_t rx_index = 0;

// 重写接收完成回调函数
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1)
    {
        // 处理接收到的字节
        if (rx_buffer[rx_index] == '\r' || rx_buffer[rx_index] == '\n')
        {
            // 收到回车或换行，处理完整命令
            if (rx_index > 0)
            {
                // 回显接收到的命令
                uint8_t echo_msg[50];
                int len = sprintf((char*)echo_msg, "Command: ");
                HAL_UART_Transmit(&huart1, echo_msg, len, 100);
                HAL_UART_Transmit(&huart1, rx_buffer, rx_index, 100);
                HAL_UART_Transmit(&huart1, (uint8_t*)"\r\n", 2, 100);
                
                // 处理命令（示例）
                if (strncmp((char*)rx_buffer, "LED ON", 6) == 0)
                {
                    HAL_GPIO_WritePin(LED_GPIO_Port, LED_Pin, GPIO_PIN_SET);
                    HAL_UART_Transmit(&huart1, (uint8_t*)"LED Turned ON\r\n", 14, 100);
                }
                else if (strncmp((char*)rx_buffer, "LED OFF", 7) == 0)
                {
                    HAL_GPIO_WritePin(LED_GPIO_Port, LED_Pin, GPIO_PIN_RESET);
                    HAL_UART_Transmit(&huart1, (uint8_t*)"LED Turned OFF\r\n", 15, 100);
                }
                
                rx_index = 0; // 重置索引
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

int main(void)
{
    HAL_Init();
    SystemClock_Config();
    
    // 初始化GPIO和UART
    MX_GPIO_Init();
    MX_USART1_UART_Init();
    
    // 启动中断接收
    HAL_UART_Receive_IT(&huart1, &rx_buffer[0], 1);
    
    // 发送欢迎信息
    uint8_t welcome[] = "UART Interrupt Demo Ready\r\n";
    HAL_UART_Transmit(&huart1, welcome, sizeof(welcome)-1, 100);
    
    while (1)
    {
        // 主循环可以处理其他任务
        HAL_Delay(1000);
    }
}
```

### 示例3：DMA方式收发数据

```c
#include "main.h"
#include <string.h>

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
        uint8_t response[50];
        int len = sprintf((char*)response, "Received %d bytes via DMA\r\n", sizeof(dma_rx_buffer));
        
        // 使用DMA发送响应
        HAL_UART_Transmit_DMA(&huart1, response, len);
        
        // 重新启动DMA接收
        HAL_UART_Receive_DMA(&huart1, dma_rx_buffer, sizeof(dma_rx_buffer));
    }
}

int main(void)
{
    HAL_Init();
    SystemClock_Config();
    
    // 初始化外设
    MX_GPIO_Init();
    MX_DMA_Init();
    MX_USART1_UART_Init();
    
    // 启动DMA接收
    HAL_UART_Receive_DMA(&huart1, dma_rx_buffer, sizeof(dma_rx_buffer));
    
    // 发送初始信息
    uint8_t init_msg[] = "UART DMA Demo Started\r\n";
    HAL_UART_Transmit_DMA(&huart1, init_msg, sizeof(init_msg)-1);
    
    uint32_t last_send_time = 0;
    
    while (1)
    {
        uint32_t current_time = HAL_GetTick();
        
        // 每5秒发送一次数据
        if (current_time - last_send_time > 5000)
        {
            last_send_time = current_time;
            
            // 准备要发送的数据
            int len = sprintf((char*)dma_tx_buffer, "System Time: %lu ms\r\n", current_time);
            
            // 使用DMA发送
            HAL_UART_Transmit_DMA(&huart1, dma_tx_buffer, len);
        }
        
        HAL_Delay(100);
    }
}
```

## 高级功能

### 1. 多串口通信

STM32通常有多个UART接口，可以同时使用：

```c
UART_HandleTypeDef huart1, huart2, huart3;

// 初始化多个UART
void MX_USART1_UART_Init(void)
{
    huart1.Instance = USART1;
    // ... 配置参数
    HAL_UART_Init(&huart1);
}

void MX_USART2_UART_Init(void)
{
    huart2.Instance = USART2;
    // ... 配置参数
    HAL_UART_Init(&huart2);
}

// 数据转发示例
void uart_forward_data(void)
{
    uint8_t data;
    
    // 从UART1接收，转发到UART2
    if (HAL_UART_Receive(&huart1, &data, 1, 0) == HAL_OK)
    {
        HAL_UART_Transmit(&huart2, &data, 1, 100);
    }
}
```

### 2. 自定义协议解析

```c
// 简单的协议解析器
typedef struct {
    uint8_t header[2];    // 协议头
    uint8_t length;       // 数据长度
    uint8_t data[32];     // 数据内容
    uint8_t checksum;     // 校验和
} uart_protocol_t;

uint8_t protocol_parse(uint8_t* buffer, uint16_t length)
{
    if (length < 4) return 0; // 最小长度检查
    
    uart_protocol_t* protocol = (uart_protocol_t*)buffer;
    
    // 检查协议头
    if (protocol->header[0] != 0xAA || protocol->header[1] != 0x55)
        return 0;
    
    // 检查数据长度
    if (protocol->length > 32 || length != (4 + protocol->length))
        return 0;
    
    // 计算校验和
    uint8_t calc_checksum = 0;
    for (int i = 0; i < 3 + protocol->length; i++)
    {
        calc_checksum ^= buffer[i];
    }
    
    if (calc_checksum != protocol->checksum)
        return 0;
    
    return 1; // 协议解析成功
}
```

### 3. 流控制（RTS/CTS）

```c
// 配置硬件流控制
huart1.Init.HwFlowCtl = UART_HWCONTROL_RTS_CTS;

// 手动控制RTS引脚
void uart_set_rts(GPIO_PinState state)
{
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_1, state); // PA1作为RTS
}
```

## 调试技巧

### 1. 使用printf重定向

```c
#include <stdio.h>

// 重写fputc函数
int fputc(int ch, FILE *f)
{
    HAL_UART_Transmit(&huart1, (uint8_t*)&ch, 1, 100);
    return ch;
}

// 现在可以使用printf
printf("System started, tick: %lu\r\n", HAL_GetTick());
```

### 2. 串口调试信息

```c
// 调试信息输出函数
void debug_printf(const char* format, ...)
{
    char buffer[128];
    va_list args;
    va_start(args, format);
    int len = vsnprintf(buffer, sizeof(buffer), format, args);
    va_end(args);
    
    if (len > 0)
    {
        HAL_UART_Transmit(&huart1, (uint8_t*)buffer, len, 1000);
    }
}
```

## 常见问题与解决方案

### 问题1：数据接收不完整

**原因：** 波特率不匹配、时钟配置错误
**解决：** 检查时钟树配置，确保波特率计算正确

### 问题2：中断接收丢失数据

**原因：** 中断优先级过低、处理时间过长
**解决：** 提高UART中断优先级，简化中断处理函数

### 问题3：DMA传输异常

**原因：** DMA通道配置错误、内存对齐问题
**解决：** 检查DMA配置，确保缓冲区地址对齐

## 性能优化建议

1. **使用DMA**：大数据量传输时使用DMA减少CPU占用
2. **合理设置缓冲区**：根据实际需求设置合适的缓冲区大小
3. **优化中断处理**：中断函数尽量简短，避免复杂操作
4. **使用双缓冲区**：实现乒乓缓冲区提高数据传输效率

## 总结

UART是STM32最常用的通信接口，通过HAL库可以方便地实现各种通信需求。掌握阻塞、中断、DMA三种传输方式，能够根据实际应用场景选择最合适的方案。