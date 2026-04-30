# GD32H759 BSP 架构设计

基于引脚与外部电路映射表设计的板级支持包（BSP）架构方案。

## 一、目录结构

```
bsp/
├── bsp.h                     # BSP 总入口，汇总包含所有模块头文件
├── bsp_pinmap.h              # 全局引脚映射（单一真源，按外设分组）
├── bsp_system.h / .c         # 系统初始化（时钟/PMU/MPU/Cache）
├── bsp_led.h / .c            # LED 驱动（3个）
├── bsp_key.h / .c            # 按键驱动（3个，含 WKUP）
├── bsp_buzzer.h / .c         # 蜂鸣器驱动
├── bsp_uart.h / .c           # 串口驱动（USART0/1, UART3/4）
├── bsp_i2c.h / .c            # I2C 驱动（触摸屏、OLED）
├── bsp_spi.h / .c            # SPI 驱动（SPI2，IO 扩展排针）
├── bsp_can.h / .c            # CAN 驱动（CAN2）
├── bsp_485.h / .c            # RS-485 驱动（UART4 + PB4 方向控制）
├── bsp_eth.h / .c            # 以太网驱动（LAN8720A RMII）
├── bsp_sdio.h / .c           # SD 卡驱动（SDIO0 + TF 卡槽）
├── bsp_lcd.h / .c            # TFT LCD 驱动（TLI + RGB888）
├── bsp_touch.h / .c          # 触摸屏驱动（I2C + 复位 + 中断）
├── bsp_oled.h / .c           # OLED 显示屏（I2C）
├── bsp_sdram.h / .c          # SDRAM 驱动（EXMC）
├── bsp_qspi.h / .c           # QSPI Flash 驱动（OSPI）
├── bsp_usb.h / .c            # USB 驱动（USBHS0/1）
├── bsp_dht11.h / .c          # 温湿度传感器（DHT11/DS18B20 单总线）
├── bsp_adc.h / .c            # ADC 驱动（4路模拟输入）
├── bsp_rtc.h / .c            # RTC 驱动（电池备份）
└── bsp_core.h / .c           # Cortex-M7 核心功能（Cache/MPU/NVIC）
```

共 24 个文件（12 个模块对 + bsp_pinmap.h + bsp.h），估计总行数约 3400 行。

---

## 二、模块依赖关系

```
bsp_system ──→ bsp_core (Cache/MPU)
    │
    ├──→ bsp_sdram ──→ bsp_core (EXMC + MPU)
    ├──→ bsp_qspi ──→ bsp_core (OSPI)
    ├──→ bsp_uart ──→ 无外部依赖
    │       ├── USART0 → 调试串口
    │       ├── USART1 → RS-232
    │       ├── UART3  → IO 扩展
    │       └── UART4  → RS-485 数据通道
    ├──→ bsp_i2c ──→ 无外部依赖
    │       ├──→ bsp_touch (I2C + GPIO 中断)
    │       └──→ bsp_oled (I2C)
    ├──→ bsp_spi ──→ 无外部依赖 (SPI2)
    ├──→ bsp_can ──→ 无外部依赖 (CAN2)
    ├──→ bsp_485 ──→ bsp_uart (UART4) + GPIO (PB4 方向控制)
    ├──→ bsp_eth ──→ bsp_core (ENET DMA + MPU)
    ├──→ bsp_sdio ──→ 无外部依赖 (SDIO0)
    ├──→ bsp_lcd ──→ 无外部依赖 (TLI)
    ├──→ bsp_usb ──→ 无外部依赖 (USBHS0/1)
    ├──→ bsp_dht11 ──→ 无外部依赖 (单总线 GPIO)
    ├──→ bsp_adc ──→ 无外部依赖 (ADC01/2)
    └──→ bsp_rtc ──→ 无外部依赖
```

`bsp_led`、`bsp_key`、`bsp_buzzer` 仅依赖原始 GPIO，不引用其他 BSP 模块。

---

## 三、引脚映射命名规范

所有引脚定义集中在 `bsp_pinmap.h`，按外设分组。命名规则：

```
BSP_<外设缩写>_<信号名>_<属性>
```

