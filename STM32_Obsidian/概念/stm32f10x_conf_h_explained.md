# stm32f10x_conf.h 文件说明

## 概述

`stm32f10x_conf.h` 是 **STM32F10x 标准外设库（Standard Peripheral Library, SPL）** 中的一个关键配置文件，由 ST（意法半导体）提供。它位于 SPL 项目的头文件目录下（通常与 `stm32f10x.h` 同级），用于**集中管理外设模块的启用/禁用以及一些编译期配置**。

## 主要作用

### 1. 外设模块的选择性包含（`#include`）

SPL 为每个外设提供了一个独立的驱动头文件（如 `stm32f10x_gpio.h`、`stm32f10x_usart.h`、`stm32f10x_adc.h` 等）。如果直接包含所有头文件，会导致编译时间变长且占用不必要的代码空间。`stm32f10x_conf.h` 允许用户通过 **注释/取消注释 `#include` 行** 来精确选择项目中实际使用的外设，编译器只会编译和链接被包含的外设模块。

### 2. 断言（Assert）控制

文件末尾通常定义了 `assert_param(expr)` 宏：

```c
#ifdef USE_FULL_ASSERT
#define assert_param(expr) ((expr) ? (void)0U : assert_failed((uint8_t *)__FILE__, __LINE__))
void assert_failed(uint8_t *file, uint32_t line);
#else
#define assert_param(expr) ((void)0U)
#endif
```

- 当定义了 `USE_FULL_ASSERT` 宏时，`assert_param()` 会在每个外设函数的参数入口检查参数合法性，非法时调用 `assert_failed()` 进入用户自定义的错误处理。
- 未定义时断言宏为空，不产生任何运行时开销。

### 3. 系统时钟与外部晶振频率定义

通常还会包含配置项目所用的 HSE（外部高速晶振）频率值：

```c
#if !defined  HSE_VALUE
#define HSE_VALUE    ((uint32_t)8000000)  // 默认 8 MHz
#endif
```

这个值会被 SPL 中的系统时钟初始化函数（如 `SystemInit()`）用来正确配置 PLL 倍频系数。

## 典型文件结构

```c
/**
  ******************************************************************************
  * @file    stm32f10x_conf.h
  * @brief   Library configuration file
  ******************************************************************************
  */

/* Define to prevent recursive inclusion */
#ifndef __STM32F10X_CONF_H
#define __STM32F10X_CONF_H

#ifdef __cplusplus
extern "C" {
#endif

/* Includes ------------------------------------------------------------------*/
/* 注释掉不需要的外设模块 */
#include "stm32f10x_adc.h"
#include "stm32f10x_bkp.h"
// #include "stm32f10x_can.h"
#include "stm32f10x_dma.h"
#include "stm32f10x_exti.h"
#include "stm32f10x_flash.h"
#include "stm32f10x_gpio.h"
// #include "stm32f10x_i2c.h"
#include "stm32f10x_rcc.h"
#include "stm32f10x_tim.h"
#include "stm32f10x_usart.h"
// ... 其他外设

/* Exported types ------------------------------------------------------------*/
/* Exported constants --------------------------------------------------------*/

/* 系统时钟相关配置 */
#if !defined  HSE_VALUE
#define HSE_VALUE    ((uint32_t)8000000)
#endif

/* Exported macro ------------------------------------------------------------*/
#ifdef USE_FULL_ASSERT
#define assert_param(expr) ((expr) ? (void)0U : assert_failed((uint8_t *)__FILE__, __LINE__))
void assert_failed(uint8_t *file, uint32_t line);
#else
#define assert_param(expr) ((void)0U)
#endif

#ifdef __cplusplus
}
#endif

#endif /* __STM32F10X_CONF_H */
```

## 使用方式

1. 在 **IDE 预处理器定义** 或编译器命令行中添加或不添加 `USE_FULL_ASSERT` 来决定是否启用参数断言。
2. 根据项目实际使用的外设，在 `stm32f10x_conf.h` 中注释掉未使用的外设头文件。
3. 如果外部晶振不是默认的 8 MHz，可以在此处覆盖 `HSE_VALUE`。

## 为什么不直接包含 stm32f10x.h？

`stm32f10x.h` 是芯片级定义（寄存器地址、中断号、数据类型等），它自身并不会自动包含外设驱动头文件。它会在编译时查找 `stm32f10x_conf.h` 并包含它：

```c
// 在 stm32f10x.h 中：
#ifdef USE_STDPERIPH_DRIVER
#include "stm32f10x_conf.h"
#endif
```

因此，通常需要在编译器中添加宏定义 `USE_STDPERIPH_DRIVER`，才能使 `stm32f10x.h` 自动包含 `stm32f10x_conf.h`，进而包含具体的外设驱动头文件。

## 总结

| 职责         | 说明                                                       |
| ------------ | ---------------------------------------------------------- |
| 外设选择     | 通过 `#include` 列表控制编译哪些外设驱动模块               |
| 断言控制     | 通过 `USE_FULL_ASSERT` 宏启用/禁用参数运行时检查           |
| 晶振频率配置 | 定义 `HSE_VALUE` 供系统时钟初始化使用                      |
| 集合入口     | 在 `USE_STDPERIPH_DRIVER` 宏开启时被 `stm32f10x.h` 自动包含 |

这是一个**单文件配置入口**，在所有 SPL 项目中都必须根据硬件和功能裁剪来编辑它。
