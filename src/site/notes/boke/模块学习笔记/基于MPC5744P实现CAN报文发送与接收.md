---
{"dg-permalink":"“基于MPC5744P实现CAN报文发送与接收”","dg-home":true,"dg-publish":true,"permalink":"/“基于MPC5744P实现CAN报文发送与接收”/","tags":"gardenEntry","dgPassFrontmatter":true}
---


# 修订记录

| 版本  | 日期       | 作者   | 修改原因     |
| ----- | ---------- | ------ | ------------ |
| 1.0.0 | 09/14/2022 | 张三强 | 初始草稿版本 |
| 1.0.1 | 09/15/2022 | 张三强 | 修改程序错误 |

# 缩略词

| 缩略词 | 全称                                         |
| ------ | -------------------------------------------- |
| CAN    | 控制器域网 (Controller Area Network)         |
| PIT    | 周期性中断定时器（Periodic Interrupt Timer） |
| LED    | 发光二极管（Light-Emitting Diode）           |
| SDK    | 软件开发工具包（Software Development Kit）   |
| GPIO   | 通用输入输出（General-Purpose Input/Output） |
| RAM    | 随机存取存储器（Random Access Memory）       |

# 1.    摘要

本篇笔记主要记录基于二代板实现CAN报文发送与接收的过程。在实践中加深对CAN模块的理解。

# 2.    准备工作

1.      安装S32DS for PA；

2.      安装S32DS 的最新版SDK 3.0.3；

3.      安装ZCANPRO。

# 3.    模块

## 3.1    CAN
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210171726241.webp)

MPC5744P中共有三个CAN接口，分别为CAN0、CAN1、CAN2。本次实践采用CAN0进行发送和接收报文。通过ZCANPRO采集报文，并向MPC5744P发送不同的报文，实现不同LED的亮灭。

# 4.    实现步骤

## 4.1    前期设置

### 4.1.1    新建工程
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210171730854.webp)

![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210171730889.webp)

新建一个工程，选择芯片MPC5744P，选择SDK最新版3.0.3。

### 4.1.2    选择芯片的封装类型
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210171752303.webp)

双击Component，在弹出窗口的Processors页面中点击MPC5744P芯片，选择144封装类型。
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210171753394.webp)

由于芯片封装类型改变，引脚的设置也需要更新。双击PinSettings，点击Switch Configuration切换配置。

### 4.1.3    配置LED引脚
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210171755874.webp)

要实现灯的亮灭，就需要对应引脚输出高低电平，根据手册配置PD[5],勾选SIUL2/gpio/53，PD[6]，勾选SIUL2/gpio/45的out选项，可以实现高低电平的输出。

### 4.1.4    配置CAN引脚
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210171756799.webp)
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210171800009.webp)

本次实践使用CAN0，由电路图可知二代板的CAN0对应的是PB0、PB1引脚，在软件中配置CAN0对应的引脚为PB0、PB1。

### 4.1.5    配置CAN
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210171803576.webp)
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210171811121.webp)

点击Components选择flexcan双击，新建can完成。双击canCom1:flexcan出现配置页面。本次实践无需修改。

### 4.1.6    配置PIT定时器
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210171812957.webp)

双击Component，在Alphabetical页面中双击pit选项，创建PIT定时器。
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210171813181.webp)

双击Components下的pit1，对pit定时器进行配置。在弹出的页面中Name是定时器的名称，Channel是配置的通道，Period units是定时器时间单位。时间单位分为两种：一种是Microsecond unit，单位为微秒；另一种为Count unit，单位为时钟信号的脉冲。下面的Timer period就为设定的时间，如果选择Microsecond unit，想要设定1秒，那么这里就需要填1000000。如果选择Count unit，PIT的时钟信号频率为100MHz，想要设定1秒，就需要填100000000。
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210171816914.webp)

在配置完成后需要将所有配置载入程序中，点击图中鼠标所指图标。

## 4.2    代码

