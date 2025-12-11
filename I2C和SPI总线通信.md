# I2C和SPI总线通信

## I2C总线通信

### I2C基本概念

I2C（Inter-Integrated Circuit）是一种同步、多主从、串行通信总线，使用两条线（SDA和SCL）进行通信。

**I2C特性：**
- 两线制：SDA（数据线）、SCL（时钟线）
- 多主从架构
- 7位或10位设备地址
- 标准模式（100kHz）、快速模式（400kHz）、高速模式（3.4MHz）

### I2C初始化配置

#### CubeMX配置I2C

1. 选择I2C外设
2. 配置模式（主模式/从模式）
3. 设置时钟速度
4. 配置从机地址
5. 启用DMA/中断（可选）

#### 手动配置I2C结构体

```c
I2C_HandleTypeDef hi2c1;

void I2C1_Init(void)
{
    hi2c1.Instance = I2C1;
    hi2c1.Init.Timing = 0x10909CEC; // 400kHz时钟
    hi2c1.Init.OwnAddress1 = 0;
    hi2c1.Init.AddressingMode = I2C_ADDRESSINGMODE_7BIT;
    hi2c1.Init.DualAddressMode = I2C_DUALADDRESS_DISABLE;
    hi2c1.Init.OwnAddress2 = 0;
    hi2c1.Init.OwnAddress2Masks = I2C_OA2_NOMASK;
    hi2c1.Init.GeneralCallMode = I2C_GENERALCALL_DISABLE;
    hi2c1.Init.NoStretchMode = I2C_NOSTRETCH_DISABLE;
    
    if (HAL_I2C_Init(&hi2c1) != HAL_OK)
    {
        Error_Handler();
    }
    
    // 配置模拟滤波器（可选）
    if (HAL_I2CEx_ConfigAnalogFilter(&hi2c1, I2C_ANALOGFILTER_ENABLE) != HAL_OK)
    {
        Error_Handler();
    }
    
    // 配置数字滤波器（可选）
    if (HAL_I2CEx_ConfigDigitalFilter(&hi2c1, 0) != HAL_OK)
    {
        Error_Handler();
    }
}
```

### I2C常用函数

#### 主模式函数

```c
// 阻塞式发送数据
uint8_t tx_data[] = {0x01, 0x02, 0x03};
HAL_I2C_Master_Transmit(&hi2c1, 0xA0, tx_data, 3, 1000);

// 阻塞式接收数据
uint8_t rx_data[10];
HAL_I2C_Master_Receive(&hi2c1, 0xA0, rx_data, 10, 1000);

// 中断方式发送
HAL_I2C_Master_Transmit_IT(&hi2c1, 0xA0, tx_data, 3);

// 中断方式接收
HAL_I2C_Master_Receive_IT(&hi2c1, 0xA0, rx_data, 10);

// DMA方式发送
HAL_I2C_Master_Transmit_DMA(&hi2c1, 0xA0, tx_data, 3);

// DMA方式接收
HAL_I2C_Master_Receive_DMA(&hi2c1, 0xA0, rx_data, 10);
```

#### 存储器访问函数

```c
// 写入存储器（带寄存器地址）
uint8_t reg_addr = 0x10;
uint8_t data[] = {0xAA, 0xBB, 0xCC};
HAL_I2C_Mem_Write(&hi2c1, 0xA0, reg_addr, I2C_MEMADD_SIZE_8BIT, data, 3, 1000);

// 读取存储器（带寄存器地址）
uint8_t read_data[5];
HAL_I2C_Mem_Read(&hi2c1, 0xA0, reg_addr, I2C_MEMADD_SIZE_8BIT, read_data, 5, 1000);
```

#### 从模式函数

```c
// 从机发送数据
HAL_I2C_Slave_Transmit(&hi2c1, tx_data, 3, 1000);

// 从机接收数据
HAL_I2C_Slave_Receive(&hi2c1, rx_data, 10, 1000);
```

### I2C实战示例

#### 示例1：读取MPU6050加速度计