属性分为：
- `_PORT`：GPIO 端口（如 `GPIOA`）
- `_PIN`：GPIO 引脚号（如 `GPIO_PIN_13`）
- `_AF`：复用功能编号（如 `GPIO_AF_0`）

示例：

```c++
/* ========== 调试接口 ========== */
#define BSP_SWDIO_PORT          GPIOA
#define BSP_SWDIO_PIN           GPIO_PIN_13
#define BSP_SWDCLK_PORT         GPIOA
#define BSP_SWDCLK_PIN          GPIO_PIN_14

/* ========== SD 卡 (SDIO0) ========== */
#define BSP_SDIO_D0_PORT        GPIOC
#define BSP_SDIO_D0_PIN         GPIO_PIN_8
#define BSP_SDIO_D1_PORT        GPIOC
#define BSP_SDIO_D1_PIN         GPIO_PIN_9
#define BSP_SDIO_D2_PORT        GPIOC
#define BSP_SDIO_D2_PIN         GPIO_PIN_10
#define BSP_SDIO_D3_PORT        GPIOC
#define BSP_SDIO_D3_PIN         GPIO_PIN_11
#define BSP_SDIO_CLK_PORT       GPIOC
#define BSP_SDIO_CLK_PIN        GPIO_PIN_12
#define BSP_SDIO_CMD_PORT       GPIOD
#define BSP_SDIO_CMD_PIN        GPIO_PIN_2
#define BSP_SDIO_CD_PORT        GPIOH
#define BSP_SDIO_CD_PIN         GPIO_PIN_2

/* ========== TFT LCD (TLI RGB888) ========== */
/* 红色通道 */
#define BSP_LCD_R3_PORT         GPIOA
#define BSP_LCD_R3_PIN          GPIO_PIN_15
#define BSP_LCD_R4_PORT         GPIOH
#define BSP_LCD_R4_PIN          GPIO_PIN_10
// ... 共 5 条 R 线、6 条 G 线、5 条 B 线、4 条控制线
```

---

## 四、模块统一 API 模式

每个外设模块遵循一致的接口约定：

| API                                   | 说明                     | 是否必须 |
|---------------------------------------|------------------------|------|
| `void bsp_xxx_init(void)`             | 初始化（GPIO、时钟、外设、DMA、中断） | 必须   |
| `void bsp_xxx_deinit(void)`           | 反初始化，释放资源              | 必须   |
| `void bsp_xxx_config(xxx_cfg_t *cfg)` | 配置运行时参数（波特率、模式等）       | 按需   |
| `uint8_t bsp_xxx_is_ready(void)`      | 查询就绪状态                 | 按需   |

### 典型示例 —— bsp_led

```c++
// bsp_led.h
#ifndef BSP_LED_H
#define BSP_LED_H

#include <stdint.h>

/* LED 编号 */
typedef enum {
    BSP_LED1 = 0,       // PD11, 红色
    BSP_LED2,           // PF7,  红色
    BSP_LED3,           // PD4,  红色
    BSP_LED_MAX
} bsp_led_t;

void bsp_led_init(void);
void bsp_led_on(bsp_led_t led);
void bsp_led_off(bsp_led_t led);
void bsp_led_toggle(bsp_led_t led);

#endif
```

---

## 五、系统初始化顺序

初始化必须遵守硬件依赖顺序，尤其是 PMU → PLL 的前后关系。