### 4.2.1    自定义函数实现
```c
#include "Cpu.h"
#include "clockMan1.h"
#include "canCom1.h"
#include "canCom2.h"
#include "dmaController1.h"
#include "pin_mux.h"

#include <stdint.h>//调用uint8_t ,uint16_t ,uint32_t 类型时需要调用头文件
#include <stdbool.h>//定义了一个布尔类型

/* 定义 */

#define LED_PORT        PTD
#define LED0            5U
#define LED1            6U

#define TX_MAILBOX  (1UL)
#define TX_MSG_ID   (1UL)
#define RX_MAILBOX  (0UL)
#define RX_MSG_ID   (2UL)

typedef enum//枚举
{
    LED0_CHANGE_REQUESTED = 0x00U,
    LED1_CHANGE_REQUESTED = 0x01U
} can_commands_list;

uint8_t ledRequested = (uint8_t)LED0_CHANGE_REQUESTED;

void SendCANData(uint32_t mailbox, uint32_t messageId, uint8_t data, uint32_t len);
void BoardInit(void);
void GPIOInit(void);

void SendCANData(uint32_t mailbox, uint32_t messageId, uint8_t data, uint32_t len)//设置发送的数据
{
/*结构体*/
    flexcan_data_info_t dataInfo =
    {
            .data_length = len,
            .msg_id_type = FLEXCAN_MSG_ID_STD
    };
    FLEXCAN_DRV_ConfigTxMb(INST_CANCOM1, mailbox, &dataInfo, messageId);//配置Tx消息缓冲区。
    FLEXCAN_DRV_Send(INST_CANCOM1, mailbox, &dataInfo, messageId, data);//发送CAN帧
}


void BoardInit(void)//初始化时钟、引脚。PIT，开启定时器
{
    CLOCK_SYS_Init(g_clockManConfigsArr, CLOCK_MANAGER_CONFIG_CNT,
                        g_clockManCallbacksArr, CLOCK_MANAGER_CALLBACK_CNT);
    CLOCK_SYS_UpdateConfiguration(0U, CLOCK_MANAGER_POLICY_FORCIBLE);

    PINS_DRV_Init(NUM_OF_CONFIGURED_PINS, g_pin_mux_InitConfigArr);
    PIT_DRV_Init(INST_PIT1, &pit1_InitConfig);//初始化PIT模块
    PIT_DRV_InitChannel(INST_PIT1, &pit1_ChnConfig0);//初始化PIT通道
    PIT_DRV_StartChannel(INST_PIT1, pit1_ChnConfig0.hwChannel);//此功能启动计时器通道计数。
}

void GPIOInit(void)//设置LED输出值
{
    PINS_DRV_ClearPins(LED_PORT, (1 << LED0) | (1 << LED1));

}

void PIT_Ch0_IRQHandler(void)//中断设置
{
    SendCANData(TX_MAILBOX, TX_MSG_ID, ledRequested, 1UL);//发送报文
  	PIT_DRV_ClearStatusFlags(INST_PIT1, pit1_ChnConfig0.hwChannel);//清除通道0中断标志
}

```
首先将设置过程中生成的.h文件添加到开始进行预编译，便可以在程序中调用自动生成好的函数了。由于本次实践中还用到了uint8_t ,uint16_t ,uint32_t 类型和布尔类型所以调用stdint.h和stdbool.h这个两个头文件。#define对LED对应的引脚定义标识符，后续编写使用标识符代替LED引脚。定义了can报文发送接收的邮箱和ID。typedef enum为枚举，通过枚举定义了两个变量，并对两个变量赋予了初值。将LED0_CHANGE_REQUESTED的值赋给ledRequested。

对函数提前进行声明，然后创建SendCANData函数，作用是设置有关要发送的数据的信息并发送报文；创建BoardInit函数作用是初始化时钟、引脚、PIT模块同时开始PIT计数；创建GPIOInit函数，此函数的作用设置LED的输出值；定义中断PIT_Ch0_IRQHandler， SendCANData为之前设定好的函数，作用是发送报文，PIT_DRV_ClearStatus Flags (INST_PIT1, pit1_ChnConfig0.hwChannel) 的作用是清除已产生的定时器中断标志。

