# system_stm32f10x.c/h — 系统时钟初始化文件

> 芯片上电后、main 函数之前，负责把系统时钟从默认的 8 MHz 内部振荡器（HSI）切换到目标频率（通常是 72 MHz）——是启动流程中的"电工"角色

---

## 1. 大白话解释

### 延续启动文件的比喻

在 [启动文件.md](./启动文件.md) 里，我们把启动文件比作"上电前帮你收拾厨房的隐形帮手"。现在细拆一下：

| 角色 | 比喻 | 对应文件 |
|------|------|---------|
| **装修队** | 铺水电、搭框架 | `startup_stm32f10x_md.s`（启动文件） |
| **电工** ⭐ | 负责把电压调到正确的值 | `system_stm32f10x.c` + `.h` ← **这本文件** |
| **家具说明书** | 告诉你每个开关长什么样 | `stm32f10x.h` |
| **你入住后的生活** | 你写的 `main()` 函数 | 用户代码 |

> **类比**：芯片出厂默认用 8 MHz 的内部振荡器（HSI），就像手机出厂默认开省电模式。`system_stm32f10x.c` 的工作就是在你写 main 之前，把时钟从"省电模式"超频到"满血 72 MHz"——而且全程不需要你动手，启动文件自动调用它。

### .h 和 .c 各管什么？

| 文件 | 类比 | 具体职责 |
|------|------|---------|
| **.h 文件** | **菜单**——告诉外面有什么菜 | 声明 `SystemInit()` 函数、`SystemCoreClock` 变量等接口 |
| **.c 文件** | **后厨**——真正炒菜的地方 | 实现时钟切换的全部代码（操作 RCC 寄存器） |

---

## 2. 专业解释

### 2.1 文件定位

这对文件属于 **CMSIS（Cortex Microcontroller Software Interface Standard）的 Device 层**，在标准外设库中位于 `CMSIS/Device/ST/STM32F10x/` 目录下。

它在启动流程中的位置（引用 [启动文件.md](./启动文件.md) 第 40-50 行的结构）：

```
芯片上电
    ↓
startup_stm32f10x_md.s  ← 启动文件（汇编），最先执行
    │
    │  ┌──────────────────────────────────────────────┐
    ├──→ SystemInit()  ← system_stm32f10x.c 实现的  │  ← ⭐ 这里
    │  └──────────────────────────────────────────────┘
    ↓
__main                  ← C 库函数，初始化全局变量
    ↓
main()                  ← 你写的用户程序
```

`.c` 文件开头（源码第 9-27 行）自己也说明了定位：

> This file provides two functions and one global variable to be called from user application:
> - `SystemInit()`: ... This function is called at startup just after reset and before branch to main program.
> - `SystemCoreClock` variable: Contains the core clock (HCLK)...
> - `SystemCoreClockUpdate()`: Updates the variable `SystemCoreClock`...

### 2.2 .h 文件的内容（三样东西）

```c
// system_stm32f10x.h 第 53、79、80 行
extern uint32_t SystemCoreClock;          // 全局变量：存储当前 CPU 主频
extern void SystemInit(void);             // 函数声明：系统初始化（重置后调用）
extern void SystemCoreClockUpdate(void);  // 函数声明：时钟更新（动态切换后调用）
```

就这三样——**短小精悍**，只对外暴露必要的接口。

### 2.3 .c 文件的内容（五大部分）

#### ① 条件编译宏——选择目标频率

源码第 106-116 行：

```c
#if defined (STM32F10X_LD_VL) || (defined STM32F10X_MD_VL) || (defined STM32F10X_HD_VL)
/* #define SYSCLK_FREQ_HSE    HSE_VALUE */
 #define SYSCLK_FREQ_24MHz  24000000
#else
/* #define SYSCLK_FREQ_HSE    HSE_VALUE */
/* #define SYSCLK_FREQ_24MHz  24000000 */
/* #define SYSCLK_FREQ_36MHz  36000000 */
/* #define SYSCLK_FREQ_48MHz  48000000 */
/* #define SYSCLK_FREQ_56MHz  56000000 */
#define SYSCLK_FREQ_72MHz  72000000    // ← 默认取消注释，即 72 MHz
#endif
```

- **取消注释不同的行**，就可以在 24 / 36 / 48 / 56 / 72 MHz 之间切换
- 最常用的是 `SYSCLK_FREQ_72MHz`（STM32F103C8 的最高主频）
- 注意：Value Line（`_VL`）型号最高只能到 24 MHz

#### ② `SystemCoreClock` 变量初始化

源码第 151-165 行，根据上面定义的宏，在**编译时**确定初始值：

```c
#ifdef SYSCLK_FREQ_HSE
  uint32_t SystemCoreClock = SYSCLK_FREQ_HSE;        // 8 MHz 或 25 MHz
#elif defined SYSCLK_FREQ_72MHz
  uint32_t SystemCoreClock = SYSCLK_FREQ_72MHz;      // 72 MHz
#elif defined SYSCLK_FREQ_48MHz
  uint32_t SystemCoreClock = SYSCLK_FREQ_48MHz;      // 48 MHz
// ...
#else
  uint32_t SystemCoreClock = HSI_VALUE;               // 8 MHz（默认 HSI）
#endif
```

