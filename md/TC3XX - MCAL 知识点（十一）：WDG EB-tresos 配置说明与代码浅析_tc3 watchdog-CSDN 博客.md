> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/weixin_43580890/article/details/131715528#t23)

**目录**

[1、概述](#t0)

[1.1、简介](#t1)

[1.2、Safety Watchdog](#t2)

[1.3、CPU Watchdogs](#t3)

[1.4、看门狗定时器的功能列表](#t4)

[1.5、Endinit 功能](#t5)

[1.6、访问 WDTxCON0 的密码](#t6)

[1.7、校验 WDTxCON0](#t7)

[1.8、修改访问 WDTxCON0](#t8)

[1.9、超时操作](#t9)

[1.10、寄存器解读](#t10)

[1.10.1、WDTSCON0（Safety WDT Control Register 0）](#t11)

[1.10.2、WDTCPUyCON0 (y=0-5)(CPUy WDT Control Register 0)](#t12)

[1.10.3、WDTSCON1(Safety WDT Control Register 1)](#t13)

[1.10.4、WDTCPUyCON1 (y=0-5)（CPUy WDT Control Register 1）](#t14)

[1.11、Mcal-Wdg 简介](#t15)

[2、EB-tresos 配置](#t16)

[2.1、配置目标](#t17)

[2.2、复位原理](#t18)

[2.3、MCU 配置](#t19)

[2.3.1、McuResetSettingConf](#t20)

[2.3.2、McuStmAllocationConf](#t21)

[2.4、ResourceM](#t22)

[2.4.1、ResourceMAllocation](#t23)

[2.5、 IRQ 配置](#t24)

[2.6、WDG 配置](#t25)

[2.6.1、WdgGeneral](#t26) 

[2.6.2、WdgSettingsConfig](#t27)

[2.6.3、WdgSettingsFast](#t28)

[2.6.4、WdgSettingsSlow](#t29)

[2.6.5、WdgTriggerTimerSetting](#t30)

[2.7、SMU 配置](#t31)

[2.7.1、SmuGeneral](#t32)

[2.7.2、SmuConfigSet](#t33)

[2.7.3、SmuCoreAlarmGroup](#t34)

[2.7.4、 Smu 的 IRQ 配置](#t35)

[3、测试代码及结果](#t36)

[3.1、测试代码](#t37)

[3.2、测试结果](#t38)

1、概述
----

### 1.1、简介

Tc3xx 包含了两类看门狗：

        · 一个安全看门狗

        · 每个核都有的定时看门狗

       · 每个[看门狗定时器](https://so.csdn.net/so/search?q=%E7%9C%8B%E9%97%A8%E7%8B%97%E5%AE%9A%E6%97%B6%E5%99%A8&spm=1001.2101.3001.7020)具有以下基本功能:

        · 可编程时基和重新加载值

        · 可编程密码保护与可配置的自动密码排序

        · 可编程时间戳检查与可编程窗口

        · 无效或缺失定时器刷新顺序导致安全警报

        WDT 提供了一种高度可靠和安全的方式来检测和恢复软件或硬件故障。的 WDT 有助于在用户指定的时间段内中止 [CPU](https://so.csdn.net/so/search?q=CPU&spm=1001.2101.3001.7020) 或系统的意外故障。

WDT 图示

![](https://i-blog.csdnimg.cn/blog_migrate/dbd39b674db8113be166b49601c44ce0.png)

         除了这个标准的 “看门狗” 功能之外，每个 wdt 都包含一个初始化结束 (ENDINIT) 功能，它可以保护关键寄存器免受意外写入的影响。

        Watchdogs 服务和修改 ENDINIT 位是在系统故障的情况下不允许的关键功能。为了保护这些功能，实现了一个复杂的方案，在访问 WDT 控制寄存器期间需要密码和保护位。任何写访问如果没有提供正确的密码或保护位的正确值，都被认为是系统故障，并导致看门狗报警。此外，即使在执行了有效的访问并且清除了 ENDINIT 位以提供对关键寄存器的访问之后，[Watchdog](https://so.csdn.net/so/search?q=Watchdog&spm=1001.2101.3001.7020) 也会对该访问窗口施加时间限制。如果 ENDINIT 位在此限制到期之前没有被正确设置，则假定系统出现故障。这些严格的要求，虽然不能保证，但是为系统操作的健壮性提供了高度的保证。

        配置选项是可用的，它使看门狗服务额外检查代码执行顺序和中间代码执行时间。如果启用了这些检查，那么任何不正确的顺序或超出限制的执行时间也将导致 SMU 警报请求。

        任意一个 WDT 过期都会触发 SMU 告警。可以对 SMU 进行编程，以提供中断或陷阱，以便在采取进一步行动 (例如复位设备或 CPU) 之前为恢复或状态记录提供一些时间。

### 1.2、Safety Watchdog

        安全看门狗定时器提供了一个整体的系统级看门狗，它独立于 CPU 看门狗定时器，还提供了另一种防止意外写入安全关键系统寄存器的保护。当启用安全 WDT 时，如果在用户可编程的时间段内没有提供服务，则会引起 SMU 报警请求。CPU 必须在此时间间隔内服务安全 WDT，以防止这种情况发生。对安全 WDT 超时的响应可以在 SMU 中配置。因此，定期对安全 WDT 进行维修，以确认系统是否按预期运行。

        通常，SCU 写保护 (ACCEN) 将被配置为只有受限制的“安全”CPU 才能配置安全关键功能。这包括为安全监督机构提供服务的能力。另外，Safety Watchdog Timer disable/enable/configuration 功能需要输入 Safety ENDINIT 密码。

        在 SCU 寄存器概述表和其他模块寄存器表中标记为 “SE” 的寄存器被认为是安全关键系统的静态属性，并且在初始化后都是写保护的。如果安全 ENDINIT 位被清除，这个写保护可能会被暂时移除。

### 1.3、CPU Watchdogs

        单独的 CPU 看门狗计时器提供了监视单独的 CPU 执行线程的能力，而不需要软件来协调公共看门狗的共享使用。

        当 CPU WDT 使能时，如果在用户可编程的时间段内没有正确服务，则会引起 SMU 告警请求。CPU 必须在此时间间隔内服务其 CPU WDT，以防止这种情况发生。SMU 可以配置对每个 CPU 看门狗超时的响应。因此，对 CPU WDT 的定期维护可以确认相应的 CPU 正在按预期执行软件序列。

        复位后，CPU0 运行，CPUO Watchdog Timer 自动启动。

        其他 CPU 最初处于 HALT 状态，因此它们对应的看门狗定时器被禁用。其他 CPU Watchdog Timers 默认不配置为生成超时重置，但可以启用。CPU 看门狗只能由相应的 CPU 配置、启用或禁用。

        在 SCU 寄存器概述表和 CPU 寄存器表中标记为 “CEy”(y= CPU 编号) 的寄存器是与 CPU 相关的关键寄存器，在运行时被认为不太可能被更改。它们在初始化后都是写保护的。如果清除对应的 cpu (y-CPU 号)Watchdog ENDINIT 位，该写保护可能会暂时取消。

### 1.4、看门狗定时器的功能列表

·16 位看门狗定时器。

· 可选输入频率: fspb/64, fspb/256 或 fspb/16384。

· 正常看门狗定时器操作的 16 位用户自定义重载值，超时模式的固定重载值

· 结合相应的 ENDINIT 位监控修改

· 复杂的密码访问机制，具有用户可定义的密码字段

· 访问错误检测: 密码无效 (第一次访问时) 或保护位无效 (第二次访问时) 触发告警请求 SMU

· 时间和逻辑监控功能:

     - 可选的代码序列检查。码序签名识别错误会触发向 SMU 发送告警请求

     - 可选的代码执行时间检查。超出预期限制的代码执行时间将触发向 SMU 发出警报请求。

· 溢出错误检测: 当 WDT 计数器溢出时，向 SMU 发出告警请求

· 可禁用看门狗功能; 访问保护和 ENDINIT 位监控功能保持开启状态

· 可配置的机制，防止看门狗在未服务的安全警告告警后重新加载，以确保未服务的警告导致 SMU 硬件响应

### 1.5、Endinit 功能

        在系统或应用程序的初始化过程中，有许多寄存器通常只被编程一次。在应用程序正常运行期间修改这些寄存器可能会对模块或整个系统的整体操作产生严重影响。

        虽然监督模式和访问保护方案提供了一定程度的保护，防止无意的修改，但它们可能不足以防止对系统关键寄存器的所有意外访问。

        为这些寄存器提供了额外的保护，称为 Endinit(“初始化结束”)。Endinit 是一种写保护方案，它只允许在特定时间内写，并且几乎不可能对受此特性保护的寄存器进行意外修改。

        Endinit 特性由每个 WDT 控制寄存器中包含的 Endinit 位组成。通过 Endinit 保护的寄存器决定是否启用写操作。只有当相应的 ENDINIT = 0 并且处于管理者模式时候，才会启用写操作。如果此条件不为真，则写入尝试将被丢弃，并且在这种情况下不会修改寄存器内容。

        为了获得最高级别的健壮性，对 ENDINIT 位的写操作受到 WDT 中实现的高度安全的访问保护方案的保护。这是一个复杂的过程，使得无意中修改 ENDINIT 位几乎是不可能的。此外，每个 WDT 通过每次软件通过清除相应的 ENDINIT 位打开对临界寄存器的访问时启动一个超时序列来监视 ENDINIT 位的修改。如果在再次设置相应的 ENDINIT 位之前，超时时间已经结束，则假定软件故障，产生看门狗故障响应。

        下面将介绍 WDT 的访问保护方案和 Endinit 超时操作。在每个模块 (包括 ScU 本身) 的寄存器概述表中，通过每个 Endinit 类型保护的寄存器在描述写访问的列中标识如下:

·．. “CEy”—CPU 关键寄存器。CPU WDT ENDINIT=0 (y=CPU 编号) 时可写

·“E”- 系统关键寄存器 - 当任何 (一个或多个)cpu 看门狗定时器 ENDINIT=0 或 EICONO.ENDINIT=0 时可写。

·．“SE”- 安全关键寄存器 - 仅当安全看门狗定时器 ENDINIT=0 或 SEICON0.ENDINIT=0 时候可写。

·. 以上任何一个都不能在任何时间访问

用于解锁各种 ENDINIT 写保护模式的选项如下图所示。

![](https://i-blog.csdnimg.cn/blog_migrate/ab5f48efceddac885e13903e2f893843.png)

 注意点

        清除 ENDINIT 位需要一些时间。在清除 ENDINIT 位之后访问受 ENDINIT 保护的寄存器必须在真正清除 ENDINIT 位时才进行。作为一种解决方案，ENDINIT 位应该在 ENDINIT 位被清除后第一次访问受 ENDINIT 保护的寄存器之前回读一次。

### 1.6、访问 WDTxCON0 的密码

        必须编写正确的密码来注册 WDTxCON0 (x=CPU, y=CPU 编号，或 S)，以便将其解锁以进行修改。软件必须事先知道正确的密码，或者在运行时计算它。为了提供独立的看门狗功能，每个看门狗定时器 (x=CPUy, y=CPU number，或 S) 的密码可以不同，程序流具有独立的看门狗功能。

        安全看门狗密码寄存器 WDTSCON0 由通用 SCU 保护方案保护，该方案只允许配置的主 (s) 具有写访问。

![](https://i-blog.csdnimg.cn/blog_migrate/8eaff5a673ac97c73870be3a992d0c20.png)

         特定于 CPU 的看门狗密码寄存器 WDTCPUyCON0 被单独保护，因此它们只能由相应的 CPU 写入.

        看门狗可以在安全应用程序中使用，以提供一个恢复时间段，在此期间软件可能试图从安全警报警告中恢复。为了确保 CPU 故障不允许错误被忽略，提供了一个选项来防止看门狗在 SMU 处于 fault 状态时解锁。此选项可以通过位 WDTxCON1.UR 启用。

        密码有效且 SMU 状态满足 WDTxSR 的要求。那么一旦密码访问完成，WDTxCON0 将被解锁。解锁条件将由 WDTxCONO.LCK=0 表示。为确保正确的服务顺序，密码访问仅在 WDTxCONO。在访问之前设置了 LCK 位。

        如果在密码访问过程中写入错误的密码值到 WDTxCON0，则产生看门狗访问错误条件的存在 WDTxSR.AE。设置访问错误，并向 SMU (Safety Management Unit) 发送告警请求。

        14 位用户自定义密码 WDTxCON0.PW 为根据应用程序的需要调整密码要求提供了额外的选项。例如，它可以用来检测意外的软件循环，或者监视例程的执行顺序。

        密码要求如下表所示。存在各种选项，下面将更详细地描述它们：

![](https://i-blog.csdnimg.cn/blog_migrate/a49bb3fc3c48e9864a9504ee9e90a422.png)

         在静态密码模式下 (WDTxSR.PAS=0)，密码只能通过有效的修改权限进行修改。密码访问的设计使得不能简单地读取寄存器并重新写入它。在重新写入之前，必须反转 (切换) 一些密码读位。这可以防止简单的故障通过简单的读 / 写序列意外解锁 WDT。

        如果启用了自动密码排序 (WDTxSR.PAS=1)，则每次密码检查 (即密码访问或检查访问) 后密码自动更改。期望的下一个密码遵循基于 14 位斐波那契 LFSR(线性反馈移位寄存器)的伪随机序列，其特征多项式为 x14+x13+x12+x2+1。修改访问也可以提供初始密码(或随后的手动密码更新)。

![](https://i-blog.csdnimg.cn/blog_migrate/32026265feb85910e0ad52fe1c040ba5.png)

         如果没有启用时间检查 (WDTxSR.TCS=0)，那么 WDTxCON0 寄存器的 REL 字段必须在密码访问期间用现有的重新加载值简单地重写。

        如果启用了时间检查 (WDTxSR.TCS=1)，则 WDTxCON0 寄存器的 REL 字段必须用当前 WDT 计数值的反转 (位翻转) 估计来写。该估计的可接受误差范围 (在 WDT 时钟周期内) 由 WDTxSR.TCT 的值指定。如果写的值估计超出了 WDTxSR 的范围。TIM +/- WDTxSR.TCT，则表示 SMU 告警状态。该机制可以检查自上次 WDT 重新启动以来经过的程序执行时间。请注意，当 WDT 在超时模式下运行时 (例如在访问 endinit 保护的寄存器之后)，仍然需要对密码或检查访问进行时间检查比较。

### 1.7、校验 WDTxCON0

        检查访问与密码访问相同，只是锁位不被清除，因此不允许后续的修改访问。在满足写数据要求的情况下，Check Access 不会触发 SMU 告警请求。Check Access 只能在 LCK 位被设置的情况下执行。

        如果在 Check Access 过程中，WDTxCON0 上写了不正确的值 (x=CPU, y=CPU 编号，或 S)，则存在 Watchdog Access Error 条件。WDTxSR。设置 AE，并向 SMU (Safety Management Unit) 发送告警请求。

### 1.8、修改访问 WDTxCON0

        如果 WDTxCONO (x=CPUy, y=CPU number, or S) 通过 Password Access 解锁成功，则后续对 WDTxCON0 的写访问可以修改该 WDTxCON0。但是，这种访问还必须满足某些要求，才能被接受并被视为有效。下表列出了所需的位模式。如果访问不遵循这些规则，则检测到看门狗访问错误条件，位 WDTxSR.AE 被设置，向 SMU (Safety Management Unit) 发送告警请求。修改访问完成后，WDTxCON0。再次设置 LCK，自动重新锁定 WDTxCON0。在可以再次修改寄存器之前，必须再次执行有效的密码访问。

![](https://i-blog.csdnimg.cn/blog_migrate/f5163ec93ec13b3b9335a6093508377b.png)

         要重新启用访问，必须首先使用有效的 Password access 解锁 WDTxCON0。在后续有效的 Modify Access 中，ENDINIT 可以被清除。对受 endinit 保护的寄存器的访问现在再次打开。但是，当 WDTxCON0 被解锁后，WDT 会自动切换到超时模式。因此访问窗口是有时间限制的。超时模式仅在再次设置 ENDINIT 后终止，需要另一个有效密码和对 WDTxCON0 的有效修改访问。

        在某些应用程序中，可能不使用 WDT 并将其禁用 (WDTxSR.DS=1)，尽管不建议这样做。在其他应用程序中，可以使用 WDT 时间戳特性，刷新之间的 WDT 访问是不可取的。在这种情况下，仍然可以使用 ENDINIT 全局控制寄存器 (EICONx) 启用对 ENDINIT 保护寄存器的临时访问。

### 1.9、超时操作

        所有 WDT 都使用 SPB 时钟 fspb。每个 WDT 前的时钟分频器提供三个 WDT 计数器频率，fspb / 64, fspb / 256 和 fspb / 16384。

        看门狗超时时间的一般计算公式为:

period = ((2^16- startvalue) *divider)/ fSPB

        参数 startvalue 表示计算超时时间的固定值 FFFC 和用户可编程的重新加载值 WDTxCON0.REL 用于计算正常周期。

        这句话的意思默认的 startvalue = 0xFFFC, 计算超时模式，通过 WDTxCON0.REL 重新装载正常周期。

        参数分频器表示 WDTxcON1 选择的用户可编程源时钟分频。IRx，可以是 64、256 或 16384。

每个看门狗定时器可以在三种不同的工作模式之一:

Time-Out Mode

Normal Mode

Disable Mode

下面概述了这些模式，以及 WDT 如何从一种模式切换到另一种模式。

**Time-Out Mode**

        应用复位后进入 CPU WDT 超时模式。“应用复位” 后会禁用其他 CPU，这个时候 CPU0 是没有被禁用的。当对寄存器 WDTxCON0 执行有效的密码访问时，也会进入超时模式。超时模式由位 WDTxSR.T0= 1。定时器设置为 0xFFFC，并开始向上计数。只有设置 ENDINIT = 1 并设置正确的访问顺序，“超时模式” 才能正常退出。如果对 WDT 进行了错误的访问，或者在设置 ENDINIT 之前定时器溢出，则向 SMU (Safety Management Unit) 发送告警请求。

        根据超时请求位 WDTxCON1.DR 的状态，从超时模式正确退出可以是正常模式或禁用模式。

**Normal Mode**

        在正常模式 (DR = 0) 下，WDT 以标准看门狗方式工作。定时器设置为 WDTxCON0.REL，并开始计数。必须在计数器溢出之前进行喂狗。服务是通过对控制寄存器 WDTxCON0 的适当访问顺序执行的。这将进入超时模式。

        如果 WDT 在计时器溢出之前没有得到服务，则假定系统出现故障。正常模式终止，并向安全管理单元 (SMU) 发送报警请求。

**Disabled Mode**

        应用复位后，除 CPU0 外，所有 CPU WDT 都处于禁用模式。为不需要 WDT 功能的应用程序提供了禁用模式。可以从 Time-Out Mode 到 disabled mode，设置 WDTxCON1.DR 的标志位。当请求 disabled mode 和位 WDTxCON0.ENDINIT 被设置时，进入禁用模式。定时器在此模式下停止。但是，关闭 WDT 只会使其停止执行标准的看门狗功能，不需要对 WDT 进行及时服务。它不会禁用超时模式。如果以禁用模式访问 WDTxCONO 寄存器，如果访问有效，则进入超时模式，如果访问无效，则向 SMU (Safety Management Unit)发送告警请求。因此，ENDINIT 监视器功能以及 (一部分) 系统故障检测仍将处于活动状态。

        如果在应用程序中使用 WDT 并且启用了 WDTxSR。DS = 0)，必须定期检修，防止溢流。

        服务分两个步骤执行，一个是有效的 Password Access，然后是有效的 Modify Access。有效的密码访问 WDTxCON0 会自动将 WDT 切换为超时模式。

        因此，修改访问权限必须在超时时间之前执行，否则会向 SMU (Safety Management Unit) 发送告警请求。在下次修改访问时，严格要求 WDTxCON0.ENDINIT 写入 1。

        注意: 为了执行正确的服务，必须将 ENDINIT 写入为 1，即使它已经设置为 1。

        更改重新加载值 WDTxCON0.REL 或用户可定义的密码 WDTxCON0。不需要 PW。当 WDT 业务正常执行时，超时模式终止，WDT 切换回原来的工作模式，WDT 业务完成。

        检查访问是可选的，可以在 WDT 被锁定时的任何时间执行。

### 1.10、寄存器解读

#### 1.10.1、**WDTSCON0****（****Safety WDT Control Register 0****）**

        这个寄存器用检查访问和密码访问中的检查数据写入。在修改访问期间，它还存储计时器重新加载值、密码更新和相应的初始化结束 (ENDINIT) 控制位。每个看门狗 x 都有一个 WDTxCONO 寄存器(x=CPU, y= CPU 编号，x=S)。对于安全看门狗定时器为 x=S。

![](https://i-blog.csdnimg.cn/blog_migrate/69ffbd5baab02e1ef3891376adb5423e.png)

<table border="1" cellspacing="0"><tbody><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><p><strong>ENDINIT</strong></p></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><p>在密码访问 (Password Access) 或检查访问 (Check Access) 期间，必须用'1'写入该位 (尽管此写入仅用于密码保护机制而不存储)。在修改访问期间，这个位必须写入所需的 ENDINIT 更新值。这个位可以用来访问具有“SE” 保护的寄存器，但是备用寄存器是 SEICONO。建议使用 ENDINIT，这样 Watchdog Timer 就不会受到影响。</p><p>0 允许访问受 endinit 保护的寄存器 (默认在 ApplicationReset 之后)</p><p>1 不允许访问受 endinit 保护的寄存器。</p></td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><p><strong>LCK</strong></p></td><td><p>锁位控制对 WDTxCON0 的访问</p><p>LCK 的当前值由硬件控制。当 WDTxSR 时，在有效的密码访问 WDTxCON0 后清除 WDTxSR.US 为 0(或当 WDTxSR.US 为 1,SMU 处于 RUN 模式)，并且在有效的修改访问 WDTxCON0 后，它会自动重新设置。在写入 WDTxCON0 时，写入该位的值仅用于密码保护机制，而不存储。</p><p>0 不锁定寄存器 WDTxCON0</p><p>1 锁定寄存器 WDTxCON0，应用复位后的默认值</p></td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><p><strong>PW</strong></p></td><td><p>访问 WDTxCON0 的用户可定义密码字段</p><p>该位字段在修改访问期间写入初始密码值。</p><p>从这个位域读取返回这个初始密码，但是位 [7:2] 被颠倒(切换)，以确保简单的读 / 写不足以服务 WDTx。</p><p>如果对应 WDTxSR.PAS = 0，那么在密码访问或检查访问期间，这个位字段必须与它的当前内容一起写入。</p><p>如果对应 WDTxSR。PAS=1，则在密码访问或检查访问期间，该位字段必须与 LFSR 序列中的下一个密码一起写入重置后的默认密码为 “00000000111100g”</p></td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><p><strong>REL</strong></p></td><td><p>WDT 的重新加载值 (也是时间检查值)</p><p>重载值可以在修改访问 WDTxCON0 期间更改 (ApplicationReset 后的默认值是 FFFCH)。如果“看门狗定时器” 启用并处于“正常定时器模式”，则在正确的看门狗服务后，它将从该值开始计数。</p><p>从这个位域读取总是返回当前的重新加载值。在密码访问或检查访问期间，此位域可用于附加检查。此类检查期间的写操作对重新加载值没有影响。</p><p>如果对应 WDTxSR.TCS=0，那么在密码访问或检查访问期间，这个位字段必须与它的当前内容一起写入。</p><p>如果对应 WDTxSR.TCS=1，那么这个位域必须用当前 WDTxSR 的反向估计来写。密码访问或检查访问时的</p><p>WDTxSR.TIM 值。</p></td></tr></tbody></table>

#### 1.10.2、**WDTCPUyCON0 (y=0-5)(CPUy WDT Control Register 0)**

        这个寄存器用检查访问和密码访问中的检查数据写入。在修改访问期间，它还存储计时器重新加载值、密码更新和相应的初始化结束 (ENDINIT) 控制位。每个看门狗 x 都有一个 WDTxCONO 寄存器(x=CPU, y= CPU 编号，x=S)。对于 CPU 看门狗定时器为 x=CPU, y=CPU 编号。

![](https://i-blog.csdnimg.cn/blog_migrate/446bf308c6cb84ac850072fab623aa8c.png)

![](https://i-blog.csdnimg.cn/blog_migrate/7e6039612ba2e01b6f3d096eda0dc55d.png)

#### 1.10.3、**WDTSCON1(Safety WDT Control Register 1)**

        每个看门狗定时器 x (x=CPU, y=CPU 编号，x=S) 都有一个 WDTxCON1 寄存器。对于安全看门狗定时器为 x=S。寄存器 WDTxCON1 管理安全 WDT 的操作。它包括禁用请求、密码配置和频率选择位。每个 WDTxCON1 寄存器都由写保护 Safety ENDINIT (SE).

        WDTSCON1 有一个额外的位 CLRIRF，可用于清除用于检测双 SMU(例如 WDT) 复位的内部复位状态。

![](https://i-blog.csdnimg.cn/blog_migrate/88f9f603ca7573022fb90e2849df5492.png)

<table border="1" cellspacing="0"><tbody><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><p><strong>CLRIRF</strong></p></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><p>清除内部复位标志</p><p>该位用于请求清除内部标志，该标志表明先前的 SMU 重置是否已经被请求。修改后，内部标志仅在重新断言安全结束 (SE) 时被清除。只要没有断言 Safety ENDINIT(SE)，内部标志就保持不变，并继续确定对进一步 SMU 重置请求的响应。当 Safety ENDINIT 被重新插入时，内部标志和这个位一起被清除。</p><p>0 没动作</p><p>1 清除 SMU 所请求的复位标志位</p></td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><p><strong>IR0</strong></p></td><td><p>输入频率请求控制 - IR1、IRO</p><p>Bit IR0 和 IR1 应该一起编程来确定 WDTx 定时器频率。</p><table border="1" cellspacing="0"><tbody><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><p>IR0</p></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><p>IR1</p></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><p>频率</p></td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><p>0</p></td><td><p>0</p></td><td><p><strong>f</strong><strong>SPB/16384</strong></p></td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><p>1</p></td><td><p>1</p></td><td><p><strong>--</strong></p></td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><p>1</p></td><td><p>0</p></td><td><p><strong>f</strong><strong>SPB/256</strong></p></td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><p>0</p></td><td><p>1</p></td><td><p><strong>f</strong><strong>SPB/64</strong></p></td></tr></tbody></table><p></p></td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><p><strong>DR</strong></p></td><td><p>禁用请求控制位</p><p>0 请求使能 WDTx</p><p>1 请求不使能 WDTx</p></td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><p><strong>IR1</strong></p></td><td><p>输入频率请求控制 - IR1、IRO</p><p>Bit IR0 和 IR1 应该一起编程来确定 WDTx 定时器频率。</p><table border="1" cellspacing="0"><tbody><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><p>IR0</p></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><p>IR1</p></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><p>频率</p></td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><p>0</p></td><td><p>0</p></td><td><p><strong>f</strong><strong>SPB/16384</strong></p></td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><p>1</p></td><td><p>1</p></td><td><p><strong>--</strong></p></td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><p>1</p></td><td><p>0</p></td><td><p><strong>f</strong><strong>SPB/256</strong></p></td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><p>0</p></td><td><p>1</p></td><td><p><strong>f</strong><strong>SPB/64</strong></p></td></tr></tbody></table><p></p></td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><p><strong>UR</strong></p></td><td><p>解锁限制请求控制位、</p><p>0 请求取消 WDTx 解锁的 SMU 限制</p><p>1 请求开启 WDTx 解锁的 SMU 限制</p></td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><p><strong>PAR</strong></p></td><td><p>密码自动序列请求位</p><p>0 每次修改访问权限或检查访问权限后，不要求自动更改密码</p><p>1 要求每次修改访问权限或检查访问权限后的密码自动顺序</p></td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><p><strong>TCR</strong></p></td><td><p>计数器检查请求位</p><p>0 在修改访问权限或检查访问权限时，只请求检查 REL 字段是否写入了已有的 REL 值.</p><p>1 在修改访问或检查访问期间，请求检查 REL 字段是否写入正确的 TIM 计数 (在 WDTxSR.TCT 的公差范围内)。</p></td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><p><strong>TCTR</strong></p></td><td><p>计时器检查容差请求</p><p>WDTxSR.TCT 控制的容忍度</p></td></tr></tbody></table>

#### 1.10.4、**WDTCPUyCON1 (y=0-5)****（****CPUy WDT Control Register 1****）**

#### ![](https://i-blog.csdnimg.cn/blog_migrate/6f65fa3d37276b459521d7c1bc06d139.png)

![](https://i-blog.csdnimg.cn/blog_migrate/0b17953e5f5b550f834551cc46d2c0e3.png)

### 1.11、Mcal-Wdg 简介

在 AUTOSAR 中，看门狗栈实现了三种监控机制:

实时监控: 用于周期软件的定时监控

期限监督: 针对非周期软件

逻辑监督: 用于监督执行顺序的正确性

看门狗 (WDG) 驱动实现 AUTOSAR(4.2.2)中规定的看门狗 SWS，提供初始化、改变操作模式和设置触发条件 (超时) 等服务，供看门狗硬件 (WDT) 用户使用。WDG 驱动的用户包括: 模式管理模块(用于初始化)，WdgM 通过 Wdglf 设置触发条件和驱动运行模式设置。WdgM 实施上述监督机制。AUTOSAR 扩展触发概念，以支持窗口看门狗模式。这需要使用硬件计时器来自触发 WDT，直到满足用户设置的触发条件。为此，WDG 驱动程序提供了选择两个计时器之一的选项，即 STM 或 GTM 计时器。由于代码库对于所有 cpu 都是通用的，为了避免出现未使用的代码，定时器选择对于所有 wdt 都是静态的(预编译)(驱动程序对所有 wdt 使用 GTM 或 STM)。

STM 依赖关系：

WDG 驱动程序依赖于 STM 的比较匹配事件，这是服务看门狗计时器所必需的。看门狗定时器使用 STMxTIM0 (x = Core Id) 寄存器来计算特定于模式的刷新时间，以服务于看门狗定时器。每个看门狗定时器被映射到一个唯一的 STM 定时器比较寄存器。

McalLib、MCU 和 WDG 驱动程序是 STM 的用户。为了避免冲突，STM 比较寄存器的分配是通过 MCU 驱动程序执行的，WDG 驱动程序使用的 STM 比较寄存器是专门分配给它的。

![](https://i-blog.csdnimg.cn/blog_migrate/c37c41a4757a1068f93d6995aa10a010.png)

2、EB-tresos 配置
--------------

### 2.1、配置目标

<table border="1" cellspacing="0"><tbody><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><p>所需模块</p></td><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><p>功能</p></td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><p>MCU</p></td><td><p>提供时钟以及配置 SMU 复位类型</p></td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><p>STM</p></td><td><p>提供比较寄存器</p></td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><p>SMU</p></td><td><p>产生复位</p></td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><p>WDG</p></td><td><p>配置看门狗信息</p></td></tr><tr><td data-darkreader-inline-border-top="" data-darkreader-inline-border-right="" data-darkreader-inline-border-bottom="" data-darkreader-inline-border-left=""><p>ResourceM</p></td><td><p>分配资源</p></td></tr></tbody></table>

### 2.2、复位原理

        注意一下：下图为找出来的 AURIX1G 的图示，执行原理是一样的，只是 Alarm 对不上而已，下图告诉我们 WDG 产生信号，给到 SMU 产生动作，例如复位。 

![](https://i-blog.csdnimg.cn/blog_migrate/31855e894db7925639a5ce1be6ff3111.png)

### 2.3、MCU 配置

#### 2.3.1、McuResetSettingConf

        MCU 处配置 SMU 产生什么样子的动作，此处不配置，实际测试也会产生复位，但是在调试器的信息窗口看不到，只是计数值重新从 0 开始计数了，然后也只能产生一次复位，所以此处需要执行配置的。

![](https://i-blog.csdnimg.cn/blog_migrate/6db77fbd13eef88664bb862adab79e5e.png)

#### 2.3.2、McuStmAllocationConf

利用 STM0 的 CMP1 进行比较，之前的 STM 定时器实验已经占用了 CMP0。

![](https://i-blog.csdnimg.cn/blog_migrate/ec1692681b10b9126322a43929390ef4.png)

### 2.4、ResourceM

#### 2.4.1、ResourceMAllocation

将 STM 的资源分配出来，此处不配置会产生错误的

![](https://i-blog.csdnimg.cn/blog_migrate/783ae0fd90cb2da9c2061735c1e21f66.png)

### 2.5、 IRQ 配置

配置 STM0 CMP1 的中断 SR1

![](https://i-blog.csdnimg.cn/blog_migrate/dfd353f6107d2cdca827b1b6183c4b92.png)

 IRQ 代码问题

![](https://i-blog.csdnimg.cn/blog_migrate/e39a978d6b034a3af07663c22bea030d.png) 注意一个问题：如下图

![](https://i-blog.csdnimg.cn/blog_migrate/d4e0c685982ae6dd44cdacf3cbe944a0.png)

### 2.6、WDG 配置

#### 2.6.1、WdgGeneral 

![](https://i-blog.csdnimg.cn/blog_migrate/b1530f38c6e8535311f76a7491a7678e.png)

 WdgDisableAllowed：是否允许在正在运行时禁用 WDG，一般不允许的。

WdgInitialTimeout/ WdgMaxTimeout：不可配置，属于禁用功能。

WdgTriggerTimerSelection：触发时间选择，选择 STM 还是 GTM，假设选择了 GTM，GtmTimerConfiguration 才可以进行配置的，此处选择了 STM 所以 GtmTimerConfiguration 是黄色的不可配置界面。

#### 2.6.2、WdgSettingsConfig

![](https://i-blog.csdnimg.cn/blog_migrate/c6fc86f1996a1455488e446bf53d9216.png)

 WdgCPUInitialTimeout：这是初始窗口周期，一旦为核心调用 Wdg_17_Scu_Init，它就处于活动状态。它用于计算初始化后用于服务 WDT 的触发计数器的值。该参数由英飞凌定义。用户可以根据应用的需要修改默认值，默认值应在指定的边界值范围内。

WdgCPUMaxTimeout：这是特定于核心的看门狗定时器的最大窗口周期。它被认为是这个特定核心的最大窗口期。

WdgCPUInitialPassword：* 密码访问的初始密码。

WdgSystemClockRef：参考时钟选择，来自于 SPB

WdgDefaultMode：默认模式选择

#### 2.6.3、WdgSettingsFast

![](https://i-blog.csdnimg.cn/blog_migrate/79c9d5318242b93a57ebe7cded9a2244.png)

 WdgFastModeTimeoutValue：WDT 快速模式超时 (以秒为单位)，这个值可以重新设置。

#### 2.6.4、WdgSettingsSlow

![](https://i-blog.csdnimg.cn/blog_migrate/2361a2777f9922498020382e84d3863c.png)

 WdgSlowModeTimeoutValue: WDT 慢速模式超时 (以秒为单位)，这个值可以重新设置。

#### 2.6.5、WdgTriggerTimerSetting

![](https://i-blog.csdnimg.cn/blog_migrate/37e558fd348f2fec85a98eb4b07fb460.png)

WdgSlowRefreshTime：WDT 慢模式 Gtm/Stm 回调周期 (以秒为单位)。

WdgFastRefreshTime：WDT 快模式 Gtm/Stm 回调周期 (以秒为单位)。

注意：超时狗只要在值溢出之前喂狗即可，例如 1S 的狗，尽量在 800ms 之内喂狗较好一些。

### 2.7、SMU 配置

SMU 配置会在 SMU 文章里面进行详细的介绍，此处仅仅说明为什么 WDG 这么配置。

#### 2.7.1、SmuGeneral

![](https://i-blog.csdnimg.cn/blog_migrate/2521c9772c193d41f6b7cf945250a788.png)

#### 2.7.2、SmuConfigSet

![](https://i-blog.csdnimg.cn/blog_migrate/af54bd60177fc401f16b4623ba7043e5.png)

![](https://i-blog.csdnimg.cn/blog_migrate/714114d93383f400638fb91a1b1f996e.png)

#### 2.7.3、SmuCoreAlarmGroup

下图两个红框里面配置其中一个复位就行了

![](https://i-blog.csdnimg.cn/blog_migrate/cc45fe26acc408759ac9c2e0f8ab0267.png)

 为什么上述配置呢？见下图，属于英飞凌预先定义好的。

![](https://i-blog.csdnimg.cn/blog_migrate/daf8698b93be62cc82448fa7deacca33.png)

#### 2.7.4、 Smu 的 IRQ 配置

Smu 的 IRQ 配置需要手动配置 Demo 并没有给出，手写代码见代码分析

3、测试代码及结果
---------

### 3.1、测试代码

1、SMU 中断代码

```
void IrqSmu_Init(void)
{
  IRQ_SFR_MODIFY32 (SRC_SMU0.U,  0xFFFFFFFF, (0x0000 | 0x1));
  IRQ_SFR_MODIFY32 (SRC_SMU1.U,  0xFFFFFFFF, (0x0000 | 0x0));
  IRQ_SFR_MODIFY32 (SRC_SMU2.U,  0xFFFFFFFF, (0x0000 | 0x0));
 
  SRC_SMU0.B.SRE = 1;
  SRC_SMU1.B.SRE = 0;
  SRC_SMU2.B.SRE = 0;
}
 
volatile uint8 smu_isr_status = 0xEE;
IFX_INTERRUPT(SMU0_ISR, 0, 0x09)
{
  /* Enable Global Interrupts */
  ENABLE();
  smu_isr_status = 1;
}
```

2、初始化

```
IrqStm_Init();
	/*SRE bit needs to be set to enable interrupts*/
	SRC_STM0SR1.B.IOVCLR 	= 1U;
	SRC_STM0SR1.B.CLRR 		= 1U;
	SRC_STM0SR1.B.SWSCLR 	= 1U;
	SRC_STM0SR1.B.SRE 		= 1U;
 
	IrqSmu_Init();
	Smu_Init(&(Smu_Config));
	Smu_ActivateRunState(SMU_RUN_COMMAND);
 
    Wdg_17_Scu_Init(&Wdg_17_Scu_Config_0);
    Wdg_17_Scu_SetMode(WDGIF_SLOW_MODE) ;
Wdg_17_Scu_SetTriggerCondition (1000);
```

3、执行代码

```
volatile uint8 WDG_ServeTrigger = 1;
			if(WDG_ServeTrigger == 1)
			{
				Wdg_17_Scu_SetTriggerCondition (1000);
			}
```

### 3.2、测试结果

不喂狗复位结果

![](https://i-blog.csdnimg.cn/blog_migrate/75232b93626d8d2c81b1d436de2272fa.png)

 不喂狗寄存器显示

![](https://i-blog.csdnimg.cn/blog_migrate/2287ac31edc5c234948dd9ae5a8ac060.png)

如下图

![](https://i-blog.csdnimg.cn/blog_migrate/144f305c81638ed1ed20f0f8c9e71354.png)