### 4.2.2    主函数实现
```c
int main(void)
{
  /*** Processor Expert internal initialization. DON'T REMOVE THIS CODE!!! ***/
  #ifdef PEX_RTOS_INIT
    PEX_RTOS_INIT();                   /* Initialization of the selected RTOS. Macro is defined by the RTOS component. */
  #endif
  /*** End of Processor Expert internal initialization.                    ***/
  #ifdef PEX_RTOS_INIT
    PEX_RTOS_INIT();                   /* Initialization of the selected RTOS. Macro is defined by the RTOS component. */
  #endif
	 BoardInit();
	 GPIOInit();
	 FLEXCAN_DRV_Init(INST_CANCOM1, &canCom1_State, &canCom1_InitConfig0);//初始化flexcan模块
	 /*结构体*/
	    flexcan_data_info_t dataInfo =
	    {
	            .data_length = 1U,
	            .msg_id_type = FLEXCAN_MSG_ID_STD
	    };

	    FLEXCAN_DRV_ConfigRxMb(INST_CANCOM1, RX_MAILBOX, &dataInfo, RX_MSG_ID);//配置接收消息缓冲区
	    while(1)//无限循环
	    {

	        flexcan_msgbuff_t recvBuff;//结构体FlexCAN消息缓冲区结构
	        FLEXCAN_DRV_Receive(INST_CANCOM1,RX_MAILBOX, &recvBuff);//接收消息至缓冲区
	        while(FLEXCAN_DRV_GetTransferStatus(INST_CANCOM1, RX_MAILBOX) == STATUS_BUSY);//等待消息接收完成
	        if((recvBuff.data[0] == LED0_CHANGE_REQUESTED) &&
	                recvBuff.msgId == RX_MSG_ID)//如果接收到的消息数据为0，id为2
	        {

	            PINS_DRV_TogglePins(LED_PORT, (1 << LED0));//LED0引脚的电平翻转

	        }
	        else if((recvBuff.data[0] == LED1_CHANGE_REQUESTED) &&
	                recvBuff.msgId == RX_MSG_ID)//如果接收到的消息数据为1，id为2
	        {

	            PINS_DRV_TogglePins(LED_PORT, (1 << LED1));//LED1引脚的电平翻转
	        }
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
这部分代码主要作用是配置can的接收缓冲区，接收ZCANPRO发送的报文，并根据接收的报文亮灭不同的灯。当接收的消息数据为0x02，ID为0x00时，对应的LED引脚电平就会翻转，通过发送不同的报文就可以控制不同的灯不断亮灭。

## 4.3    代码下载
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210171821419.webp)

将开发板通过PE micro连接至电脑，点击S32DS窗口上方的闪电图标，出现如下窗口。
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210171821788.webp)

左侧选择你需要下载的程序，同一个程序会出现两个版本，一个是刷入Flash断电后程序依旧存在，一个是刷入RAM断电后程序会被擦除。
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210171823844.webp)

对下载接口进行选择，点击Interface后的框，使用二代板所以选择第一项，Apply保存后就可以点击Flash进行下载。

## 4.4    报文发送与接收

将CAN0的high与low通过线束连接到对应周立功USB CANFD CAN0接口的high和LOW。周立功连接电脑。首先在电脑上打开软件ZCANPRO，点击设备管理，选择对应的设备类型点击打开设备。然后点击启动，若CAN_high与CAN_low之间有终端电阻选择使能，没有终端电阻选择禁能。完成后点击确认。此时给电路板上电，打开IG，视图1每过一秒就会出现一条报文。
![](https://picgo-1257922557.cos.ap-beijing.myqcloud.com/img/202210171825150.webp)

点击发送数据中的普通发送，数据长度填1，数据可以填00或01，发送次数和间隔可以根据自己的意愿调整。然后点击立即发送，就可以观察到电路板上LED的亮灭，软件CAN视图也会出现发送的报文。

# 5.    参考文档

| 序号 | 文献                        |
| ---- | --------------------------- |
| 1    | MPC5744PRM  V6.1    10/2017 |
|2|EPS_PowerPark_Sch_V2|







