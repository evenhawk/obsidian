# SysTick 系统定时器

**一句话**：SysTick 是 Cortex-M 内核自带的 24 位递减计数器，由 HCLK（或 HCLK/8）驱动，数到 0 时产生中断或置位标志，是"周期性任务"和"毫秒延时"的底层来源。

## 1. 定位：内核外设，不是片上外设

- SysTick 属于 **Cortex-M 内核私有外设**，和 NVIC、SCB 同属内核，**不在 STM32 的片上外设之列**
- 寄存器位于内核私有外设区，起始地址 `0xE000E010`
- 因此它**不需要在 RCC 里开时钟**，也不出现在时钟树图上；时钟直接用 HCLK
- 时钟源由 `CTRL.CLKSOURCE` 选择：`1` = 内核时钟 HCLK（72 MHz），`0` = HCLK/8（9 MHz）

## 2. 工作原理

系统定时器通过 AHB 总线上的时钟对**装载值进行递减**，达到定时效果。

```
写 LOAD → 使能 ENABLE → VAL 每个时钟减 1 → 减到 0
      → 下一个时钟：COUNTFLAG 置 1；若 TICKINT=1 则进入 SysTick 中断
      → VAL 自动重装载为 LOAD，继续递减（循环）
```

要点：

- 计数方向是**递减**，不是递增
- 一个完整周期是 `LOAD + 1` 个时钟节拍（从 LOAD 数到 0，0 本身也算一拍）
- 只要 `ENABLE=1` 就一直循环，不会自动停

## 3. 寄存器结构

主要控制三个寄存器，另有一个校准寄存器不常用。

| 寄存器 | 偏移 | 作用 |
| --- | --- | --- |
| `CTRL` | `0x00` | 控制和状态 |
| `LOAD` | `0x04` | 重装载值（只有低 24 位有效） |
| `VAL` | `0x08` | 当前计数值（读 = 当前值，写 = 清零） |
| `CALIB` | `0x0C` | 校准值，一般是 HCLK/8 下 10 ms 的计数值，很少用 |

`CTRL` 的四个关键位：

| 位 | 名称 | 含义 |
| --- | --- | --- |
| 0 | `ENABLE` | `1` = 启动计数 |
| 1 | `TICKINT` | `1` = 数到 0 产生中断；`0` = 只置 COUNTFLAG |
| 2 | `CLKSOURCE` | `1` = HCLK；`0` = HCLK/8 |
| 16 | `COUNTFLAG` | 数到 0 时硬件置 1；**读 CTRL 或写 VAL 清零** |

两个容易记错的写操作行为：

- **写 `VAL`**：写任意值都会把计数器清零，同时清掉 `COUNTFLAG`（写 0 只是习惯写法，写别的数效果一样）
- **写 `LOAD`**：超过 `0xFFFFFF` 的部分会被截断，而且新值要等当前这一轮数完才生效

## 4. 定时时间怎么算

一个节拍 = `1 / f_systick` 秒，一个周期 = `LOAD + 1` 个节拍，所以：

```
LOAD + 1 = f_systick × T
ticks    = SystemCoreClock × T
```

| 符号 | 含义 | 单位 |
| --- | --- | --- |
| `SystemCoreClock` | 内核时钟频率 | Hz |
| `f` | 期望 SysTick 中断频率 | Hz |
| `T` | 期望中断周期，`T = 1/f` | s |
| `T_ms` / `T_us` | 周期的毫秒 / 微秒表示 | ms / us |

```C
ticks = SystemCoreClock * T;
```

例如 72 MHz 下要 1 ms 中断一次：`ticks = 72 000 000 × 0.001 = 72000`。

**最大定时时间**（关键限制，很容易踩）：

```
2^24 / 72 MHz ≈ 0.233 s ≈ 233 ms
```

也就是说 72 MHz 下单次最长只能定时约 233 ms；想要更长，只能软件累加（循环调用多次，或用变量记次数）。若把时钟源换成 HCLK/8（9 MHz），上限约 1.86 s。

