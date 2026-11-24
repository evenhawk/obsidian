---
title: SysTick 系统定时器
tags:
  - stm32
  - 嵌入式
  - 定时器
status: draft
updated: 2026-11-24
---

# SysTick 系统定时器

## 工作原理

系统定时器通过 AHB 总线上的时钟来对装载值进行递减，达到定时效果。

## 寄存器结构体

主要控制三个寄存器：

- **CTRL**：控制及状态寄存器
- **LOAD**：重装载寄存器
- **VAL**：当前数值寄存器

还有一个 CALIB 校准寄存器不常用。

## 中断优先级

内核里的外设和片上外设的中断优先级配置的寄存器不一样：

- 内核里的外设配置 SCB 寄存器
- 片上外设是配置 NVIC 寄存器

但是都是用 4 个位来表示中断优先级。

## 初始化

函数接收一个 uint32 的数，配置步骤如下：

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

## 计算传参公式

- `SystemCoreClock`：内核时钟频率，单位 Hz
- `f`：期望 SysTick 中断频率，单位 Hz
- `T`：期望中断周期，单位 s
- `T_ms`：期望中断周期，单位 ms
- `T_us`：期望中断周期，单位 us

```C
ticks = SystemCoreClock * T;
```

传参 = 系统频率 × 定时时间/s

## 轮询判断是否计数完毕

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
