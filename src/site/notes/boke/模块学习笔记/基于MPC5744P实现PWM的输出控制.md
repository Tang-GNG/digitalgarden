---
{"dg-permalink":"“基于MPC5744P实现PWM的输出控制”","dg-home":true,"dg-publish":true,"permalink":"/“基于MPC5744P实现PWM的输出控制”/","tags":"gardenEntry","dgPassFrontmatter":true}
---


# 修订记录

| 版本  | 日期       | 作者   | 修改原因     |
| ----- | ---------- | ------ | ------------ |
| 1.0.0 | 09/22/2022 | 张三强 | 初始草稿版本 |


# 缩略词
| 缩略词 | 全称                                       |
| ------ | ------------------------------------------ |
| SDK    | 软件开发工具包(Software Development Kit)   |
| GPIO   | 通用输入输出(General-Purpose Input/Output) |
| RAM    | 随机存取存储器(Random Access Memory)       |
| PWM    | 脉冲宽度调制(Pulse Width Modulation)       |


# 1.    摘要

本篇笔记主要记录基于二代板实现PWM信号的输出和调节。在实践中加深对PWM模块的理解。

# 2.    准备工作

1.      安装S32DS for PA；

2.      安装S32DS 的最新版SDK 3.0.3；

# 3.    模块

## 3.1    PWM
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210201507162.webp)

MPC5744P144封装类型的芯片中PWM有4个子模块，每个子模块有3路输出，分别为PWMA、PWMB、PWMC。本次实践采用Sub-Module 0这一模块的PWMA0、PWMB0路进行输出。MPC5744P的PWM为16位精度，占空比最小能达到1:65535。PWM模块带故障管理，可以互补输出也可以独立输出，支持中心对齐，边沿对齐，相移，双开关四种输出(多用于单电阻采样，三相重构)。

# 4.    实现步骤

## 4.1    前期设置

### 4.1.1    新建工程

![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210201508442.webp)
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210201510336.webp)

新建一个工程，选择芯片MPC5744P，选择SDK最新版3.0.3。

### 4.1.2    选择芯片的封装类型

![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210201512917.webp)

双击Component，在弹出窗口的Processors页面中点击MPC5744P芯片，选择144封装类型。
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210201514296.webp)

由于芯片封装类型改变，引脚的设置也需要更新。双击PinSettings，点击Switch Configuration切换配置。

### 4.1.3    添加FlexPWM模块

![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210201515549.webp)


双击Component，在Alphabetical页面中双击flexpwm，创建flexpwm模块。

### 4.1.4    配置PWM引脚
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210201517385.webp)

根据二代板电路图选PA[10]、PA[11]两个引脚作为PWM的输出。
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210201519173.webp)


双击PinSettings在Routing页面中点击FlexPWM选项，在页面中FlexPWM_1下的FlexPWM Input/Output Channel A0行中，Pin/Signal列选择PC[13],Direction列选择Output，FlexPWM Input/Output Channel B0行中，Pin/Signal列选择PC[14],Direction列选择Output。
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210201520045.webp)

点击Pins选项，在Pin Filter中搜索pc点击PC[13]的SelectedFunction列，在弹出的页面中选择FlexPWM_1/fpwm_ab/A0的Output选项，点击Done退出。点击PC[14]的SelectedFunction列，在弹出的页面中选择FlexPWM 1/fpwm ab/B0的Output选项，点击Done退出。

### 4.1.5    配置PWM模块

![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210201607276.webp)

双击clockMan1选项，在Clock Values Summary下滑可以看到FlexPWM1的时钟频率。
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210201608852.webp)

点击Setting选项，可以看到FlexPWM为MOTC时钟，在System clock区域便可配置MOTC的时钟源和分频。
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210201609146.webp)


双击flexPWM1，在弹出页面中对flexPWM1模块进行配置。点击All，由于之前所选的A0通道在FlexPWM1中故在Device选项中选择FlexPWM1。Counter init为计数器初始化选项、Clock source为时钟源选项、Clock frequency为时钟频率、Prescaler为预分频数选项、Channel pair operation为通道对的选项、Complementary Source为互补源的选项、Reload logic为重新加载逻辑选项、Reload Source为重新加载信号源选项、Reload frequency为重新加载频率选项、Force triggering signal为强制触发信号选项。把本次实践这些选项根据检测项目进行更改。
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210201611941.webp)

点击Sub-module signal setup，Period配置PWM的周期。已知PWM时钟频率为16MHz，配置PWM的周期为16000，则PWM的频率就为16MHz/16000=1KHz。勾选Output enable PWM A、Output enable PWM B，为输出口使能。
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210201617988.webp)

在配置完成后需要将所有配置载入程序中，点击图中鼠标所指图标。

## 4.2    代码

### 4.2.1    自定义函数实现
```c
#include "Cpu.h"

  volatile int exit_code = 0;
/* User includes (#include below this line is not maintained by Processor Expert) */
#include <stdint.h>
#include <stdbool.h>

#define STEP 1
#define MAX_VAL 16000 //设置周期


void delay(volatile int cycles)
  {
      /* 延迟 */
      while(cycles--);
  }
```
首先将设置过程中生成的.h文件添加到开始进行预编译，便可以在程序中调用自动生成好的函数了。由于本次实践中还用到了uint8_t 、uint16_t 、uint32_t 类型和布尔类型所以调用stdint.h和stdbool.h这个两个头文件。#define定义STEP为1，是脉冲宽度更新的阶梯值，#define定义MAX_VAL为周期16000。通过自减循环创建延迟函数。

