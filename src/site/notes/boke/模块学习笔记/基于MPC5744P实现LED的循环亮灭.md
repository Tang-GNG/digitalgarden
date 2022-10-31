---
{"dg-permalink":"“基于MPC5744P实现LED的循环亮灭”","dg-home":true,"dg-publish":true,"permalink":"/“基于MPC5744P实现LED的循环亮灭”/","tags":"gardenEntry","dgPassFrontmatter":true}
---

# 修订记录
| 版本  | 日期       | 作者   | 修改原因                             |
| ----- | ---------- | ------ | ------------------------------------ |
| 1.0.0 | 08/26/2022 | 张三强 | 初始草稿版本                         |
| 1.0.1 | 08/29/2022 | 张三强 | 增加图片注释，修改文本错误           |
| 1.0.2 | 09/02/2022 | 张三强 | 修改文档错误，增加电路管脚图片与注释 |
| 1.0.3 | 10/27/2022 | 张三强 | 修改电路板为二代板                   |

# 缩略词

| 缩略词 | 全称                                         |
| ------ | -------------------------------------------- |
| GPIO   | 通用输入输出（General-Purpose Input/Output） |
| PIT    | 周期性中断定时器（Periodic Interrupt Timer） |
| SDK    | 软件开发工具包（Software Development Kit）   |
| RAM    | 随机存取存储器（Random Access Memory）       |
| LED    | 发光二极管（Light-Emitting Diode）        |




# 1.    摘要

本篇笔记主要记录基于二代板实现LED亮灭循环的过程。在实践中加深对PIT和GPIO模块的理解。

# 2.    准备工作

1.      安装S32DS for PA；
2.      安装S32DS 的最新版SDK 3.0.3。

# 3.    模块

## 3.1    PIT
![|500](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210171536698.webp)
![|500](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210171639924.webp)

 MPC5744P中的PIT（Periodic Interrupt Timer）是32位通用定时器，4通道，是递减定时器。由手册中的外设时钟图可知，PIT的时钟来源为PBRIDGEx_CLK，改变PBRIDGEx_CLK的分频数就可以改变   PIT的时钟频率。可以通过PIT发送定时中断，在设定时间后改变控制LED引脚的电平实现LED的亮灭循环。

## 3.2    GPIO
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210281346489.webp)

![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210281346848.webp)

上图为二代板LED电路图和控制LED的芯片引脚图。由图可知，控制LED_GREEN的引脚为PD5，若PD5为低电平，LED_GREEN亮；若PD5为高电平LED_GREEN灭。想要让LED_GREEN循环亮灭，需要让PD5不断输出高低电平。

# 4.    实现步骤

## 4.1    前期设置

### 4.1.1    新建工程
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210171646734.webp)
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210171648899.webp)
新建一个工程，选择芯片MPC5744P，选择SDK最新版3.0.3。

### 4.1.2    选择芯片的封装类型
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210171650466.webp)


双击Component，在弹出窗口的Processors页面中点击MPC5744P芯片，选择144封装类型。
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210171652739.webp)
由于芯片封装类型改变，引脚的设置也需要更新。双击PinSettings，点击Switch Configuration切换配置。

### 4.1.3    配置LED引脚
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210281359379.webp)
要实现灯的亮灭，就需要对应引脚输出高低电平，根据手册配置PD[5],勾选SIUL2/gpio/53的out选项，可以实现高低电平的输出。

### 4.1.4    配置PIT定时器

双击Component，在Alphabetical页面中双击pit选项，创建PIT定时器。
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210171655371.webp)

双击Components下的pit1，对pit定时器进行配置。在弹出的页面中Name是定时器的名称，Channel是配置的通道，Period units是定时器时间单位。时间单位分为两种：一种是Microsecond unit，单位为微秒；另一种为Count unit，单位为时钟信号的脉冲。下面的Timer period就为设定的时间，如果选择Microsecond unit，想要设定1秒，那么这里就需要填1000000。如果选择Count unit，PIT的时钟信号频率为100MHz，想要设定1秒，就需要填100000000。
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210171655119.webp)

在配置完成后需要将所有配置载入程序中，点击图中鼠标所指图标。

## 4.2    代码

