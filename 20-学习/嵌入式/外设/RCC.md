# STM32 RCC 时钟

## 介绍
RCC全名Reset and clock control 复位与时钟控制器
是单片机内部一个负责产生/选择/分配时钟信号的精密网络.
它的核心作用是将原始时钟源,经过倍频/分频/选择,转化为芯片内每个部件所需的时钟频率.

## 时钟树
时钟树主体结构如图所示
![[Pasted image 20261122092055.png]]

## 时钟源
STM32 常见的高速时钟源包括 HSI 内部 RC 振荡器和 HSE 外部晶振。
HSI 在 STM32F1 上通常为 8 MHz，但它是内部 RC 振荡器，不是晶振；
HSE 的频率取决于开发板上的外部晶振，常见值为 8 MHz。
一般产品会优先使用较稳定的 HSE，但需要根据具体芯片和电路确认频率与起振情况。

时钟信号先从HSE出来到PLL锁相环
在PLL锁相环的作用下倍频乘9 让原来来自HSE的8M的信号变为72M
对于常见的 STM32F1 系列，72 MHz 是常使用的系统时钟频率

然后PLL出来的频率再前往三个预分频器,三个预分频器HCLK,PCLK1,PCLK2同时对应三条总线AHB系统总线,APB1低速外设总线 ,APB2高速外设总线
在这里HCLK分频系数为1,代表不分频,出来仍然是72M
PCLK1分频系数为2, STM32F1 规定 **APB1 最高 36MHz**
PCLK2分频系数为1,输出72M
然后使能系统时钟,设置时钟源为PLL

**PLL 参数只能在 PLL 关闭时改**，所以必须在切换 `SYSCLK` 到 PLL 之前配好
## 配置流程
startup 启动文件里会先调用 `SystemInit()`,标准库在 `system_stm32f10x.c` 里已经把时钟配到 72M(宏 `SYSCLK_FREQ_72MHz`),所以很多人一行时钟配置都不写也能跑到 72M
自己再写一遍配置之前要先知道这件事,否则等于重复配置

HSE的配置流程
1. 先把所有的配置清零
2. 配置HSE使能
3. 等待使能完成
4. 操作Flash预缓存区,配置为2
5. 使能Flash预缓存区
6. 设置预分频系数
7. 配置PLL锁相环
8. 使能PLL锁相环
9. 等待锁相环使能
10. 设置系统时钟来源为锁相环
11. 验证系统时钟来源是否为PLL
唯一要设置为参数的就是配置PLL时候的倍频系数
HSI 不同的就是在配置PLL的时候要先分频系数为2,然后倍频系数为16为最大
(8/2)* 16 = 64M
## 配置代码

**使用HSE到72M**
8M过PLL倍频9变为72M
* HCLK:1分频不变72M,传递给AHB系统总线
* PCLK1:2分频72/2=36M,传递给APB1低速外设总线
* PCLK2:1分频72M,传递给APB2高速外设总线
*超频可以修改PLL锁相环的倍频因子*
*但 `RCC_PLLMul_16` 已经是最大档,HSE 8M × 16 = 128M*
*超频之后 Flash 等待周期、ADC 和 USB 的分频都要跟着改,稳定性不保证*
```C
#include "stm32f10x.h"

void RCC_HSE_72M_Init(void)
{
	//调用函数清零配置
    RCC_DeInit();
    //配置HSE使能
    RCC_HSEConfig(RCC_HSE_ON);
    //等待使能完成,起振失败就退出,系统留在HSI 8M,不要死等
    if(RCC_WaitForHSEStartUp() == ERROR)
    {
        return;
    }
    //72M要2个Flash等待周期,同时打开预取缓冲
    FLASH_SetLatency(FLASH_Latency_2);
    FLASH_PrefetchBufferCmd(FLASH_PrefetchBuffer_Enable);
	//锁相环配置,这里不对HSE分频,倍频系数9,原来8M,过PLL后变为72M
    RCC_PLLConfig(RCC_PLLSource_HSE_Div1, RCC_PLLMul_9);
    //使能锁相环
    RCC_PLLCmd(ENABLE);
    //等待锁相环配置完成
    while(RCC_GetFlagStatus(RCC_FLAG_PLLRDY) == RESET);
	//配置三个预分频器
    RCC_HCLKConfig(RCC_SYSCLK_Div1);
    RCC_PCLK1Config(RCC_HCLK_Div2);
    RCC_PCLK2Config(RCC_HCLK_Div1);
	//设置系统时钟来源为PLL锁相环
    RCC_SYSCLKConfig(RCC_SYSCLKSource_PLLCLK);
    //验证系统时钟来源是否为PLL
    while(RCC_GetSYSCLKSource() != 0x08);
}
```