```c
#include "main.h"
#include <math.h>

I2C_HandleTypeDef hi2c1;

// MPU6050设备地址
#define MPU6050_ADDR 0x68

// MPU6050寄存器地址
#define MPU6050_WHO_AM_I 0x75
#define MPU6050_PWR_MGMT_1 0x6B
#define MPU6050_ACCEL_XOUT_H 0x3B

// 初始化MPU6050
uint8_t mpu6050_init(void)
{
    uint8_t check, data;
    
    // 检查设备ID
    HAL_I2C_Mem_Read(&hi2c1, MPU6050_ADDR<<1, MPU6050_WHO_AM_I, 1, &check, 1, 100);
    if (check != 0x68) // MPU6050的WHO_AM_I寄存器值应为0x68
    {
        return 0; // 设备未找到
    }
    
    // 唤醒MPU6050
    data = 0x00;
    HAL_I2C_Mem_Write(&hi2c1, MPU6050_ADDR<<1, MPU6050_PWR_MGMT_1, 1, &data, 1, 100);
    
    return 1; // 初始化成功
}

// 读取加速度数据
void mpu6050_read_accel(int16_t* accel_data)
{
    uint8_t buffer[6];
    
    // 读取加速度数据（6字节：X、Y、Z各2字节）
    HAL_I2C_Mem_Read(&hi2c1, MPU6050_ADDR<<1, MPU6050_ACCEL_XOUT_H, 1, buffer, 6, 100);
    
    // 组合数据（高位在前）
    accel_data[0] = (int16_t)((buffer[0] << 8) | buffer[1]); // X轴
    accel_data[1] = (int16_t)((buffer[2] << 8) | buffer[3]); // Y轴
    accel_data[2] = (int16_t)((buffer[4] << 8) | buffer[5]); // Z轴
}

int main(void)
{
    HAL_Init();
    SystemClock_Config();
    
    // 初始化I2C和UART
    MX_I2C1_Init();
    MX_USART1_UART_Init();
    
    // 初始化MPU6050
    if (!mpu6050_init())
    {
        // printf("MPU6050初始化失败\r\n");
        while(1);
    }
    
    // printf("MPU6050初始化成功\r\n");
    
    int16_t accel_data[3];
    
    while (1)
    {
        // 读取加速度数据
        mpu6050_read_accel(accel_data);
        
        // 转换为重力加速度（MPU6050默认量程为±2g）
        float accel_x = accel_data[0] / 16384.0f;
        float accel_y = accel_data[1] / 16384.0f;
        float accel_z = accel_data[2] / 16384.0f;
        
        // 输出数据
        // printf("Accel: X=%.3fg, Y=%.3fg, Z=%.3fg\r\n", accel_x, accel_y, accel_z);
        
        HAL_Delay(100); // 每100ms读取一次
    }
}
```

#### 示例2：OLED显示屏驱动（SSD1306）

```c
#include "main.h"
#include <string.h>

I2C_HandleTypeDef hi2c1;

// SSD1306设备地址
#define SSD1306_ADDR 0x78

// SSD1306命令定义
#define SSD1306_SET_CONTRAST 0x81
#define SSD1306_DISPLAY_ALL_ON_RESUME 0xA4
#define SSD1306_DISPLAY_ON 0xAF
#define SSD1306_MEMORY_MODE 0x20
#define SSD1306_COLUMN_ADDR 0x21
#define SSD1306_PAGE_ADDR 0x22

// OLED显示缓冲区
uint8_t oled_buffer[1024]; // 128x64分辨率，8页

// 发送命令到OLED
void oled_write_command(uint8_t command)
{
    HAL_I2C_Mem_Write(&hi2c1, SSD1306_ADDR, 0x00, 1, &command, 1, 100);
}

// 发送数据到OLED
void oled_write_data(uint8_t data)
{
    HAL_I2C_Mem_Write(&hi2c1, SSD1306_ADDR, 0x40, 1, &data, 1, 100);
}

// 初始化OLED
void oled_init(void)
{
    // 初始化命令序列
    oled_write_command(0xAE); // 关闭显示
    oled_write_command(0x20); // 设置内存地址模式
    oled_write_command(0x00); // 水平地址模式
    oled_write_command(0xB0); // 设置页地址
    oled_write_command(0xC8); // 设置扫描方向
    oled_write_command(0x00); // 设置列地址低4位
    oled_write_command(0x10); // 设置列地址高4位
    oled_write_command(0x40); // 设置起始行
    oled_write_command(0x81); // 设置对比度
    oled_write_command(0xFF); // 对比度值
    oled_write_command(0xA1); // 设置段重映射
    oled_write_command(0xA6); // 设置正常显示
    oled_write_command(0xA8); // 设置多路复用比率
    oled_write_command(0x3F); // 1/64 duty
    oled_write_command(0xA4); // 输出跟随RAM内容
    oled_write_command(0xD3); // 设置显示偏移
    oled_write_command(0x00); // 无偏移
    oled_write_command(0xD5); // 设置显示时钟分频比/振荡器频率
    oled_write_command(0xF0); // 设置分频比
    oled_write_command(0xD9); // 设置预充电周期
    oled_write_command(0x22); // 
    oled_write_command(0xDA); // 设置COM引脚硬件配置
    oled_write_command(0x12); // 
    oled_write_command(0xDB); // 设置VCOMH
    oled_write_command(0x20); // 0.77xVcc
    oled_write_command(0x8D); // 设置充电泵
    oled_write_command(0x14); // 启用充电泵
    oled_write_command(0xAF); // 打开OLED面板
}

// 清屏
void oled_clear(void)
{
    memset(oled_buffer, 0, sizeof(oled_buffer));
}

// 更新显示
void oled_update(void)
{
    for (uint8_t page = 0; page < 8; page++)
    {
        oled_write_command(0xB0 + page); // 设置页地址
        oled_write_command(0x00);         // 设置列地址低4位
        oled_write_command(0x10);         // 设置列地址高4位
        
        // 发送该页的数据
        for (uint8_t col = 0; col < 128; col++)
        {
            oled_write_data(oled_buffer[page * 128 + col]);
        }
    }
}

int main(void)
{
    HAL_Init();
    SystemClock_Config();
    
    // 初始化I2C
    MX_I2C1_Init();
    
    // 初始化OLED
    oled_init();
    oled_clear();
    
    // 简单的显示测试
    // 这里可以添加图形绘制函数
    
    while (1)
    {
        oled_update();
        HAL_Delay(100);
    }
}
```