```c++
void bsp_init(void) {
    /* ===== 阶段 1：核心初始化（无时钟依赖）===== */
    bsp_core_init();            // Cache 使能、MPU 默认配置、NVIC 优先级分组

    /* ===== 阶段 2：电源和时钟 ===== */
    bsp_system_pmu_config();    // SMPS/LDO 模式配置（必须在 PLL 前！）
    bsp_system_clock_config();  // HXTAL 25MHz → PLL → 600MHz AHB → APB 总线时钟

    /* ===== 阶段 3：内存（依赖系统时钟）===== */
    bsp_sdram_init();           // EXMC SDRAM 初始化，含 MPU 非对齐访问区域配置
    bsp_qspi_init();            // OSPI NOR Flash 内存映射模式

    /* ===== 阶段 4：基础外设 ===== */
    bsp_led_init();
    bsp_key_init();
    bsp_buzzer_init();
    bsp_uart_init();            // 尽早初始化，为 printf 调试提供通道
    bsp_rtc_init();
    bsp_adc_init();

    /* ===== 阶段 5：通信外设 ===== */
    bsp_i2c_init();
    bsp_spi_init();
    bsp_can_init();
    bsp_485_init();             // 依赖 UART4（bsp_uart_init 已完成）
    bsp_eth_init();             // 依赖 MPU 配置（bsp_core_init 已完成）
    bsp_sdio_init();

    /* ===== 阶段 6：显示和人机交互 ===== */
    bsp_lcd_init();             // LCD 优先初始化，作为主显示
    bsp_touch_init();           // 依赖 I2C（bsp_i2c_init 已完成）
    bsp_oled_init();            // 依赖 I2C

    /* ===== 阶段 7：外部设备 ===== */
    bsp_usb_init();
    bsp_dht11_init();
}
```

---

## 六、各模块设计要点

### 6.1 bsp_core — Cortex-M7 核心

负责：
- D-Cache / I-Cache 使能
- MPU 默认区域配置
- NVIC 优先级分组

MPU 区域规划：

| 区域       | 地址范围       | 大小    | 内存类型   | Cache     | Buffer   | 用途            |
|----------|------------|-------|--------|-----------|----------|---------------|
| SDRAM    | 0xC0000000 | 32 MB | Normal | WT        | Non-Buff | 允许非对齐访问 SDRAM |
| ENET 描述符 | 用户定义       | —     | Normal | Non-Cache | Non-Buff | 以太网 DMA 一致性   |
| ENET 缓冲区 | 用户定义       | —     | Normal | Non-Cache | Non-Buff | 以太网 DMA 一致性   |

> 注：ENET 的 MPU 区域由 `bsp_eth_init()` 内部注册，`bsp_core` 仅提供默认配置。

### 6.2 bsp_system — 系统时钟

负责：
- PMU SMPS/LDO 模式配置
- 外部晶振使能（HXTAL 25MHz, LXTAL 32.768KHz）
- PLL 锁相环配置（输出 600MHz SYSCLK）
- AHB/APB1/APB2/APB4 总线时钟分频
- `SystemCoreClock` 全局变量更新

关键约束（AN111）：
1. SMPS 配置必须在 PLL 之前
2. PMU 模式必须与外部电路匹配（本项目：SMPS+LDO, SMPS 输出 2.5V → `PMU_SMPS_2V5_SUPPLIES_LDO`）
3. 外设时钟源需确保已稳定才能切换

### 6.3 bsp_uart — 多实例串口

用枚举索引 4 路串口实例：

| 索引 | 名称               | 串口号    | TX 引脚 | RX 引脚 | 用途                         |
|----|------------------|--------|-------|-------|----------------------------|
| 0  | `BSP_UART_DEBUG` | USART0 | PA9   | PA10  | 调试串口（底板 7Pin 排针）           |
| 1  | `BSP_UART_RS232` | USART1 | PD5   | PD6   | RS-232（经 SIT3232EESE）      |
| 2  | `BSP_UART_IO`    | UART3  | PH13  | PH14  | IO 扩展排针 J17                |
| 3  | `BSP_UART_RS485` | UART4  | PB6   | PB12  | RS-485 数据通道（经 SIT3088EESA） |

接口函数：

```c++
void bsp_uart_init(void);
void bsp_uart_send(bsp_uart_id_t id, const uint8_t *data, uint16_t len);
uint16_t bsp_uart_recv(bsp_uart_id_t id, uint8_t *buf, uint16_t max_len);
void bsp_uart_set_baudrate(bsp_uart_id_t id, uint32_t baud);
void bsp_uart_set_rx_callback(bsp_uart_id_t id, void (*cb)(uint8_t));
```

### 6.4 bsp_485 — RS-485 方向自动控制

封装 UART4，透明处理 PB4 方向切换：

| 信号          | MCU 引脚 | 连接                     |
|-------------|--------|------------------------|
| 485_TX (DI) | PB6    | SIT3088EESA 引脚 4       |
| 485_RX (RO) | PB12   | SIT3088EESA 引脚 1       |
| 485_DE/RE   | PB4    | SIT3088EESA 引脚 2+3（短接） |

