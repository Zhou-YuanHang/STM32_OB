# core_cm3.c/h — Cortex-M3 内核支持文件

> ARM 官方提供的 Cortex-M3 内核"操作手册"——定义 NVIC、SCB、SysTick 等内核外设的寄存器结构和操作函数，是所有 Cortex-M3 芯片（不限 STM32）的公共底层接口

---

## 1. 大白话解释

### 延续比喻体系

在前几篇文档中，我们搭好了这样一个体系：

| 角色 | 比喻 | 对应文件 |
|------|------|---------|
| **装修队** | 铺水电、搭框架 | `startup_stm32f10x_md.s`（启动文件） |
| **电工** | 把电压调到 72 MHz | `system_stm32f10x.c/h` |
| **家具说明书** | 每个外设（GPIO、USART 等）的用法 | `stm32f10x.h` |
| **国家建筑规范** ⭐ | **所有房子都遵守的电路标准** | `core_cm3.h` ← **这本文件** |

> `core_cm3.h` 不是 ST 写的，而是 **ARM 官方**写的。它规定了 Cortex-M3 这个 CPU 内核本身怎么用——怎么开关中断、怎么设置系统定时器、怎么读写内核寄存器。不管你用 STM32、NXP、还是 TI 的 Cortex-M3 芯片，这套接口都通用。

### 外设 vs 内核

| 谁管 | 管什么 | 文件 |
|------|--------|------|
| **内核**（Cortex-M3） | NVIC 中断控制器、SysTick 系统定时器、SCB 系统控制块…… | `core_cm3.h` |
| **外设**（STM32） | GPIO、USART、TIM、ADC…… | `stm32f10x.h` |

打个比方：
- `core_cm3.h` = 告诉你**电灯开关怎么接火线零线**（内核规则）
- `stm32f10x.h` = 告诉你**你家开关装在哪个位置、什么颜色**（具体芯片的实现）

### 一句话大白话

> 如果 `stm32f10x.h` 是"外设说明书"，那 `core_cm3.h` 就是"CPU 说明书"。它教你如何开关中断、如何用系统滴答定时器（SysTick）、如何读写内核寄存器（NVIC、SCB），而且这套说明书适用于**所有** Cortex-M3 芯片——不限于 STM32。

---

## 2. 专业解释

### 2.1 文件定位

`core_cm3.h` 属于 **CMSIS 的 Core 层**（`CMSIS/Include/` 目录），由 **ARM 官方**维护，不是 ST 写的。

CMSIS 三层架构：

```
CMSIS Core 层（ARM 官方）
  core_cm3.h  ← ⭐ 当前文件
      │
      │  被 stm32f10x.h 第 478 行包含
      ▼
CMSIS Device 层（ST 官方）
  stm32f10x.h（外设寄存器定义）
  system_stm32f10x.h（系统时钟声明）
      │
      │  被用户代码包含
      ▼
外设库层（ST 官方）
  stm32f10x_gpio.h
  stm32f10x_usart.h
  stm32f10x_tim.h
  ……
```

**`core_cm3.h` 是整个链条的**最底层**，所有上层的文件最终都依赖它提供的内核定义。**

> 注意：`core_cm3.h` 不在你的工程文件夹里——它是 CMSIS 标准库的一部分，由编译器/IDE 在搜索路径中找到。`stm32f10x.h` 第 478 行用 `#include "core_cm3.h"` 把它拉进来，编译器会在 CMSIS 的 Include 目录下找到它。

### 2.2 .h 文件的五大内容板块

#### ① volatile 限定符宏（`__IO` / `__I` / `__O`）

`core_cm3.h` 定义了三个核心宏，被整个标准库广泛使用：

```c
#define __I     volatile const    /*!< 只读：寄存器只允许读，写无效 */
#define __O     volatile          /*!< 只写：寄存器只允许写，读无效 */
#define __IO    volatile          /*!< 读写：既可读又可写 */
```

**为什么要加 `volatile`？**
- 硬件寄存器的值可能被 CPU 之外的因素改变（外设自行更新）
- 没有 `volatile`，编译器可能优化掉看似"多余"的读写操作
- 详细原理见 [寄存器与存储器映射.md](./寄存器与存储器映射.md) 中关于 `volatile` 的说明

**在 `stm32f10x.h` 中的实际使用**（第 495-517 行）：

```c
typedef __IO int32_t  vs32;      // 可读写的 volatile 有符号 32 位变量
typedef __I  int32_t  vsc32;     // 只读的 volatile 有符号 32 位变量
typedef __IO uint32_t vu32;      // 可读写的 volatile 无符号 32 位变量
// ...
```