`SystemCoreClock` 是一个全局变量，用户代码可以直接读取它来知道当前 CPU 跑多快，例如配置 SysTick 定时器时就需要它。

#### ③ `SystemInit()`——核心函数

源码第 212-269 行。由启动文件的 `Reset_Handler` 调用，执行流程：

```
① 复位 RCC 寄存器到默认状态
    │  RCC->CR  |= 0x00000001   // 开启 HSI
    │  RCC->CFGR &= 0xF8FF0000  // 复位时钟分频配置
    │  RCC->CR  &= 0xFEF6FFFF   // 关闭 HSE、PLL
    ↓
② 禁用所有时钟中断
    │  RCC->CIR = 0x009F0000    // 清除所有 RCC 中断标志
    ↓
③ 调用 SetSysClock() 配置 PLL
    │  SetSysClock();           // 根据目标频率宏走不同的配置分支
    ↓
④ 设置向量表偏移
    │  SCB->VTOR = FLASH_BASE | 0x0;  // 向量表在 Flash 起始地址
```

```c
// 精简后的 SystemInit() 核心骨架（源码第 212-269 行）
void SystemInit (void)
{
    /* ① 复位 RCC 时钟配置到默认状态 */
    RCC->CR |= (uint32_t)0x00000001;       // 开启 HSI
    RCC->CFGR &= (uint32_t)0xF8FF0000;     // 复位 SW、HPRE 等分频位
    RCC->CR &= (uint32_t)0xFEF6FFFF;       // 关闭 HSEON、PLLON
    RCC->CFGR &= (uint32_t)0xFF80FFFF;     // 复位 PLL 相关位

    /* ② 禁用所有 RCC 中断 */
    RCC->CIR = 0x009F0000;

    /* ③ 由 SetSysClock() 配置系统时钟 */
    SetSysClock();

    /* ④ 设置向量表偏移（默认在 Flash 起始） */
    SCB->VTOR = FLASH_BASE | VECT_TAB_OFFSET;
}
```

#### ④ `SetSysClockTo72()`——最常用的时钟配置函数

以默认的 `SYSCLK_FREQ_72MHz` 为例，`SetSysClock()` 内部调用 `SetSysClockTo72()`（源码第 987-1080 行），步骤为：

| 步骤 | 操作 | 代码 |
|------|------|------|
| 1 | 开启外部晶振 HSE（8 MHz） | `RCC->CR \|= RCC_CR_HSEON` |
| 2 | 等待 HSE 就绪（超时检测） | `while((RCC->CR & RCC_CR_HSERDY) == 0)` |
| 3 | 开启 Flash 预取缓冲 | `FLASH->ACR \|= FLASH_ACR_PRFTBE` |
| 4 | 设置 Flash 等待周期为 2（72 MHz 必须） | `FLASH->ACR \|= FLASH_ACR_LATENCY_2` |
| 5 | HCLK = SYSCLK（不分频） | `RCC->CFGR \|= RCC_CFGR_HPRE_DIV1` |
| 6 | PCLK2 = HCLK（APB2 不分频） | `RCC->CFGR \|= RCC_CFGR_PPRE2_DIV1` |
| 7 | PCLK1 = HCLK/2（APB1 二分频） | `RCC->CFGR \|= RCC_CFGR_PPRE1_DIV2` |
| 8 | PLL 配置：HSE × 9 = 72 MHz | `RCC->CFGR \|= RCC_CFGR_PLLMULL9` |
| 9 | 开启 PLL，等待锁定 | `RCC->CR \|= RCC_CR_PLLON` → 等待 `RCC_CR_PLLRDY` |
| 10 | 切换系统时钟源为 PLL | `RCC->CFGR \|= RCC_CFGR_SW_PLL` |

#### ⑤ `SystemCoreClockUpdate()`——运行时的时钟更新

源码第 306 行起。如果在程序运行时动态改变了时钟配置（例如降频省电），调用此函数可以**根据当前 RCC 寄存器的值重新计算实际频率**，更新 `SystemCoreClock` 变量。

```c
// 典型使用场景
void enter_low_power_mode(void)
{
    // 切换到 HSI，降低功耗
    RCC->CFGR &= ~RCC_CFGR_SW;        // 清除 SW 位
    RCC->CFGR |= RCC_CFGR_SW_HSI;     // 选择 HSI 为系统时钟
    SystemCoreClockUpdate();           // ← 更新 SystemCoreClock
    // 现在 SystemCoreClock = 8 MHz
}
```

### 2.4 可选配置项

源码第 118-129 行提供了一些可选宏：

