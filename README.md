# Enuo_rtos_1
从零开始构建实时操作系统1——任务切换

## 1.前言

> 随着计算机技术和微电子技术的迅速发展，嵌入式系统应用领域越来越广泛，尤其是其具备低功耗技术的特点得到人们的重视。随着工信部提出NB-IoT基站建设具体目标、三大运营商加速建设，即将迎来万物互联的新时代，这是信息产业继移动互联网之后的下一个万亿级市场，这些为实时操作系统的应用提供了广阔的前景。

  嵌入式实时操作系统将会部署到越来越多的设备中，这就要求工程师深入地了解嵌入式实时操作系统。**本系列文章将和大家一起从零开始构建一个嵌入式实时操作系统，我将用最简单直白的方式一步一步搭建，我将用一篇文章的方式来总结搭建中的每个节点阶段，并开源软件工程和源代码。**
  

## 2.嵌入式实时操作系统

嵌入式实时操作系统是一个特殊的程序，是一个支持多任务的运行环境。嵌入式实时操作系统最大的特点就是“实时性”，如果有一个任务需要执行，实时操作系统会立即执行该任务，不会有较长的延时。典型的实时操作系统有uCOS ，RT-Thread，FreeRTOS ，VxWorks，WinCE等。

![在这里插入图片描述](https://img-blog.csdnimg.cn/1ff3713e471c4115b0b67d1ff2e9e5ff.png)

**嵌入式实时操作系统是一个特殊的程序(通常称为内核)，它可以创建和控制所有任务**。嵌入式实时操作系统除了包含一个内核以外，还提供其他服务，如文件系统，协议栈，图形用户界面等。本文的重点在于了解嵌入式实时操作系统内核的工作原理和结构，因此文中提到的实时操作系统通常指的是操作系统内核。**实时操作系统内核通常要占用5%左右的CPU运行时间，另外内核是一个软件代码，需要额外占用ROM空间和RAM空间。**

嵌入式实时操作系主要由以下3个子系统组成：

> 	1、任务调度子系统 
> 		
> 2、任务通信子系统 
> 	
> 3、内存管理子系统


![在这里插入图片描述](https://img-blog.csdnimg.cn/a6a491e1c549489cb7af9031c7cddb32.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)


## 3.实现目标

**本文讲解构建嵌入式实时操作系统的第一个节点阶段：实现简单的任务切换功能。**
由于代码区的数据是不变的，处理器寄存器的值和栈空间的值决定程序运行状态。让每个任务“独享”一个栈空间，当我们将任务运行时的处理器寄存器的值保存起来时，这样就实现保存任务的运行状态。同样的当我们把保存的任务运行时的处理器寄存器的值装载到处理的寄存器中时，这样就恢复了任务的运行状态，任务继续运行起来。

切换任务的原理是：**每个任务有一个“独享”栈空间，通过保存和装载任务运行时的处理器寄存器的值，实现任务的暂停和恢复运行。暂停一个任务后再恢复另外一个任务就完成了一次任务切换。**

任务代码，任务栈空间和处理器状态如下图：

![在这里插入图片描述](https://img-blog.csdnimg.cn/19b8c08310c24bf2a9714047e6a676dc.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)


## 4.实验环境

硬件是基于意法半导体的STM32F401（ARM公司的Cortex-M4内核），软件开发使用的是KEIL V5.2 开发工具。

![在这里插入图片描述](https://img-blog.csdnimg.cn/65ced8751a1e4776bab37dd6bfa7fb17.png)

软件工程如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/9ed8602fbd2a4d7abdc357eac044b7e3.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_20,color_FFFFFF,t_70,g_se,x_16)

软件工程中包含：main.c ，startup_stm32f401xc.s 和 readme三个文件。startup_stm32f401xc.s文件为STM32F401的启动文件，main.c文件实现任务切换功能，readme文件用于记录版本修改日志。

## 5.代码实现

切换任务的原理是让每个任务都有一个“独享”栈空间，通过保存和装载任务运行时的处理器寄存器的值，实现任务的暂停和恢复运行。暂停一个任务后再恢复另外一个任务就完成了一次任务切换。
因此需要实现：

> 1、每个任务的独立栈空间。
> 
>  2、实现任务的暂停和恢复。 
>  
>  3、实现任务的调度。

**5.1实现独立栈空间**
栈空间代码如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/0f39a4b614ec4884a9e52f36f794f42c.png)

为每个任务定义一个静态数组，当任务运行时将处理器的栈指针指向任务“自己的”静态数组，从而实现独立栈空间。栈空间用来存放局部变量，中断调用和函数调用时的处理器寄存器的值。任务切换时需要将处理器寄存器的值保存到任务的独立栈空间。

在保存任务运行状态时需要保存处理器寄存器值到栈空间，因此需要深入了解处理器寄存器的用途和出入栈顺序，Cortex-M4内核的寄存器和寄存器中断自动入栈的顺序图如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/d6924e4058704928a79101273b67b9ff.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

初始化栈空间的代码如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/0b19a7bdfc864f4984bd98224497841e.png)

栈空间初始化后的状态如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/7968854e311e43d682bbbea862f4a264.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)