所有外设寄存器结构体（如 `GPIO_TypeDef`）的成员都是 `volatile uint32_t`，正是依靠这些宏。

#### ② NVIC（嵌套向量中断控制器）操作函数

NVIC 是 Cortex-M3 内核的中断控制器，`core_cm3.h` 提供了一组 **`static inline`** 函数来操作它：

| 函数 | 作用 | 使用示例 |
|------|------|---------|
| `NVIC_EnableIRQ(IRQn)` | 使能某个外设中断 | `NVIC_EnableIRQ(USART1_IRQn)` |
| `NVIC_DisableIRQ(IRQn)` | 禁用某个外设中断 | `NVIC_DisableIRQ(TIM2_IRQn)` |
| `NVIC_SetPriority(IRQn, priority)` | 设置中断优先级 | `NVIC_SetPriority(USART1_IRQn, 1)` |
| `NVIC_GetPriority(IRQn)` | 读取当前优先级 | `prio = NVIC_GetPriority(USART1_IRQn)` |
| `NVIC_SetPriorityGrouping(group)` | 设置优先级分组（抢占/子优先级分配） | `NVIC_SetPriorityGrouping(0x05)` |
| `NVIC_SystemReset()` | 系统软件复位 | `NVIC_SystemReset()` |

**关键特点：**
- 这些函数都是 `static inline`，编译后直接展开为几条寄存器操作指令，**没有函数调用开销**
- 它们依赖 `stm32f10x.h` 中定义的 `IRQn_Type` 枚举（第 167-472 行）和 `__NVIC_PRIO_BITS`（第 160 行，值为 4）

```c
// NVIC 函数的典型实现（伪代码，真实实现在 core_cm3.h 中）
static inline void NVIC_EnableIRQ(IRQn_Type IRQn)
{
    NVIC->ISER[((uint32_t)(IRQn) >> 5)] = (1 << ((uint32_t)(IRQn) & 0x1F));
}
// 原理：将中断号写入 ISER（中断设置使能寄存器）对应的位
```

#### ③ SCB（系统控制块）寄存器结构体

`SCB_Type` 结构体定义了 Cortex-M3 的系统控制块寄存器：

```c
typedef struct {
    volatile uint32_t CPUID;   // 0x00 — CPU ID 寄存器
    volatile uint32_t ICSR;    // 0x04 — 中断控制和状态寄存器
    volatile uint32_t VTOR;    // 0x08 — 向量表偏移寄存器 ← ⭐ 最常用
    volatile uint32_t AIRCR;   // 0x0C — 应用中断和复位控制寄存器
    volatile uint32_t SCR;     // 0x10 — 系统控制寄存器
    volatile uint32_t CCR;     // 0x14 — 配置和控制寄存器
    // ...
} SCB_Type;

#define SCB ((SCB_Type *) SCB_BASE)  // 将 SCB 指针指向内核地址
```

**`SCB->VTOR` 的实际用途**（引用 [system_stm32f10x系统文件.md](./system_stm32f10x系统文件.md)）：

```c
// system_stm32f10x.c 第 265 行
SCB->VTOR = FLASH_BASE | VECT_TAB_OFFSET;
```

这行代码告诉 CPU：中断向量表在 Flash 的起始地址（偏移 `VECT_TAB_OFFSET`）。在做 Bootloader（IAP）时，通过修改 `VTOR` 可以把向量表重定位到 SRAM 或其他地址，实现程序跳转。

**`SCB->AIRCR` 的应用**：写入特定序列可以触发系统复位：

```c
#define SCB_AIRCR_VECTKEY ((uint32_t)0x05FA << 16)
SCB->AIRCR = SCB_AIRCR_VECTKEY | SCB_AIRCR_SYSRESETREQ;  // 系统复位
```

#### ④ SysTick 定时器结构体和配置函数

`SysTick_Type` 结构体：

```c
typedef struct {
    volatile uint32_t CTRL;    // 0x00 — 控制和状态寄存器
    volatile uint32_t LOAD;    // 0x04 — 重装载值寄存器
    volatile uint32_t VAL;     // 0x08 — 当前值寄存器
    volatile uint32_t CALIB;   // 0x0C — 校准寄存器
} SysTick_Type;
```

**`SysTick_Config()`** — 一行代码配置系统滴答定时器：