## SPI总线通信

### SPI基本概念

SPI（Serial Peripheral Interface）是一种同步、全双工、主从式串行通信接口。

**SPI特性：**
- 四线制：SCK、MOSI、MISO、CS
- 全双工通信
- 高速传输（可达几十MHz）
- 支持多从机（通过片选信号）

### SPI初始化配置

#### CubeMX配置SPI

1. 选择SPI外设
2. 配置模式（主模式/从模式）
3. 设置时钟极性和相位
4. 配置数据大小和位序
5. 设置时钟分频
6. 启用DMA/中断（可选）

#### 手动配置SPI结构体

```c
SPI_HandleTypeDef hspi1;

void SPI1_Init(void)
{
    hspi1.Instance = SPI1;
    hspi1.Init.Mode = SPI_MODE_MASTER;
    hspi1.Init.Direction = SPI_DIRECTION_2LINES;
    hspi1.Init.DataSize = SPI_DATASIZE_8BIT;
    hspi1.Init.CLKPolarity = SPI_POLARITY_LOW;
    hspi1.Init.CLKPhase = SPI_PHASE_1EDGE;
    hspi1.Init.NSS = SPI_NSS_SOFT;
    hspi1.Init.BaudRatePrescaler = SPI_BAUDRATEPRESCALER_64;
    hspi1.Init.FirstBit = SPI_FIRSTBIT_MSB;
    hspi1.Init.TIMode = SPI_TIMODE_DISABLE;
    hspi1.Init.CRCCalculation = SPI_CRCCALCULATION_DISABLE;
    hspi1.Init.CRCPolynomial = 10;
    
    if (HAL_SPI_Init(&hspi1) != HAL_OK)
    {
        Error_Handler();
    }
}
```

### SPI常用函数

#### 基本通信函数

```c
// 阻塞式发送数据
uint8_t tx_data[] = {0x01, 0x02, 0x03};
HAL_SPI_Transmit(&hspi1, tx_data, 3, 1000);

// 阻塞式接收数据
uint8_t rx_data[10];
HAL_SPI_Receive(&hspi1, rx_data, 10, 1000);

// 全双工通信
uint8_t tx_data[] = {0x01, 0x02, 0x03};
uint8_t rx_data[3];
HAL_SPI_TransmitReceive(&hspi1, tx_data, rx_data, 3, 1000);

// 中断方式发送
HAL_SPI_Transmit_IT(&hspi1, tx_data, 3);

// 中断方式接收
HAL_SPI_Receive_IT(&hspi1, rx_data, 10);

// DMA方式发送
HAL_SPI_Transmit_DMA(&hspi1, tx_data, 3);

// DMA方式接收
HAL_SPI_Receive_DMA(&hspi1, rx_data, 10);
```

### SPI实战示例