**栈是一中先入后出的数据结构，Cortex-M4内核的栈操作方式倍设置成了向下生长**。psp_array用于保存任务栈指针，psp_array[0]任务0栈指针指向task0_stack[112]，其中task0_stack[116]保存PC程序指针值，task0_stack[117]保存状态寄存器（符合Cortex-M4内核寄存器出栈顺序：手动出栈8个寄存器，硬件自动出栈8个寄存器）。

**5.2实现任务的暂停和恢复**
代码如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/ab0c7805cc4c40efb40e465c6a832c5d.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)


cortex-M4内核有一个PendSV(可挂起的系统调用)异常，其异常编号为14并且具有可编程的优先级。当软件将PendSV设置成挂起时，程序将进入PendSV异常（中断）。
**将PendSV异常优先级设置为最低，其它中断函数都可以得到正常响应，不会受到PendSV异常影响，在PendSV异常中执行任务切换**，时序框图如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/2a6ea2258a594a71b2ea15acd51dc0c2.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

PendSV_Handler为Cortex-M4内核中断服务函数，进入中断函数时处理器自动保存了R0,R1,R2,R3, R12,LR,PC,XPSR,在PendSV_Handler中断程序中完成R4~R11入栈保存工作，从而实现任务保存工作。

```c
/* 读取当前进程栈指针数值 */
MRS R0,PSP                        	
/* 保存R4-R11八个寄存器的值到当前任务栈中  同时将回写的地址写入R0 */
STMDB R0!,{R4-R11} 
```

psp_array[0]为任务0的栈指针， psp_array[1]为任务1的栈指针。以下代码实现任务栈指针切换

```c
/* 读取psp_array 地址 */
LDR R3, =__cpp(&psp_array)         
/* 将当前进程PSP指针值 写入 相应的 PSP_array 位置  */
STR  R0,[R3,R2,LSL #2]             
/* 获取下个进程序号 */
LDR R4,=__cpp(&next_task)          
LDR R4,[R4]
/* R1为&curr_task   将下个进程序号写入curr_task中 */
STR R4,[R1]                        
/* psp_array读取更新后的curr_task的PSP指针数值 */
LDR R0,[R3,R4,LSL #2] 
```

在PendSV_Handler中断程序中完成R4~R11寄存器出栈，PendSV_Handler中断程序返回时处理器自动出栈R0,R1,R2,R3, R12,LR,PC,XPSR，从而实现任务恢复工作。

```c
/* 出栈 R4-R11八个寄存器 */
LDMIA R0!,{R4-R11}                 
/* 设置PSP指针 */
MSR PSP,R0	
/* 中断返回 */
BX LR 
```

**5.3实现任务的调度**
任务调度的代码如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/29b31d92eed941ceaedb753d6888916e.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_19,color_FFFFFF,t_70,g_se,x_16)

SysTick_Handler为定时器中断程序，实现时间片轮流改变目标任务，并挂起PendSV_Handle中断，退出SysTick_Handler中断程序时进入PendSV_Handle中断程序。


## 6.运行结果

代码仿真运行如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/8ef932acd481466a8f61ca3a18828ae4.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_20,color_FFFFFF,t_70,g_se,x_16)![在这里插入图片描述](https://img-blog.csdnimg.cn/7efd0da3a7aa4e55b9d2e911b48ba2c9.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_20,color_FFFFFF,t_70,g_se,x_16)

**task_num0和task_num1这两个变量依次自加，代码实现任务轮流切换功能。**

**希望获取源码的朋友们在评论区里留言。**

>未完待续…
>
>实时操作系统系列将持续更新
>
>创作不易希望朋友们点赞，转发，评论，关注
>
>您的点赞，转发，评论，关注将是我持续更新的动力
>
>作者：李巍
>
>Github：liyinuoman2017
>
>CSDN：liyinuo2017
>
>今日头条：程序猿李巍