方向控制逻辑：
- 发送前：`PB4 = 1`（发送使能）
- 发送完成后：利用 UART TC（发送完成）中断 → `PB4 = 0`（切回接收）
- 默认：`PB4 = 0`（接收模式，由上拉电阻保证）

### 6.5 bsp_eth — 以太网 RMII

PHY: LAN8720A-CP-TR, RMII 模式, PHY 地址 0x00, 50MHz 参考时钟由 PHY 输出到 MCU。

RMII 引脚映射（12 个信号 + 2 个 LED）：

| RMII 信号           | MCU 引脚 | PHY 引脚       | 方向         |
|-------------------|--------|--------------|------------|
| ETH0_RMII_REF_CLK | PA1    | REFCLKO (14) | PHY → MCU  |
| ETH0_MDC          | PC1    | MDC (13)     | MCU → PHY  |
| ETH0_MDIO         | PA2    | MDIO (12)    | 双向         |
| ETH0_RMII_CRS_DV  | PA7    | CRS_DV (10)  | PHY → MCU  |
| ETH0_RMII_TX_EN   | PB11   | TXEN (16)    | MCU → PHY  |
| ETH0_RMII_TXD0    | PG13   | TXD0 (17)    | MCU → PHY  |
| ETH0_RMII_TXD1    | PC5    | TXD1 (18)    | MCU → PHY  |
| ETH0_RMII_RXD0    | PC4    | RXD0 (7)     | PHY → MCU  |
| ETH0_RMII_RXD1    | PG14   | RXD1 (8)     | PHY → MCU  |
| ETH0_RESET        | PF6    | NRST (15)    | MCU → PHY  |
| ENET_LED1         | PH6    | LED1 (17)    | 绿灯，470R 限流 |
| ENET_LED2         | PH12   | LED2 (18)    | 黄灯，470R 限流 |

**MPU 配置要点（AN111）**：以太网描述符和收发缓冲区所在内存区域必须通过 MPU 设置为 Non-Bufferable / Non-Cacheable，以确保 DMA 数据一致性。

### 6.6 bsp_sdram — EXMC SDRAM

芯片：IS42S16160J-6BLI (16M × 16bit = 32 MB)

共占用 38 个 MCU 引脚（地址线 13 + Bank 2 + 控制 6 + 数据 16 + 时钟 1），全部为板载直接连接，不经过 BTB。

MPU 要求：将 0xC0000000 起 32 MB 区域配置为 Normal Write-Through，以允许非对齐访问（否则默认 Device 类型会导致 HardFault）。

### 6.7 bsp_qspi — OSPI Flash

芯片：GD25Q64ESIGR (64 Mbit = 8 MB)

| 信号           | MCU 引脚 |
|--------------|--------|
| OSPIM_P0_CSN | PB10   |
| OSPIM_P0_IO0 | PF8    |
| OSPIM_P0_IO1 | PF9    |
| OSPIM_P0_IO2 | PE2    |
| OSPIM_P0_IO3 | PA3    |
| OSPIM_P0_SCK | PA6    |

支持内存映射模式，将 Flash 内容直接映射到 MCU 地址空间供 XIP（片内执行）使用。

### 6.8 bsp_lcd — TFT LCD

接口：FPC_0R5_40P_D, 24 位 RGB888 TLI。所有数据线经 22Ω 终端匹配电阻。

信号分组：
- **红色 R[7:3]**（5 条）：PA15, PH10, PH11, PA8, PG6
- **绿色 G[7:2]**（6 条）：PD3, PC7, PH4, PH15, PG10, PC0
- **蓝色 B[7:3]**（5 条）：PB9, PB8, PB5, PG12, PG11
- **控制**（4 条）：PB3/PIXCLK, PC6/HSYNC, PA4/VSYNC, PF10/DE
- **背光**（1 条）：PG3/BL

### 6.9 bsp_i2c — I2C 主机

用于触摸屏和 OLED 通信：

| I2C 信号  | MCU 引脚 | 连接设备               |
|---------|--------|--------------------|
| I2C_SCL | PH7    | 触摸屏 SCL + OLED SCL |
| I2C_SDA | PH8    | 触摸屏 SDA + OLED SDA |