#### 示例1：SPI Flash读写（W25Q64）

```c
#include "main.h"

SPI_HandleTypeDef hspi1;

// W25Q64命令定义
#define W25X_WriteEnable 0x06
#define W25X_WriteDisable 0x04
#define W25X_ReadStatusReg 0x05
#define W25X_WriteStatusReg 0x01
#define W25X_ReadData 0x03
#define W25X_FastReadData 0x0B
#define W25X_PageProgram 0x02
#define W25X_BlockErase 0xD8
#define W25X_SectorErase 0x20
#define W25X_ChipErase 0xC7
#define W25X_PowerDown 0xB9
#define W25X_ReleasePowerDown 0xAB
#define W25X_DeviceID 0xAB
#define W25X_ManufactDeviceID 0x90
#define W25X_JedecDeviceID 0x9F

// 片选控制（软件控制）
#define FLASH_CS_LOW()  HAL_GPIO_WritePin(FLASH_CS_GPIO_Port, FLASH_CS_Pin, GPIO_PIN_RESET)
#define FLASH_CS_HIGH() HAL_GPIO_WritePin(FLASH_CS_GPIO_Port, FLASH_CS_Pin, GPIO_PIN_SET)

// 发送命令到Flash
void flash_send_command(uint8_t cmd)
{
    FLASH_CS_LOW();
    HAL_SPI_Transmit(&hspi1, &cmd, 1, 100);
    FLASH_CS_HIGH();
}

// 读取Flash状态寄存器
uint8_t flash_read_status(void)
{
    uint8_t status;
    uint8_t cmd = W25X_ReadStatusReg;
    
    FLASH_CS_LOW();
    HAL_SPI_Transmit(&hspi1, &cmd, 1, 100);
    HAL_SPI_Receive(&hspi1, &status, 1, 100);
    FLASH_CS_HIGH();
    
    return status;
}

// 等待Flash操作完成
void flash_wait_busy(void)
{
    while (flash_read_status() & 0x01) // 检查BUSY位
    {
        HAL_Delay(1);
    }
}

// 读取Flash ID
uint32_t flash_read_id(void)
{
    uint8_t cmd = W25X_JedecDeviceID;
    uint8_t id_data[3];
    
    FLASH_CS_LOW();
    HAL_SPI_Transmit(&hspi1, &cmd, 1, 100);
    HAL_SPI_Receive(&hspi1, id_data, 3, 100);
    FLASH_CS_HIGH();
    
    return (id_data[0] << 16) | (id_data[1] << 8) | id_data[2];
}

// 从Flash读取数据
void flash_read_data(uint32_t addr, uint8_t* buffer, uint32_t len)
{
    uint8_t cmd[4];
    cmd[0] = W25X_ReadData;
    cmd[1] = (addr >> 16) & 0xFF;
    cmd[2] = (addr >> 8) & 0xFF;
    cmd[3] = addr & 0xFF;
    
    FLASH_CS_LOW();
    HAL_SPI_Transmit(&hspi1, cmd, 4, 100);
    HAL_SPI_Receive(&hspi1, buffer, len, 1000);
    FLASH_CS_HIGH();
}

// 向Flash写入数据（页编程）
void flash_write_page(uint32_t addr, uint8_t* data, uint32_t len)
{
    uint8_t cmd[4];
    
    // 使能写入
    flash_send_command(W25X_WriteEnable);
    
    cmd[0] = W25X_PageProgram;
    cmd[1] = (addr >> 16) & 0xFF;
    cmd[2] = (addr >> 8) & 0xFF;
    cmd[3] = addr & 0xFF;
    
    FLASH_CS_LOW();
    HAL_SPI_Transmit(&hspi1, cmd, 4, 100);
    HAL_SPI_Transmit(&hspi1, data, len, 1000);
    FLASH_CS_HIGH();
    
    flash_wait_busy();
}

int main(void)
{
    HAL_Init();
    SystemClock_Config();
    
    // 初始化SPI和GPIO
    MX_SPI1_Init();
    MX_GPIO_Init();
    
    // 读取Flash ID
    uint32_t flash_id = flash_read_id();
    // printf("Flash ID: 0x%06lX\r\n", flash_id);
    
    // 测试读写
    uint8_t test_data[16] = "Hello SPI Flash!";
    uint8_t read_buffer[16];
    
    // 写入数据
    flash_write_page(0x000000, test_data, 16);
    
    // 读取数据
    flash_read_data(0x000000, read_buffer, 16);
    
    // 验证数据
    if (memcmp(test_data, read_buffer, 16) == 0)
    {
        // printf("SPI Flash测试成功!\r\n");
    }
    else
    {
        // printf("SPI Flash测试失败!\r\n");
    }
    
    while (1)
    {
        HAL_Delay(1000);
    }
}
```