**使用HSI不开PLL倍频,直接用 HSI 当系统时钟源**
AHB:8M
PCLK1:4M
PCLK2:8M
```C
void RCC_HSI_Init(void)
{
    RCC_DeInit();
    RCC_HSICmd(ENABLE);
    while(RCC_GetFlagStatus(RCC_FLAG_HSIRDY) == RESET);

    RCC_SYSCLKConfig(RCC_SYSCLKSource_HSI);
    while(RCC_GetSYSCLKSource() != 0x00);
    //8M在24M以内,Flash等待周期保持默认0,不用改

    RCC_HCLKConfig(RCC_SYSCLK_Div1);
    RCC_PCLK1Config(RCC_HCLK_Div2);
    RCC_PCLK2Config(RCC_HCLK_Div1);
}
```

**HSI经过PLL倍频**
AHB:64M
PCLK1:32M
PCLK2:64M
```C
void RCC_HSI_PLL_64M_Init(void)
{
    RCC_DeInit();
    RCC_HSICmd(ENABLE);
    while(RCC_GetFlagStatus(RCC_FLAG_HSIRDY) == RESET);

    RCC_PLLConfig(RCC_PLLSource_HSI_Div2, RCC_PLLMul_16);
    RCC_PLLCmd(ENABLE);
    while(RCC_GetFlagStatus(RCC_FLAG_PLLRDY) == RESET);
    //64M同样要2个Flash等待周期
    FLASH_SetLatency(FLASH_Latency_2);
    FLASH_PrefetchBufferCmd(FLASH_PrefetchBuffer_Enable);

    RCC_HCLKConfig(RCC_SYSCLK_Div1);
    RCC_PCLK1Config(RCC_HCLK_Div2);
    RCC_PCLK2Config(RCC_HCLK_Div1);

    RCC_SYSCLKConfig(RCC_SYSCLKSource_PLLCLK);
    while(RCC_GetSYSCLKSource() != 0x08);
}
```

## 总线上的外设
### AHB 总线

AHB 是高速总线，`RCC_AHBPeriphClockCmd` 开启

- DMA1、DMA2
- CRC 校验单元
- FSMC（外部存储器扩展，只有大容量型号才有）
- SDIO（SD 卡接口，大容量）
- RCC 本身也挂在 AHB
- Flash、SRAM：存储器，不需要软件开门控时钟

### APB2 高速外设总线

`RCC_APB2PeriphClockCmd`

- GPIOA / GPIOB / GPIOC / GPIOD / GPIOE / GPIOF / GPIOG
- AFIO（复用功能重映射、EXTI 外部中断控制）
- TIM1（高级定时器，带死区、刹车）、TIM8（大容量）
- ADC1、ADC2、ADC3
- SPI1
- USART1

### APB1 低速外设总线

`RCC_APB1PeriphClockCmd`

- TIM2、TIM3、TIM4、TIM5（通用定时器）、TIM6、TIM7（基本定时器）
- SPI2、SPI3
- USART2、USART3、UART4、UART5
- I2C1、I2C2
- CAN
- USB
- WWDG 窗口看门狗
- BKP 备份域
- PWR 电源控制