### 6.10 bsp_sdio — SD 卡

接口：SDIO0 4-bit 模式，TF 卡槽

| 信号        | MCU 引脚  | 上拉    |
|-----------|---------|-------|
| SDIO0_D0  | PC8     | 4.7KΩ |
| SDIO0_D1  | PC9     | 4.7KΩ |
| SDIO0_D2  | PC10    | 4.7KΩ |
| SDIO0_D3  | PC11    | 4.7KΩ |
| SDIO0_CLK | PC12    | 4.7KΩ |
| SDIO0_CMD | PD2     | 4.7KΩ |
| SDIO0_CD  | PH2 (?) | —     |

数据线串 120Ω 终端电阻，时钟线串 80.6Ω。

### 6.11 bsp_can — CAN 总线

| 信号      | MCU 引脚 | 收发器引脚                   |
|---------|--------|-------------------------|
| CAN2_TX | PD13   | SIT1042AQT/3 引脚 1 (TXD) |
| CAN2_RX | PD12   | SIT1042AQT/3 引脚 4 (RXD) |

收发器 SIT1042AQT/3，CANH/CANL 之间 120Ω 终端电阻。支持 CAN-FD。

### 6.12 bsp_adc — 模拟输入

4 路模拟输入，全部引出至底板：

| 输入        | MCU 引脚 | 来源      |
|-----------|--------|---------|
| ADC01_IN0 | PA0_C  | 底板排针 J4 |
| ADC01_IN1 | PA1_C  | 底板排针 J4 |
| ADC2_IN0  | PC2_C  | 底板排针 J4 |
| ADC2_IN1  | PC3_C  | 底板排针 J4 |

### 6.13 其他简单模块

| 模块        | 引脚             | 说明                 |
|-----------|----------------|--------------------|
| LED1–LED3 | PD11, PF7, PD4 | 低电平点亮，1KΩ 限流       |
| KEY1/WKUP | PA0            | 唤醒功能，10KΩ 上拉，低电平有效 |
| KEY2      | PB6            | 用户按键，10KΩ 上拉，低电平有效 |
| KEY3      | PD7            | 用户按键，10KΩ 上拉，低电平有效 |
| 蜂鸣器       | PB0            | NPN 三极管驱动，高电平有效    |
| RTC       | VBAT           | CR1220 纽扣电池备份      |

---

## 七、Cache 与 DMA 一致性规则

所有使用 DMA 的模块（ENET、SDIO、SPI、ADC）必须遵守：

1. **缓冲区对齐**：地址 32 字节对齐，大小为 32 字节的整数倍
2. **DMA 写入 SRAM → CPU 读取**：DMA 完成后调用 `SCB_InvalidateDCache_by_Addr(addr, size)` 使 Cache 行无效
3. **CPU 写入 Cache → DMA 读取 SRAM**：启动 DMA 前调用 `SCB_CleanDCache_by_Addr(addr, size)` 将 Cache 刷新到 SRAM

Cache 行大小 = 32 字节，所有操作以行为单位。

---

## 八、实施顺序

按依赖关系和复杂度分批进行：

| 批次 | 模块                                                        | 优先级 | 说明                 |
|----|-----------------------------------------------------------|-----|--------------------|
| 1  | `bsp_pinmap.h`, `bsp_core`, `bsp_system`                  | 最高  | 基础，所有模块依赖          |
| 2  | `bsp_led`, `bsp_key`, `bsp_buzzer`, `bsp_uart`, `bsp_rtc` | 高   | 简单外设，快速验证 GPIO 和时钟 |
| 3  | `bsp_sdram`, `bsp_qspi`                                   | 高   | 内存，解锁大容量存储         |
| 4  | `bsp_i2c`, `bsp_spi`, `bsp_can`, `bsp_485`, `bsp_sdio`    | 中   | 通信，依赖基础外设          |
| 5  | `bsp_lcd`, `bsp_touch`, `bsp_oled`                        | 中   | 显示，依赖 I2C          |
| 6  | `bsp_eth`, `bsp_usb`, `bsp_dht11`, `bsp_adc`              | 低   | 复杂外设和传感器           |