## 5. 中断优先级：内核外设和片上外设不一样

内核里的外设和片上外设的中断优先级配置的寄存器不一样：

- **内核外设**（SysTick、PendSV、SVC 等）：配置 **SCB** 寄存器（具体是 `SCB->SHP[]`）
- **片上外设**（GPIO、USART、TIM 等）：配置 **NVIC** 寄存器（`NVIC->IP[]`）

但是两者都用 **4 个位**表示中断优先级（STM32F1 的 `__NVIC_PRIO_BITS = 4`），即 16 级，`0~15`，数值越小优先级越高。

SysTick 是内核异常，异常号 15，对应的 `SysTick_IRQn = -1`。所以代码里虽然写的是 `NVIC_SetPriority(SysTick_IRQn, ...)`，库函数内部判断到负的 IRQn 后，实际操作的是 SCB 而不是 NVIC——这正是它和普通外设的区别。

## 6. 初始化：SysTick_Config

函数接收一个 `uint32_t` 类型的数，配置步骤如下：

1. 先判断接收的参数是否超出寄存器最大值，也就是 2^24
2. 再初始化重装载寄存器的值，这里需要使用参数进行计算
3. 配置 NVIC 中断优先级
4. 清除当前计数值，操作 VAL 寄存器
5. 配置中断控制寄存器 CTRL

代码：

```C
__STATIC_INLINE uint32_t SysTick_Config(uint32_t ticks)
{
  // 检查 ticks 是否超出 24 位重装载寄存器的最大值
  if ((ticks - 1UL) > SysTick_LOAD_RELOAD_Msk) {
    return (1UL);                   // 参数无效，返回错误
  }
  SysTick->LOAD  = (uint32_t)(ticks - 1UL);                         // 设置重装载值
  NVIC_SetPriority (SysTick_IRQn, (1UL << __NVIC_PRIO_BITS) - 1UL); // 设置中断优先级（默认为最低）
  SysTick->VAL   = 0UL;                                             // 清除当前值
  SysTick->CTRL  = SysTick_CTRL_CLKSOURCE_Msk | // 选择处理器时钟
                   SysTick_CTRL_TICKINT_Msk   | // 使能中断
                   SysTick_CTRL_ENABLE_Msk;     // 启动定时器
  return (0UL);                         // 成功
}
```

步骤与代码的对应关系：

| 步骤 | 对应代码 | 说明 |
| --- | --- | --- |
| 1 | `if ((ticks - 1UL) > SysTick_LOAD_RELOAD_Msk)` | `Msk` 就是 `0xFFFFFF`，超了返回 1 |
| 2 | `SysTick->LOAD = ticks - 1UL` | 因为周期是 `LOAD+1`，要 N 拍就写 N-1 |
| 3 | `NVIC_SetPriority(SysTick_IRQn, ...)` | `(1<<4)-1 = 15`，设为最低优先级 |
| 4 | `SysTick->VAL = 0UL` | 清当前值和 COUNTFLAG |
| 5 | `SysTick->CTRL = CLKSOURCE \| TICKINT \| ENABLE` | 三位置 1：选 HCLK、开中断、启动 |

为什么第 2 步是 `ticks - 1`：`LOAD` 存的是"重装载值"，而实际数完一轮需要 `LOAD+1` 个节拍（LOAD、LOAD-1、…、1、0）。想要 N 个节拍，就得写 `N-1`。

## 7. 两种使用方式

| 对比项 | 中断方式 | 轮询方式 |
| --- | --- | --- |
| `TICKINT` | `1` | `0` |
| 由谁响应 | 自动进入 `SysTick_Handler()` | 自己在 `while` 里查 `COUNTFLAG` |
| 是否阻塞 CPU | 否 | 是 |
| 典型用途 | 系统时基、OS tick、周期任务 | `delay` 类延时 |
| 本例函数 | `SysTick_Config()` | `delay_ms_poll()` |

## 8. 轮询判断是否计数完毕

