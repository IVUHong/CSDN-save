> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/qq_40462906/article/details/136257068?ops_request_misc=&request_id=&biz_id=102&utm_term=tc3XX%E9%85%8D%E7%BD%AEWDG&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduweb~default-0-136257068.142^v100^pc_search_result_base4&spm=1018.2226.3001.4187)

**1 概述**
--------

 TC3xx 包含以下看门狗计时器：  
     . 一个安全的看门狗计时器  
     . 每个 CPU 独立的看门狗计时器  
有以下基本功能：  
    . 可编程的时间基础和重新加载值  
    . 可编程的密码保护与可配置的自动密码排序  
    . 可编程的时间戳检查与可编程窗口  
    . 无效或丢失定时器刷新序列导致安全警报

### ![](https://i-blog.csdnimg.cn/blog_migrate/a5156194f49f7f7451ee8ae8e8c4a198.png)

  
2 安全看门狗
----------

      安全监视计时器提供了一个独立于 [CPU](https://so.csdn.net/so/search?q=CPU&spm=1001.2101.3001.7020) 监视计时器的整体 CPU 监视器，并提供了另一种保护，防止意外写入安全关键系统寄存器。当 Safety WDT 被启用时，如果它没有在用户可编程的时间段内得到服务，它可能会引起 SMU 报警请求。  
      CPU 必须在此时间间隔内服务于 Safety WDT，以防止其发生。对 Safety WDT 超时的响应可以在 SMU 中进行配置。  
 

3 CPU 看门狗
---------

     单独的 CPU 监视器计时器提供了监视单独的 CPU 执行线程的能力，而不需要软件来协调一个共同的监视器的共享使用。  
     当 CPU WDT 被启用时，如果在用户可编程的时间段内没有正确地服务，它可能会导致 SMU 报警请求。CPU 必须在此时间间隔内服务其 CPU WDT，以防止这种情况发生。  
     对每个 CPUy 监视器超时的响应是可以在 SMU 内配置的。因此，定期维护 CPU WDT 可以确认相应的 CPU 正在按预期执行软件序列。

     重置后，CPU0 运行，CPU0 监视器计时器自动启动。其他 cpu 最初处于 HALT 状态，因此它们相应的[看门狗计时器](https://so.csdn.net/so/search?q=%E7%9C%8B%E9%97%A8%E7%8B%97%E8%AE%A1%E6%97%B6%E5%99%A8&spm=1001.2101.3001.7020)被禁用。其他 CPU 监视器没有配置为生成超时重置，但可以启用。CPU 监视器只能由其相应的 CPU 进行配置、启用或禁用。

4 寄存器写保护机制
----------

### ![](https://i-blog.csdnimg.cn/blog_migrate/fe74a1013d1c9a9177c2ecdc2a1a1bdb.png)

EndInit 的保护有三种：

1)“CE” 保护，只有把每一个 CPU 的 ENDINIT 设为 0 后，这个 CPU 的 critical registers 保护才被解除（可写）。

2)“E” 保护，任意一个 CPU 的 ENDINIT 设为 0 后，所有 CPU 的 system critical registers 保护就解除了。

3)“SE” 保护，Safety Watchdog 的 ENDINIT 设为 0 后，Safety EndInit 的保护就解除了。

ENDINIT 设置为 0 的操作，需要一套较为复杂的操作序列。

ENDINIT 置为 0 需要一段时间，只有等到 ENDINIT 真的为 0 后才能执行往下的操作，不然可能会产生异常。

  
5 看门狗模式
----------

每个看门狗计时器可以在三种不同的操作模式下操作：

 . Time-Out Mode

  .Normal Mode

  .Disable Mode

     ![](https://i-blog.csdnimg.cn/blog_migrate/8d5eb31e2674f86e40aafa9bd8adfff1.png)

        复位以后 CPU 的 Watchdog 默认是处在 Time-Out Mode 下的，WDT 在 Time-Out Mode 下就会从 0xFFFC 开始往上计数，如果计数到 0xFFFF 就会溢出，如果在计数到 0xFFFF 之前对 WDT_CON0 进行了 password access 后对 WDT_CON0 进行了 Modify access，重新对 WDT 进了 reload value 到 REL 值，这样 WDT 从 Time-Out 模式切换到了 Normal Mode，这个时候 WDT 开始从 REL 值往上计数。

     在 Noraml Mode 下计数到 0xFFFF 后 WDT 就溢出了，触发 SMU 的 Timeout 的 Alarm，这个 Alarm 会触发 SMU 里面的一个 Recovery time 进行计数，Recovery time 也 timeout 后就会产生一个 SMU 的 reset。因此如果开启了 watchdog 功能，必须在超时之前重新加载 REL 值。

6  看门狗使用
--------

     看门狗模块主要有三类寄存器，保护寄存器 WDTCPUyCON0（y=0-5），系统寄存器 WDTCPUyCON1 (y=0-5) ，状态寄存器 WDTCPUySR（y=0-5）。

### 6.1  解锁寄存器

    在对看门狗寄存器进行操作时，需要输入正确的 password 才能进行操作，默认 password 为 60.

又有 PW 位 bit2-bit8 进行的翻转，调试器读取出来 PW 实际为 3.

    当写入正确的 password 值后，LCK 会自动解锁，再次写入 password 值，LCK 会自动锁住。

![](https://i-blog.csdnimg.cn/blog_migrate/53c897d063a3ce22a55392f6a6d8ed20.png)

![](https://i-blog.csdnimg.cn/blog_migrate/ebf0e28ccda4506df65b1c1b7378a635.png)

### 6.2  加载超时计数值

      当写入正确的 password 后，可重新加载 REL 值，定时器会从加载值向上计数，加到 FFFF 值，后溢出。因此需要在计数值溢出之前， 在中断函数或周期任务里面重新加载 REL 值。

![](https://i-blog.csdnimg.cn/blog_migrate/4d155fd7e355bbecaf54b90b7826392f.png)![](https://i-blog.csdnimg.cn/blog_migrate/5ac48198c1853df701e423332189d1ad.png)

### 6.3  使能看门狗功能

     通过控制 系统寄存器 WDTCPUyCON1 中 DS 位，可以使能看门狗功能。![](https://i-blog.csdnimg.cn/blog_migrate/ae70698824c5e5a7c76f500446145649.png)![](https://i-blog.csdnimg.cn/blog_migrate/91b961b58856306ab58fb1953dbaa7cf.png)

### 6.4 观测看门狗状态

    通过状态寄存器 WDTCPUySR 的可以观测看门狗状态。

     如 DS 反映看门狗是否使能，OE 反映当前超时状态, TIM 位反映计数值。

![](https://i-blog.csdnimg.cn/blog_migrate/998a7a2784815cfa67240103b6b46563.png)![](https://i-blog.csdnimg.cn/blog_migrate/2aa2ae89b6e1662115d0a336cba5455f.png)

![](https://i-blog.csdnimg.cn/blog_migrate/726dc034ae922685844e78ffdff4aaab.png)

![](https://i-blog.csdnimg.cn/blog_migrate/7b60ac506937dc1df737fc53676bbb77.png)