```c
static inline uint32_t SysTick_Config(uint32_t ticks)
{
    if (ticks > SysTick_LOAD_RELOAD_Msk)  return 1;  // 超范围
    SysTick->LOAD  = ticks - 1;           // 设置重载值
    NVIC_SetPriority(SysTick_IRQn, (1 << __NVIC_PRIO_BITS) - 1); // 最低优先级
    SysTick->VAL   = 0;                   // 清空计数器
    SysTick->CTRL  = SysTick_CTRL_CLKSOURCE_Msk |
                     SysTick_CTRL_TICKINT_Msk   |
                     SysTick_CTRL_ENABLE_Msk;    // 使能，开中断
    return 0;
}
```

**典型用法**——产生 1ms 系统时钟节拍：

```c
// 在 main 函数开头调用
SysTick_Config(SystemCoreClock / 1000);  // 每 1ms 触发一次 SysTick 中断
```

- `SystemCoreClock` 来自 [system_stm32f10x.h](./system_stm32f10x系统文件.md)，值为 72,000,000
- `72,000,000 / 1000 = 72,000`，即每计数 72,000 次产生一次中断
- 中断处理函数是 `SysTick_Handler()`，需要在用户代码中实现

#### ⑤ 内核寄存器访问的内联汇编函数

`core_cm3.h` 用 `__ASM` 内联汇编实现了最底层的 CPU 控制函数：

| 函数 | 作用 | 汇编指令 |
|------|------|---------|
| `__enable_irq()` | 开启全局中断 | `CPSIE I` |
| `__disable_irq()` | 关闭全局中断 | `CPSID I` |
| `__get_PRIMASK()` | 读取当前中断屏蔽状态 | `MRS R0, PRIMASK` |
| `__set_PRIMASK(value)` | 设置中断屏蔽寄存器 | `MSR PRIMASK, R0` |
| `__NOP()` | 空操作（延时一个周期） | `NOP` |
| `__WFI()` | 等待中断（进入休眠，中断唤醒） | `WFI` |
| `__WFE()` | 等待事件（进入休眠，事件唤醒） | `WFE` |

**使用场景举例：**

```c
// 进入临界区（关中断）
__disable_irq();
// ……执行不可被打断的代码……
// 退出临界区（开中断）
__enable_irq();

// 进入休眠模式等待中断
__WFI();

// 精确延时（NOP 延时一个指令周期）
__NOP();
__NOP();
```

### 2.3 .c 文件的内容

`core_cm3.c` 内容极少，主要提供 **ITM（Instrumentation Trace Macrocell）** 调试通道函数：

```c
// 通过 SWO 引脚输出一个字符（用于 printf 重定向到底层）
uint32_t ITM_SendChar(uint32_t ch)
{
    if ((ITM->TCR & ITM_TCR_ITMENA_Msk) &&            // ITM 已使能
        (ITM->TER & 1UL))                              // ITM 通道 0 已使能
    {
        while (ITM->PORT[0].u32 == 0);                 // 等待发送缓冲空
        ITM->PORT[0].u8 = (uint8_t)ch;                // 发送字符
    }
    return ch;
}
```

**一般工程中不需要添加 `core_cm3.c`**，除非你打算用 SWO 引脚做 `printf` 调试输出。

### 2.4 依赖关系图

```
core_cm3.h（ARM 官方，Cortex-M3 通用）
    │
    │  依赖 stm32f10x.h 提供的配置：
    │  ├─ IRQn_Type          ← stm32f10x.h 第 167 行（中断号枚举）
    │  ├─ __NVIC_PRIO_BITS   ← stm32f10x.h 第 160 行（= 4 位优先级）
    │  ├─ __MPU_PRESENT      ← stm32f10x.h 第 156-159 行（MPU 是否存在）
    │  └─ __Vendor_SysTickConfig ← stm32f10x.h 第 161 行（= 0，使用标准 SysTick）
    │
    │  提供（被 stm32f10x.h 及用户代码使用）：
    │  ├─ __IO / __I / __O   → volatile 限定符宏
    │  ├─ NVIC_Type          → NVIC 寄存器操作函数
    │  ├─ SCB_Type           → SCB 系统控制块
    │  ├─ SysTick_Type       → SysTick 系统定时器
    │  ├─ ITM_Type           → ITM 调试通道
    │  └─ 内核汇编函数       → __enable_irq / __disable_irq / __WFI 等
    │
    ▼
stm32f10x.h 第 478 行 #include "core_cm3.h"
    │
    ▼
用户代码和库文件
```

### 2.5 与已有文件的关系