| 宏 | 作用 | 默认 |
|----|------|------|
| `DATA_IN_ExtSRAM` | 启用外部 SRAM（HD/XL 型号专用） | 注释掉 |
| `VECT_TAB_SRAM` | 向量表重定位到 SRAM（IAP Bootloader 场景） | 注释掉 |
| `VECT_TAB_OFFSET` | 向量表基地址偏移（必须为 0x200 的倍数） | `0x0` |

### 2.5 时钟树图（F103 标准配置）

```
HSE 外部晶振                PLL (×9)
  8 MHz ─────────→ ┌──────┐    72 MHz
                   │ PLL  │──────→ SYSCLK
                   └──────┘        │
                                   ├── AHB (/1) ──→ HCLK (72 MHz) → CPU、Flash、DMA
                                   │                ├── APB2 (/1) ─→ PCLK2 (72 MHz) → GPIO、USART1、SPI1、TIM1
                                   │                └── APB1 (/2) ─→ PCLK1 (36 MHz) → USART2~5、TIM2~7、I2C、SPI2
                                   │
                                   └── Flash 等待周期 = 2 个等待状态
```

> 注意：APB1 最高只能到 36 MHz，所以必须二分频；APB2 可以跑到 72 MHz。

### 2.6 与启动文件的关系

[启动文件.md](./启动文件.md) 第 100-111 行展示了 `Reset_Handler` 的汇编代码：

```asm
Reset_Handler   PROC
                IMPORT  SystemInit
                LDR     R0, =SystemInit     ; 加载 SystemInit 地址
                BLX     R0                  ; 调用 SystemInit()
                IMPORT  __main
                LDR     R0, =__main
                BX      R0                  ; 跳转到 __main
                ENDP
```

配合关系总结：

```
芯片上电 → Reset_Handler（启动文件）
              │
              ├─→ 设置 SP、配置向量表
              ├─→ 调用 SystemInit()     ← 来自 system_stm32f10x.c
              │       └─→ 配置 HSE → PLL → 72 MHz
              └─→ 跳转 __main → main()
```

**没有 `system_stm32f10x.c`**，启动文件也能跑完流程，但芯片会一直运行在默认的 8 MHz HSI——就像买了台高性能电脑但一直在用省电模式。

### 2.7 与 stm32f10x.h 的关系

在 [stm32f10x.h头文件.md](./stm32f10x.h头文件.md) 中已经提到过：

| 文件 | 职责 |
|------|------|
| **stm32f10x.h** | 定义外设寄存器结构体、基地址宏、中断编号——**管"外设长什么样"** |
| **system_stm32f10x.h** | 声明系统时钟初始化函数——**管"系统时钟怎么跑"** |

注意 `system_stm32f10x.c` 第 65 行 `#include "stm32f10x.h"`——它依赖于 `stm32f10x.h` 中定义的 `RCC`、`FLASH` 等寄存器结构体和 `HSE_VALUE` 等宏。

### 2.8 常见问题

**Q：换了不同频率的外部晶振怎么办？**

修改 `stm32f10x.h` 中的 `HSE_VALUE` 宏：

```c
// stm32f10x.h（默认 8 MHz）
#define HSE_VALUE    ((uint32_t)8000000)

// 如果你的板子用的是 12 MHz 晶振，改为
#define HSE_VALUE    ((uint32_t)12000000)
```

同时调整 `SetSysClockTo72()` 中的 PLL 倍频系数，使 `HSE × PLL_MUL = 72 MHz`。

**Q：想跑 48 MHz 而不是 72 MHz？**

1. 注释掉 `#define SYSCLK_FREQ_72MHz  72000000`
2. 取消注释 `#define SYSCLK_FREQ_48MHz  48000000`
3. 重新编译

`SetSysClock()` 会自动根据宏选择对应的 `SetSysClockTo48()` 函数。

**Q：`SystemCoreClockUpdate()` 什么时候需要手动调用？**

- 如果在 `main()` 中通过直接操作 RCC 寄存器改变了时钟源或分频系数
- 例如进入低功耗模式后切回高速模式
- 一般情况下不需要——启动时 `SystemInit()` 已经设好了，`SystemCoreClock` 在编译时就有初始值

---

## 3. 一句话总结

| 文件 | 一句话 |
|------|--------|
| `system_stm32f10x.h` | 对外公告：声明 `SystemInit`、`SystemCoreClock` 等接口 |
| `system_stm32f10x.c` | 实际干活：上电后把系统时钟从 8 MHz 配置到目标频率（默认 72 MHz） |

> **一句话总结**：这对文件负责在 main 之前把芯片的"心脏"调好——从默认的 8 MHz 慢速模式提速到满血 72 MHz，让后面的代码能全速运行。

### 关联链接

- [启动文件.md](./启动文件.md) — 启动文件调用 `SystemInit` 的完整流程
- [stm32f10x.h头文件.md](./stm32f10x.h头文件.md) — `HSE_VALUE` 等宏的来源；与 system 文件的职责对比
- [寄存器与存储器映射.md](./寄存器与存储器映射.md) — RCC 寄存器结构体映射原理
