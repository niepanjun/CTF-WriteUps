64 NX D

目标是控制执行流

已知64位———有6个寄存器         动态链接———在主程序ELF中没有syscall          有NX———栈堆无法执行

因为没有发现什么可以利用的函数———所以自己拼一个


> [!NOTE]
> 为什么动态“必须泄漏libc”、静态“不用”：
> 
>
> 动态链接时 libc 是外挂文件，运行时的加载基址被 ASLR 随机化，你不知道 system//bin/sh 在哪；
>
> 静态链接时 libc 已经是二进制本体的一部分，且静态题几乎一定 No-PIE，所有地址你拿文件一查就知道。

> [!WARNING]
>
> 泄露的真实逻辑
> 
> 主 ELF 没有 puts 的代码，它只有 puts@plt（一段跳板）和 puts@got（一个 8 字节指针槽）。
> 
> 真正的 puts 机器码在 libc.so.6 里。
> 
> 第一次调用 puts 之后，puts@got 会被填成 libc 里 puts 的真实运行时地址——我们"泄漏"读的就是这个 puts@got
> 

> [!NOTE]
>puts计算基地址原理
> 
>其实只有一个运行时 puts 地址 + 一个文件偏移常量：
> 
> puts_addr：你泄漏出来的，运行时地址（被 ASLR 随机化，比如 0x7f1234809c0）。
> 
> puts_off：puts 在 libc 文件里的固定偏移（常量，比如 0x809c0），来自 LibcSearcher 或题目附带的 libc。
> 
> libc_base = puts_addr − puts_off。