### 4.2.1    中断函数实现
```c
#include "Cpu.h"
#include "pit1.h"
#include "clockMan1.h"
#include "pin_mux.h"

  volatile int exit_code = 0;
/* User includes (#include below this line is not maintained by Processor Expert) */

/*! 
  \brief The main function for the project.
  \details The startup initialization sequence is the following:
 * - startup asm routine
 * - main()
*/
#define PORT PTD
#define LED 5

void PIT_Ch0_IRQHandler(void)
{
  		PINS_DRV_TogglePins(PORT, (1 << LED));
  		PIT_DRV_ClearStatusFlags(INST_PIT1, pit1_ChnConfig0.hwChannel);     /* 清除通道0中断标志 */
}
```

首先将设置过程中生成的.h文件添加到开始进行预编译，便可以在程序中调用自动生成好的函数了。之后开始定义一个中断，PINS_DRV_TogglePins这个函数的作用是将设定的引脚的电平翻转，实现LED的亮灭。PIT_DRV_ClearStatusFlags(INST_PIT1, pit1_ChnConfig0.hwChannel) 的作用是清除已产生的定时器中断标志。

### 4.2.2    主函数实现
```c
int main(void)
{
  /* Write your local variable definition here */

  /*** Processor Expert internal initialization. DON'T REMOVE THIS CODE!!! ***/
  #ifdef PEX_RTOS_INIT
    PEX_RTOS_INIT();                   /* Initialization of the selected RTOS. Macro is defined by the RTOS component. */
  #endif
  /*** End of Processor Expert internal initialization.                    ***/

        /* 初始化时钟*/
    	CLOCK_SYS_Init(g_clockManConfigsArr,   CLOCK_MANAGER_CONFIG_CNT,
    	               g_clockManCallbacksArr, CLOCK_MANAGER_CALLBACK_CNT);
    	CLOCK_SYS_UpdateConfiguration(0U, CLOCK_MANAGER_POLICY_AGREEMENT);
    	/* 初始化配置引脚 */
    	PINS_DRV_Init(NUM_OF_CONFIGURED_PINS, g_pin_mux_InitConfigArr);
    	/* 初始化 PIT */
    	PIT_DRV_Init(INST_PIT1, &pit1_InitConfig);
    	/* 初始化通道 0 */
    	PIT_DRV_InitChannel(INST_PIT1, &pit1_ChnConfig0);
    	/* 开始通道0计数 */
    	PIT_DRV_StartChannel(INST_PIT1, pit1_ChnConfig0.hwChannel);

  /* Write your code here */
  /* For example: for(;;) { } */

  /*** Don't write any code pass this line, or it will be deleted during code generation. ***/
  /*** RTOS startup code. Macro PEX_RTOS_START is defined by the RTOS component. DON'T MODIFY THIS CODE!!! ***/
  #ifdef PEX_RTOS_START
    PEX_RTOS_START();                  /* Startup of the selected RTOS. Macro is defined by the RTOS component. */
  #endif
  /*** End of RTOS startup code.  ***/
  /*** Processor Expert end of main routine. DON'T MODIFY THIS CODE!!! ***/
  for(;;) {
    if(exit_code != 0) {
      break;
    }
  }
  return exit_code;
  /*** Processor Expert end of main routine. DON'T WRITE CODE BELOW!!! ***/
}
```
这部分代码主要作用是对时钟、引脚、pit及通道的初始化和pit定时器启动计时。由于pit配置的时间为1秒，当时间到一秒后中断触发，LED灯亮，再过1秒，中断触发，LED引脚的电平被翻转，LED灯灭，一直进行循环，LED也就以2秒为周期不断亮灭。

## 4.3    代码下载
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210171706568.webp)

将开发板通过USB连接至电脑，点击S32DS窗口上方的闪电图标，出现如下窗口。
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210171707961.webp)

左侧选择你需要下载的程序，同一个程序会出现两个版本，一个是刷入Flash断电后程序依旧存在，一个是刷入RAM断电后程序会被擦除。
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210171707587.webp)

对下载接口进行选择，点击Interface后的框，使用开发板所以选择最后一项，Apply保存后就可以点击Flash进行下载。

# 5.    参考文档
| 序号 | 文献                        |
| ---- | --------------------------- |
| 1    | MPC5744PRM  V6.1    10/2017 |
| 2    | SCH-29333-E                 |
| 3    | EPS_PowerPark_Sch_V2.pdf    |








