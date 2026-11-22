---
title: STM32 RCC 时钟配置
tags:
  - stm32
  - 嵌入式
  - 时钟
status: draft
updated: 2026-10-06
---
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
PCLK1分频系数为2,毕竟是低速总线,出来的是36M
PCLK2分频系数为1,输出72M
然后使能系统时钟,设置时钟源为PLL

但是注意PLL在程序中要在设置三个预分频器之后设置,因为在作为系统时钟来源时不能关闭锁相环
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
还有最开始没有等待HSI使能的函数,要通过操作CR寄存器的方式来验证
```C
#include "stm32f10x.h"

void RCC_HSE_72M_Init(void)
{
	//调用函数清零配置
    RCC_DeInit();
    //配置HSE使能
    RCC_HSEConfig(RCC_HSE_ON);
    //等待使能完成
    while(RCC_GetFlagStatus(RCC_FLAG_HSERDY) == RESET);
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
	//设置系统
    RCC_SYSCLKConfig(RCC_SYSCLKSource_PLLCLK);
    while(RCC_GetSYSCLKSource() != 0x08);
}

int main(void)
{
    RCC_HSE_72M_Init();
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);

    while(1)
    {

    }
}
```