#### 示例2：SPI TFT显示屏驱动

```c
#include "main.h"

SPI_HandleTypeDef hspi2;

// TFT控制引脚定义
#define TFT_CS_PIN   GPIO_PIN_0
#define TFT_CS_PORT  GPIOA
#define TFT_DC_PIN   GPIO_PIN_1
#define TFT_DC_PORT  GPIOA
#define TFT_RST_PIN  GPIO_PIN_2
#define TFT_RST_PORT GPIOA

// 控制引脚宏定义
#define TFT_CS_LOW()  HAL_GPIO_WritePin(TFT_CS_PORT, TFT_CS_PIN, GPIO_PIN_RESET)
#define TFT_CS_HIGH() HAL_GPIO_WritePin(TFT_CS_PORT, TFT_CS_PIN, GPIO_PIN_SET)
#define TFT_DC_LOW()  HAL_GPIO_WritePin(TFT_DC_PORT, TFT_DC_PIN, GPIO_PIN_RESET)
#define TFT_DC_HIGH() HAL_GPIO_WritePin(TFT_DC_PORT, TFT_DC_PIN, GPIO_PIN_SET)
#define TFT_RST_LOW() HAL_GPIO_WritePin(TFT_RST_PORT, TFT_RST_PIN, GPIO_PIN_RESET)
#define TFT_RST_HIGH() HAL_GPIO_WritePin(TFT_RST_PORT, TFT_RST_PIN, GPIO_PIN_SET)

// 发送命令到TFT
void tft_write_command(uint8_t cmd)
{
    TFT_DC_LOW(); // 命令模式
    TFT_CS_LOW();
    HAL_SPI_Transmit(&hspi2, &cmd, 1, 100);
    TFT_CS_HIGH();
}

// 发送数据到TFT
void tft_write_data(uint8_t data)
{
    TFT_DC_HIGH(); // 数据模式
    TFT_CS_LOW();
    HAL_SPI_Transmit(&hspi2, &data, 1, 100);
    TFT_CS_HIGH();
}

// 初始化TFT显示屏
void tft_init(void)
{
    // 硬件复位
    TFT_RST_HIGH();
    HAL_Delay(10);
    TFT_RST_LOW();
    HAL_Delay(10);
    TFT_RST_HIGH();
    HAL_Delay(120);
    
    // 初始化命令序列（以ST7735为例）
    tft_write_command(0x11); // 睡眠退出
    HAL_Delay(120);
    
    tft_write_command(0x21); // 显示反转开
    
    tft_write_command(0xB1); // 帧率控制
    tft_write_data(0x05);
    tft_write_data(0x3A);
    tft_write_data(0x3A);
    
    tft_write_command(0xB2); // 帧率控制
    tft_write_data(0x05);
    tft_write_data(0x3A);
    tft_write_data(0x3A);
    
    tft_write_command(0xB3); // 帧率控制
    tft_write_data(0x05);
    tft_write_data(0x3A);
    tft_write_data(0x3A);
    tft_write_data(0x05);
    tft_write_data(0x3A);
    tft_write_data(0x3A);
    
    tft_write_command(0x36); // 内存数据访问控制
    tft_write_data(0x08);
    
    tft_write_command(0x3A); // 接口像素格式
    tft_write_data(0x05);    // 16位像素
    
    tft_write_command(0x29); // 显示开
}

// 设置显示窗口
void tft_set_window(uint16_t x1, uint16_t y1, uint16_t x2, uint16_t y2)
{
    tft_write_command(0x2A); // 列地址设置
    tft_write_data(x1 >> 8);
    tft_write_data(x1 & 0xFF);
    tft_write_data(x2 >> 8);
    tft_write_data(x2 & 0xFF);
    
    tft_write_command(0x2B); // 行地址设置
    tft_write_data(y1 >> 8);
    tft_write_data(y1 & 0xFF);
    tft_write_data(y2 >> 8);
    tft_write_data(y2 & 0xFF);
    
    tft_write_command(0x2C); // 内存写
}

// 绘制像素点
void tft_draw_pixel(uint16_t x, uint16_t y, uint16_t color)
{
    tft_set_window(x, y, x, y);
    tft_write_data(color >> 8);
    tft_write_data(color & 0xFF);
}

// 填充矩形
void tft_fill_rect(uint16_t x, uint16_t y, uint16_t w, uint16_t h, uint16_t color)
{
    tft_set_window(x, y, x + w - 1, y + h - 1);
    
    uint32_t pixels = w * h;
    
    TFT_DC_HIGH();
    TFT_CS_LOW();
    
    for (uint32_t i = 0; i < pixels; i++)
    {
        uint8_t data[2] = {color >> 8, color & 0xFF};
        HAL_SPI_Transmit(&hspi2, data, 2, 100);
    }
    
    TFT_CS_HIGH();
}

int main(void)
{
    HAL_Init();
    SystemClock_Config();
    
    // 初始化SPI和GPIO
    MX_SPI2_Init();
    MX_GPIO_Init();
    
    // 初始化TFT
    tft_init();
    
    // 清屏为黑色
    tft_fill_rect(0, 0, 128, 160, 0x0000); // 黑色
    
    // 绘制红色矩形
    tft_fill_rect(10, 10, 50, 30, 0xF800); // 红色
    
    // 绘制绿色矩形
    tft_fill_rect(70, 10, 50, 30, 0x07E0); // 绿色
    
    // 绘制蓝色矩形
    tft_fill_rect(10, 50, 50, 30, 0x001F); // 蓝色
    
    while (1)
    {
        HAL_Delay(1000);
    }
}
```

