---
{"dg-permalink":"“基于MPC5744P实现通过eTimer输出PWM并控制LED”","dg-home":true,"dg-publish":true,"permalink":"/“基于MPC5744P实现通过eTimer输出PWM并控制LED”/","tags":"gardenEntry","dgPassFrontmatter":true}
---


# 修订记录

| 版本  | 日期       | 作者   | 修改原因     |
| ----- | ---------- | ------ | ------------ |
| 1.0.0 | 10/14/2022 | 张三强 | 初始草稿版本 |


# 缩略词

| 缩略词 | 全称                                       |
| ------ | ------------------------------------------ |
| SDK    | 软件开发工具包(Software Development Kit)   |
| GPIO   | 通用输入输出(General-Purpose Input/Output) |
| RAM    | 随机存取存储器(Random Access Memory)       |
| PWM    | 脉冲宽度调制(Pulse Width Modulation)       |
| LED    | 发光二极管（Light-Emitting Diode）         |

# 1.    摘要

本篇笔记主要记录基于二代板实现同eTimer模块输出PWM并控制LED。在实践中加深对eTimer模块的理解。

# 2.    准备工作

1.      安装S32DS for PA；

2.      安装S32DS 的最新版SDK 3.0.3；

# 3.    模块

## 3.1    eTimer

MPC5744P芯片每个eTimer模块含有6个相同的计数通道，每个计数通道含有16位计数器、分频器、保持寄存器各一个，捕捉寄存器、比较寄存器、比较预加载寄存器各两个，还有四个控制寄存器。eTimer模块的时钟由MOTC_CLK提供，故在使用eTimer模块之前需要进行MOTC_CLK时钟的配置。eTimer可进行比较、捕捉、脉冲产生等多种功能。

# 4.    实现步骤

## 4.1    前期设置

### 4.1.1    新建工程

第一步：点击新建工程。

第三步：点击选择SDK。

第二步：选择芯片。

新建一个工程，选择芯片MPC5744P，选择SDK最新版3.0.3。

### 4.1.2    选择芯片的封装类型

第二步：点击Processors页面。

第三步：点击MPC5744P。

第四步：点击144封装。

第一步：双击Component。

第五步：点击Finish。

双击Component，在弹出窗口的Processors页面中点击MPC5744P芯片，选择144封装类型。

第六步：双击击pin。

第七步：点击Switch Configuration。

由于芯片封装类型改变，引脚的设置也需要更新。双击PinSettings，点击Switch Configuration切换配置。

### 4.1.3    添加eTimer模块

第二步：双击eTimer。

第一步：双击Components。

双击Component，在Alphabetical页面中双击eTimer，创建eTimer模块。

### 4.1.4    配置eTimer引脚

第三步：点击ETIMER_0栏选择PC[12]，output。

第二步：点击点击ETIMER栏

第一步：双击选项

双击pin选项，在Routing页面的ETIMER栏中ETIMER_0的Channel5选项中选择PC[12]，Direction列选择output。

第七步：点击Done保存。

第六步：选ETIMER 0/etc/5的Out选项。

第五步：点击PC[12]的Selected Function列

第四步：输入pc进行搜索。

点击Pins选项，搜索PC，点击PC[12]的Selected Function列，勾选ETIMER 0/etc/5的Out选项，点击Done保存。

第九步：勾选SIUL2/gpio/53的out选项

第八步：输入pd搜索。

再次搜索pd，点击PD[5]的Selected Function列，勾选SIUL2/gpio/53的out选项，点击Done保存。

本次测试主要为实现通过eTimer输出PWM并控制灯的亮灭，所以选择PC[12]作为PWM的输出引脚，PD[5]为控制LED亮灭的引脚。

### 4.1.5    配置eTimer模块

第二步：点击修改MOTC_CLK时钟。

第一步：双击clockMan1选项。

eTimer模块的时钟由MOTC_CLK提供，故在使用eTimer模块之前需要进行MOTC_CLK时钟的配置。

第四步：点击修改通道计数模式为上下沿计数。

第三步：点击新建通道。

eTimer模块需要新建三个通道，改成上下沿计数模式。通道3负责控制LED，通道4负责更改PWM的脉冲宽度，通道5负责控制PWM的频率。

第六步：点击选择相应输入源，通道3分频128，通道4分频64，通道5分频4。

第五步：点击Primary Input选项

在eTimer模块中Primary Input选项中的选择通道中输入信号的来源。通道3选择使用总线时钟除以128作为输入源，通道4选择使用总线时钟除以64作为输入源，通道5选择使用总线时钟除以4作为输入源。

## 4.2    代码

### 4.2.1    自定义函数实现

本次设置了三个函数，ETIMER0_Ch3_IRQHandler函数的作用是在经过延时后对引脚的电平进行翻转并清除中断标志。ETIMER0_Ch4_IRQHandler函数的作用是设置PWM脉冲宽度先增加后减少不断循环并清除中断标志。ETIMER0_Ch5_IRQHandler函数的作用是清除中断标志。

### 4.2.2    主函数实现

主函数中开始对时钟、引脚、eTimer模块进行了初始化，设置了通道3、4的中断，MER_ETIDRV_EnableInterruptSource(INST_ETIMER1,ETIMER_CH_IRQ_SOURCE_TOFIE,3)的作用是启用了通道3的中断，在计数器溢出时启用中断，在eTimer模块中计数器为16位，所以当计数值超过65535时中断启动，ETIMER_DRV_StartTimerChannels函数作用是启用了通道计数。

## 4.3    代码下载

第一步：点击⚡图标

将开发板通过USB连接至电脑，点击S32DS窗口上方的闪电图标，出现如下窗口。

第二步：选择需要下载的程序。

左侧选择你需要下载的程序，同一个程序会出现两个版本，一个是刷入Flash断电后程序依旧存在，一个是刷入RAM断电后程序会被擦除。

第五步：点击Flash下载程序。

第四步：点击保存设置。

第三步：选择第一个下载接口。点击选择

对下载接口进行选择，点击Interface后的框，使用二代板所以选择第一项，Apply保存后就可以点击Flash进行下载。

# 5.    示波器检测

对PC[12]引脚进行检测，检测结果如下图。输出的PWM随时间先增加后减少不断循环，频率为60.59Hz与程序设置计算出的频率16MHz/4/65535=61Hz基本符合。

# 6.    总结

1. 对eTimer模块中断触发条件不清楚，通过查询芯片手册了解到中断的触发条件为计数器溢出；

2. 对eTimer模块定时器通道3、4的计数溢出中断时间不清楚，同查询手册了解到计数器寄存器为16位，通过对时钟频率、寄存器位数和分频数的计算得出通道3中断启用的周期约为0.52s，通道4中断启用周期约为0.26s，与实际测量的周期基本相符；

3.输出PWM频率由由通道5控制。

# 7.    参考文档

| 序号 | 文献                        |
| ---- | --------------------------- |
| 1    | MPC5744PRM  V6.1    10/2017 |
| 2    | EPS_PowerPark_Sch_V2        |