### 4.2.2    主函数实现
```c
int main(void)
{
  /* 局部变量 */
	  uint16_t dutyCycle = MAX_VAL;
	  bool increaseDutyCycle = false;

	  uint16_t dutyCycle1 = MAX_VAL1;
	  bool increaseDutyCycle1 = false;
  /*** Processor Expert internal initialization. DON'T REMOVE THIS CODE!!! ***/
  #ifdef PEX_RTOS_INIT
    PEX_RTOS_INIT();                   /* Initialization of the selected RTOS. Macro is defined by the RTOS component. */
  #endif
  /*** End of Processor Expert internal initialization.                    ***/
  /*初始化时钟、引脚和PWM*/
    CLOCK_SYS_Init(g_clockManConfigsArr, CLOCK_MANAGER_CONFIG_CNT,
    			g_clockManCallbacksArr, CLOCK_MANAGER_CALLBACK_CNT);
    CLOCK_SYS_UpdateConfiguration(0U, CLOCK_MANAGER_POLICY_AGREEMENT);

    PINS_DRV_Init(NUM_OF_CONFIGURED_PINS, g_pin_mux_InitConfigArr);

	FLEXPWM_DRV_SetupPwm(INST_FLEXPWM1, 0U, &flexPWM1_ModuleSetup0, &flexPWM1_SignalSetup0);//设置pwm信号属性和模块设置
	FLEXPWM_DRV_CounterStart(INST_FLEXPWM1, 0U);//FlexPWM计数器复位

	 while (1)
		 {
		     if (increaseDutyCycle == false)//如果increaseDutyCycle为false
		     {
		         if (dutyCycle < STEP)//如果dutyCycle小于STEP
		             increaseDutyCycle = true;//increaseDutyCycle为true
		         else
			         dutyCycle -= STEP;//否则dutyCycle值为dutyCycle减STEP
		     }
		     else
		     {
		         if (dutyCycle > MAX_VAL - STEP)
		             increaseDutyCycle = false;
		         else
			         dutyCycle += STEP;
		     }
		     FLEXPWM_DRV_SetDeadtime(INST_FLEXPWM1,0u,2000,FLEXPWM_DEADTIME_COUNTER_0);
		     FLEXPWM_DRV_SetDeadtime(INST_FLEXPWM1,0u,2000,FLEXPWM_DEADTIME_COUNTER_1);

		     FLEXPWM_DRV_UpdatePulseWidth(INST_FLEXPWM1, 0U, dutyCycle, dutyCycle, FlexPwmEdgeAligned);//更新PWM脉冲宽度
		     FLEXPWM_DRV_LoadCommands(INST_FLEXPWM1, (1UL << 0));//设置PWM触发命令
		     delay(0x2FF);
		 }

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
主函数内对局部变量进行定义，初始化时钟、引脚和PWM，在while(1)无限循环内对脉冲宽度进行调节，FLEXPWM_DRV_UpdatePulseWidthPWM函数的作用是对PWM的脉冲宽度进行更新。FLEXPWM_DRV_LoadCommands(INST_FLEXPWM1, (1UL << 0))语句的作用加载预分频器和PWM值。delay(0x2FF)语句的作用是插入延时增加循环间隔时间。

## 4.3    代码下载

将开发板通过USB连接至电脑，点击S32DS窗口上方的闪电图标，出现如下窗口。
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210201622766.webp)

左侧选择你需要下载的程序，同一个程序会出现两个版本，一个是刷入Flash断电后程序依旧存在，一个是刷入RAM断电后程序会被擦除。
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210201623055.webp)

对下载接口进行选择，点击Interface后的框，使用开发板所以选择最后一项，Apply保存后就可以点击Flash进行下载。

## 4.4    示波器检验
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210201624308.webp)

根据开发板电路图，示波器采样点选择在R45、R43电阻右侧引脚。

### 4.4.1    独立输出

在Channel pair operation栏点击选择Complementary，设置为独立输出。
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210201625653.webp)

程序下载完成后示波器检测结果为下图。程序设定PWM频率为1KHz，示波器检测PWM频率为986.3Hz。 程序设置PWM为独立输出，示波器检测PWM为独立输出。
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210201626764.webp)

### 4.4.2    互补输出设置死区

设置互补输出可以点击flexPWM1在弹出页面的Channel pair operation栏进行选择，Complementary选项为互补，Independent选项为独立，点击选择Complementary。
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210201627824.webp)

死区的设置需要调用FLEXPWM_DRV_SetDeadtime函数。死区操作仅适用于互补通道操作。写入寄存器的11位值以外围时钟周期为单位。已知此次实践中PWM模块的时钟频率为16MHz，在程序中设定的寄存器写入值为2000，则死区时间为2000/16MHz=0.125ms。
```c
FLEXPWM_DRV_SetDeadtime(INST_FLEXPWM1,0u,2000,FLEXPWM_DEADTIME_COUNTER_0);
FLEXPWM_DRV_SetDeadtime(INST_FLEXPWM1,0u,2000,FLEXPWM_DEADTIME_COUNTER_1);
```
程序下载完成后，示波器检测结果为下图。程序设定PWM信号的频率为1KHz，示波器显示PWM频率为987.09Hz。程序设定PWM互补输出，示波器检测结果为互补输出。程序设定死区时间为0.125
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210201634791.webp)

# 5.    总结

在本次实践中遇到了如下问题：

1. 程序下载完成后示波器未能检查测到信号；

2. 示波器检测PWM信号出现波形被干扰。

解决方法：

1. 输出口未使能，设置使能后可以检测到波形；

2. 示波器检测时未接地，连接二代板接地端后检测波形正常。

# 6.    参考文档

| 序号 | 文献                        |
| ---- | --------------------------- |
| 1    | MPC5744PRM  V6.1    10/2017 |
|2|EPS_PowerPark_Sch_V2|



                             



