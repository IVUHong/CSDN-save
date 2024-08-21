> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/weixin_43580890/article/details/132213672)

**目录**

[1、概述](#1%E3%80%81%E6%A6%82%E8%BF%B0)

[2、MCAL 配置](#2%E3%80%81MCAL%E9%85%8D%E7%BD%AE)

[2.1、Fls 配置](#2.1%E3%80%81Fls%E9%85%8D%E7%BD%AE)

[2.1.1、FlsConfigSet](#2.1.1%E3%80%81FlsConfigSet)

[2.1.2、FlsGeneral](#2.1.2%E3%80%81FlsGeneral)

[2.1.3、FlsSector](#2.1.3%E3%80%81FlsSector)

[3、测试代码及结果](#3%E3%80%81%E6%B5%8B%E8%AF%95%E4%BB%A3%E7%A0%81%E5%8F%8A%E7%BB%93%E6%9E%9C)

[3.1、测试代码](#3.1%E3%80%81%E6%B5%8B%E8%AF%95%E4%BB%A3%E7%A0%81)

[3.1.1、初始化](#3.1.1%E3%80%81%E5%88%9D%E5%A7%8B%E5%8C%96)

[3.1.2、执行代码](#3.1.2%E3%80%81%E6%89%A7%E8%A1%8C%E4%BB%A3%E7%A0%81)

[3.2、测试结果](#3.2%E3%80%81%E6%B5%8B%E8%AF%95%E7%BB%93%E6%9E%9C)

[3.2.1、Fls 写](#3.2.1%E3%80%81Fls%E5%86%99)

[3.2.2、Fls 读](#3.2.2%E3%80%81Fls%E8%AF%BB)

[3.2.3、Fls 擦除](#3.2.3%E3%80%81Fls%E6%93%A6%E9%99%A4)

1、概述
----

        FLS [驱动程序](https://so.csdn.net/so/search?q=%E9%A9%B1%E5%8A%A8%E7%A8%8B%E5%BA%8F&spm=1001.2101.3001.7020)为 DFlash 0 的初始化、读取、写入和擦除提供了定义良好的配置和标准服务。除此之外，我们还提供一些非 autosar 服务示例 Fls_17_Dmu_CompareWordsSync、Fls_17_Dmu_CancelNonEraseJobs、Fls_17_Dmu_VerifyErase，LFls_17_Dmu_VerifySectorErase、Fls_17_Dmu_GetNotifCaller 等。用户通过 FLS 驱动程序获得对底层 DFlash0 的封装访问。FLS 驱动程序的范围仅限于 DFlash0 Bank。

软硬件匹配关系如下：

![](https://i-blog.csdnimg.cn/blog_migrate/2706b63e40bf700af997dba2c2f52ff9.png)

        FLS 驱动程序使用 DMU 进行读、写、挂起、恢复、用户内容计数 (加固) 和擦除 DFlash0 内存等操作。

        以下由 SFR 标志通知的硬件事件在 FLS 驱动程序中使用: 在编程、擦除、读取或擦除挂起 / 恢复操作中发生错误时，会引发错误标志 Erase verify error (EVER): 当出现擦除验证错误时，由 Erase 命令设置此标志 PVER (Program verify error): 当出现程序验证错误时，由程序命令设置该标志保护错误 (PROER): 当在受保护的内存段上执行写或擦除命令时，该标志由硬件设置 Operation Error (OPER): 当 FSI (Flash standard interface) 出现错误时，由硬件设置该标志 SQER (Sequence Error): 当执行错误的 DMU 命令序列时，由硬件设置此标志忙结束(EOBM): 这个标志使中断能够报告擦除和编程操作的结束

FLS 可以关联到中断，但是感觉没必要。

![](https://i-blog.csdnimg.cn/blog_migrate/2f778615f5c07bb27712e05e393ae257.png)

         Fls_17_Dmu_MainFunction() 是 FLS 驱动提供的唯一调度函数。这个函数应该定期调用，这样它就可以在没有硬件中断支持的情况下处理作业。这个 api 是一个服务，用于执行 Flash 读、写、擦除和比较作业的处理。擦除或写操作的超时监视是基于 Fls_17_Dmu_MainFunction() 周期时间完成的。对于读取或比较，没有进行超时监控，因为通过 Fls_17_Dmu_MainFunction () cycletime 来监控的读取时间非常小。如果在连续循环中调用 Fls_17_Dmu_MainFunction() 作为 - while(1) 或以比 FlsCallCycle 更快的速度调用，则可能会发生意外超时。对于这种情况，用户需要使用 Fls_17_Dmu_ControlTimeoutDet () API 禁用超时监视。一旦禁用，将根本不监视超时。与超时相关的安全和 DET 错误也不会被触发。

2、[MCAL](https://so.csdn.net/so/search?q=MCAL&spm=1001.2101.3001.7020) 配置
-------------------------------------------------------------------------

        由于 Fls 的配置较为独立，与其他模块牵涉较少，所以，此处，直接描述配置了。

### 2.1、Fls 配置

#### 2.1.1、FlsConfigSet

用于存放 flash 驱动运行时配置参数的容器。    ![](https://i-blog.csdnimg.cn/blog_migrate/74760bb9aec35bca3902ee482fda1c0f.png)

 FlsDefaultMode：该参数为设备[初始化](https://so.csdn.net/so/search?q=%E5%88%9D%E5%A7%8B%E5%8C%96&spm=1001.2101.3001.7020)后 DFLASHO (data flash)的默认读取模式。假设以 MEMIF_MODE_SLOW 模式 (32 字节) 读取对用户来说是合理的，则选择默认值。MEMIF_MODE_FAST: 驱动程序工作在快速 (突发) 模式。MEMIF_MODE_SLOW: 表示驱动工作在慢速模式。有自己的时尚

FlsMaxReadFastMode：Flash 驱动器在快速模式下作业处理的一个周期中读取的最大字节数。该参数的配置也会影响 “比较” 和“空白检查”操作。注意: FlsMaxReadFastMode 配置的值应该大于 FlsMaxReadNormalMode 配置的值。因此，假设从数据闪存 (DFLASHO) 读取的字对齐的读地址大于的值，则设置默认值 FlsMaxReadNormalMode.

FlsMaxReadNormalMode：在正常模式下，Flash 驱动器的作业处理在一个周期内读取的最大字节数。该参数的配置也会影响 “比较” 和“空白检查”操作。假设从 DFLASHo 读取的地址是 word 对齐的，并且小于 FlsMaxReadFastMode 的值，则给出默认值。

#### 2.1.2、FlsGeneral

用于存放闪存驱动器通用参数的容器。这些参数总是预编译的。

![](https://i-blog.csdnimg.cn/blog_migrate/ffa368c91d2ae3c921eaabc3afe29200.png)

 上述主要是一些 API 的开关，需要注意的如下

FlsIfxFeeUse：假设使用英飞凌的 FEE，此处需要打开

FlsTotalSize：所选择芯片本身的 DFLASH 大小

FlsAcLoadOnJobStart：这个优点特殊，与 2xx 的不太一样，FlsAcLoadOnJobStart = TRUE 此参数指定从中复制和执行擦除命令周期的 RAM 地址。FlsAcLoadOnJobStart = FALSE 此处输入的值将被忽略，并使用擦除命令周期在 PFlash 中的位置。注意: 此参数未被使用，因此不支持。在 TC3xx 中，Pflash 和 Dflash 可以并行读取，因此不需要将 Dflash 访问代码加载到 RAM 中。

#### 2.1.3、FlsSector

这个可以人为分割多个逻辑块，因为是 DEMO，所以直接按照默认的了。

![](https://i-blog.csdnimg.cn/blog_migrate/4b4b385d59b4d16528a8386897454778.png)

 FlsNumberOfSectors：这个逻辑块分为几块，例如此处默认分为 2 块，每块大小为 0x20000，

FlsSectorSize：这个逻辑块的大小

FlsSectorStartaddress：在 0xAF000000 基础上的偏移地址是多少。

3、测试代码及结果
---------

### 3.1、测试代码

#### 3.1.1、初始化

```
#if FLSDEMOENABLE
			Fls_17_Dmu_Init(&Fls_17_Dmu_Config);
#endif
```

#### 3.1.2、执行代码

1ms 周期放置异步周期调度函数

```
#if FLSDEMOENABLE
			Fls_17_Dmu_MainFunction();
#endif
```

其他周期调度执行代码如下：

```
#if FLSDEMOENABLE
volatile uint8 Fls_DemoStateFlag = 0;
volatile Std_ReturnType Fls_EraseReturn 	 = 100;
volatile Std_ReturnType Fls_WriteReturn 	 = 100;
volatile Std_ReturnType Fls_ReadReturn  	 = 100;
volatile uint32         Fls_OperationAddress = 0;
uint8    Fls_WriteBuffer[16] = {0x01,0x02,0x03,0x04,0x05,0x06,0x07,0x08,0x09,0x10,0x11,0x12,0x13,0x14,0x15,0x16};
uint8    Fls_ReadBuffer[16]  = {0x00,0x00};
MemIf_StatusType  Fls_Dmu_GetStatuss = MEMIF_UNINIT;
MemIf_JobResultType Fls_Dmu_GetJobResults = MEMIF_JOB_OK;
void Fls_DemoFunction(void)
{
	if(Fls_DemoStateFlag == 1)
	{
		Fls_EraseReturn = Fls_17_Dmu_Erase(Fls_OperationAddress,0x1);
		Fls_DemoStateFlag = 10;
	}
	if(Fls_DemoStateFlag == 2)
	{
		Fls_WriteReturn = Fls_17_Dmu_Write(Fls_OperationAddress,Fls_WriteBuffer,16);
		Fls_DemoStateFlag = 20;
	}
	if(Fls_DemoStateFlag == 3)
	{
		Fls_ReadReturn = Fls_17_Dmu_Read(Fls_OperationAddress,Fls_ReadBuffer,8);
		Fls_DemoStateFlag = 30;
	}
 
	Fls_Dmu_GetStatuss = Fls_17_Dmu_GetStatus();
	Fls_Dmu_GetJobResults = Fls_17_Dmu_GetJobResult();
}
#endif
volatile uint32 Fls_IllegalStateRuning  = 0;
void Fee_17_IllegalStateNotification(void)
{
	Fls_IllegalStateRuning++;
}
```

### 3.2、测试结果

#### 3.2.1、Fls 写

当调用写函数的时候，能正确写入对应地址

![](https://i-blog.csdnimg.cn/blog_migrate/5af9c875cbdb3d2ea55e5fc6a832f8bf.png)

#### 3.2.2、Fls 读

 调用读之前数据如下

![](https://i-blog.csdnimg.cn/blog_migrate/44d29bef6fb123c03c6333e666b0d0fc.png)

 调用读之后

![](https://i-blog.csdnimg.cn/blog_migrate/f58552a04efec8ab9c50df054ab841af.png)

#### 3.2.3、Fls 擦除

 擦除之前

![](https://i-blog.csdnimg.cn/blog_migrate/d86e00927a473ec537e5a2953f6059fc.png)

 擦除之后

![](https://i-blog.csdnimg.cn/blog_migrate/a8125c9edf38508d7de4d6e9dcd42e04.png)