## 总线通信对比

| 特性 | I2C | SPI |
|------|-----|-----|
| 线数 | 2线（SDA、SCL） | 4线（MOSI、MISO、SCK、CS） |
| 速度 | 较慢（100kHz-3.4MHz） | 较快（可达几十MHz） |
| 复杂度 | 简单，硬件开销小 | 较复杂，需要更多引脚 |
| 寻址方式 | 软件寻址（设备地址） | 硬件寻址（片选信号） |
| 多设备支持 | 容易（总线拓扑） | 需要多个片选信号 |
| 应用场景 | 传感器、EEPROM等 | Flash、显示屏、ADC等 |

## 调试技巧

### 1. I2C总线扫描

```c
// 扫描I2C总线上的设备
void i2c_scan(void)
{
    uint8_t error;
    
    // printf("Scanning I2C bus...\r\n");
    
    for (uint8_t addr = 1; addr < 127; addr++)
    {
        // 尝试与设备通信
        error = HAL_I2C_IsDeviceReady(&hi2c1, addr << 1, 3, 10);
        
        if (error == HAL_OK)
        {
            // printf("Device found at address: 0x%02X\r\n", addr);
        }
    }
}
```

### 2. SPI信号质量测试

```c
// 测试SPI通信速率
void spi_speed_test(void)
{
    uint8_t test_data[100];
    uint8_t receive_data[100];
    uint32_t start_time, end_time;
    
    // 填充测试数据
    for (int i = 0; i < 100; i++)
    {
        test_data[i] = i;
    }
    
    start_time = HAL_GetTick();
    
    // 执行多次传输
    for (int i = 0; i < 1000; i++)
    {
        HAL_SPI_TransmitReceive(&hspi1, test_data, receive_data, 100, 1000);
    }
    
    end_time = HAL_GetTick();
    
    uint32_t total_bytes = 1000 * 100 * 2; // 双向传输
    float speed_kbps = (total_bytes * 8.0f) / (end_time - start_time);
    
    // printf("SPI Speed: %.2f kbps\r\n", speed_kbps);
}
```

## 常见问题与解决方案

### I2C常见问题

**问题1：总线锁死**
**原因：** 从机未正确响应
**解决：** 发送多个时钟脉冲复位总线

**问题2：通信超时**
**原因：** 设备地址错误、总线干扰
**解决：** 检查设备地址、添加上拉电阻

### SPI常见问题

**问题1：数据错位**
**原因：** 时钟极性和相位配置错误
**解决：** 检查设备手册，正确配置CPOL和CPHA

**问题2：通信不稳定**
**原因：** 时钟频率过高、信号质量差
**解决：** 降低时钟频率、检查硬件连接

## 总结

I2C和SPI是STM32中常用的串行通信接口，各有优缺点。I2C适合连接多个低速设备，SPI适合高速数据传输。通过HAL库提供的统一接口，可以方便地实现各种外设的驱动和控制。