```C
void delay_ms_poll(uint32_t ms)
{
    uint32_t ticks = (uint64_t)SystemCoreClock * ms / 1000U;

    SysTick->CTRL = 0U;                  // 先关闭，确保 TICKINT=0
    SysTick->LOAD = ticks - 1U;
    SysTick->VAL  = 0U;                  // 清当前值和 COUNTFLAG
    SysTick->CTRL = SysTick_CTRL_CLKSOURCE_Msk |
                    SysTick_CTRL_ENABLE_Msk;   // 不使能中断

    while ((SysTick->CTRL & SysTick_CTRL_COUNTFLAG_Msk) == 0U) {
        /* 等待一次计数完成 */
    }

    SysTick->CTRL = 0U;                  // 关闭
}
```

这一段里每一步都有目的：

| 语句 | 为什么不能省 |
| --- | --- |
| `CTRL = 0U`（第一次） | 确保 `TICKINT=0`，否则等计数结束会跳进中断 |
| `VAL = 0U` | 清掉上一次遗留的 `COUNTFLAG`，否则下面的 `while` 会立刻通过、延时失效 |
| `CTRL = CLKSOURCE \| ENABLE` | 只开时钟源和使能，**故意不置 TICKINT** |
| `while (...COUNTFLAG...)` | `COUNTFLAG` 是只读标志，硬件数到 0 时置 1 |
| `CTRL = 0U`（第二次） | 结束后关掉，避免定时器继续跑 |

`(uint64_t)` 强转不能省：`72 000 000 × 1000` 已经超过 32 位上限，不转成 64 位会溢出。

## 9. 常见坑

1. **超过约 233 ms 的延时不能一次完成**。上面这个 `delay_ms_poll` 没做上限检查，传入 `ms > 233` 时 `ticks` 超过 24 位，`LOAD` 会被截断，实际延时比预期短得多。正确做法是循环调用，或分多次累加。
2. **忘记清 `VAL` / `COUNTFLAG`**，轮询会立刻通过，延时形同虚设。
3. **和 HAL 库打架**。`HAL_Init()` 内部已经调用 `HAL_InitTick()`，把 SysTick 配成 1 ms 中断供 `HAL_Delay()`、`HAL_GetTick()` 使用。这时再调 `SysTick_Config()` 改掉节拍，`HAL_Delay` 就不准了。
4. **和 RTOS 打架**。FreeRTOS 这类系统把 SysTick 当作系统 tick，被 OS 接管后不能再自定义。
5. **改完 `LOAD` 不清 `VAL`**，新的重装载值要等当前一轮数完才生效，看起来像"改了不起作用"。
6. **`SysTick_Config()` 是覆盖式配置**，调用两次会重置已有节拍，不会叠加。

## 10. 典型应用

- `HAL_Delay()` / `HAL_GetTick()`：底层就是 SysTick 的 1 ms 中断
- RTOS 的系统 tick 中断
- 通用的"每隔 N 毫秒做一次"的软件定时

## 复习速查

| 要回答的问题 | 记住这个 |
| --- | --- |
| 它是谁的定时器 | Cortex-M 内核私有外设，不用开 RCC 时钟 |
| 位数 / 上限 | 24 位，最大重装载值 `0xFFFFFF` |
| 计数方向 | 递减 |
| 一轮几拍 | `LOAD + 1` |
| 72 MHz 最长定时 | 约 233 ms |
| 时钟源 | `CTRL.CLKSOURCE`：1 = HCLK，0 = HCLK/8 |
| 标志位 | `CTRL.COUNTFLAG`（bit16），读 CTRL 或写 VAL 清零 |
| 优先级寄存器 | 内核外设走 SCB，片上外设走 NVIC；都是 4 位、16 级 |
| 中断函数名 | `SysTick_Handler()` |
| 定时公式 | `ticks = SystemCoreClock × T` |
| 与 HAL 的关系 | HAL 已占用 SysTick 做 1 ms 时基 |