| 文件 | 谁管 | 层级 | 来源 |
|------|------|------|------|
| **core_cm3.h** | Cortex-M3 **内核**的寄存器和操作函数 | CMSIS Core 层 | ARM 官方 |
| **stm32f10x.h** | STM32F10x **外设**的寄存器定义 | CMSIS Device 层 | ST 官方 |
| **system_stm32f10x.c/h** | STM32F10x 的**系统时钟**配置 | CMSIS Device 层 | ST 官方 |
| **startup_stm32f10x_md.s** | 芯片**启动**和向量表 | 设备启动层 | ST 官方 |

层级关系总结：

```
core_cm3.h（ARM 内核规范）
    ↑ 提供底层内核定义
stm32f10x.h（ST 外设定义 + system_stm32f10x.h）
    ↑ 组合外设和内核
用户代码
```

### 2.6 常见问题

**Q：`core_cm3.h` 和 `core_cm3.c` 必须都加进工程吗？**

- **`.h` 必须**——它被 `stm32f10x.h` 第 478 行包含，`stm32f10x.h` 又会被几乎所有代码包含，所以是**间接强制依赖**
- **`.c` 通常不需要**——`core_cm3.c` 只提供了 ITM 调试输出函数，大部分工程用不到。NVIC、SCB、SysTick 的函数全是 `static inline` 定义在 `.h` 中的，不需要 `.c`

**Q：为什么 NVIC 和 SysTick 的函数没有 `.c` 实现？**

因为它们都是 **`static inline` 函数**，定义在头文件中。编译时，调用这些函数的地方会直接被展开为寄存器操作指令——不产生函数调用、不压栈、不影响性能。

```c
// 用户代码
NVIC_EnableIRQ(USART1_IRQn);

// 编译后展开为（伪代码）
*(NVIC_BASE + (37 >> 5) * 4) |= (1 << (37 & 0x1F));
// 就是一条 STR 指令
```

**Q：其他 Cortex-M 芯片也能用类似的文件吗？**

是的。ARM 为每种 Cortex-M 内核都提供了对应的 Core 头文件：

| 内核 | 头文件 |
|------|--------|
| Cortex-M0 / M0+ | `core_cm0.h` |
| Cortex-M3 | `core_cm3.h` ← 本文 |
| Cortex-M4 / M4F | `core_cm4.h` |
| Cortex-M7 | `core_cm7.h` |
| Cortex-M33 | `core_cm33.h` |

接口风格完全一致——都有 `NVIC_EnableIRQ`、`SysTick_Config`、`__enable_irq` 等函数。换了芯片型号，接口名不用改，底层实现自动适配。

**Q：`__enable_irq()` 和 `NVIC_EnableIRQ()` 有什么区别？**

| 函数 | 作用范围 | 实现方式 |
|------|---------|---------|
| `__enable_irq()` | 全局中断开关（**所有**中断） | CPU 内核指令 `CPSIE I` |
| `NVIC_EnableIRQ(IRQn)` | 单个外设中断开关 | 写 NVIC 的 ISER 寄存器 |

通俗说：`__enable_irq()` 是总闸，`NVIC_EnableIRQ()` 是分闸。要正常使用中断，**总闸和分闸都要打开**。

---

## 3. 一句话总结

| 文件 | 一句话 |
|------|--------|
| **core_cm3.h** | ARM Cortex-M3 内核的"操作手册"——提供 NVIC、SCB、SysTick 等内核外设的寄存器定义和操作函数（全是 `static inline`，无调用开销） |
| **core_cm3.c** | ITM 调试输出的实现（`ITM_SendChar`），一般工程可不加 |

> **一句话总结**：`core_cm3.h` 是 Cortex-M3 生态的"地基"，定义了所有 CM3 芯片都通用的内核寄存器接口——无论你用的是 STM32、GD32 还是 NXP 的 LPC17xx，NVIC、SysTick、SCB 的操作方式都来自这一个文件。

### 关联链接

- [stm32f10x.h头文件.md](./stm32f10x.h头文件.md) — 第 478 行包含 `core_cm3.h` 的位置，以及第 148-161 行 CMSIS 配置宏的定义
- [system_stm32f10x系统文件.md](./system_stm32f10x系统文件.md) — `SCB->VTOR` 的实际使用场景和 `SystemCoreClock` 变量
- [寄存器与存储器映射.md](./寄存器与存储器映射.md) — `volatile` 和结构体映射原理
- [启动文件.md](./启动文件.md) — 向量表与 NVIC 的配合
- [中断向量表.md](./中断向量表.md) — 中断号枚举与向量表的关系
