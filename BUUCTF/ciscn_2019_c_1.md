**新手上路，欢迎交流指错**

**参考与致谢**  
> 本题解题思路参考了以下资源，感谢原作者的分享：
> https://blog.csdn.net/2301_76794844/article/details/148954691 作者: @谁把我灯关了

先checksec

<img width="361" height="148" alt="image" src="https://github.com/user-attachments/assets/576c02f5-0140-4e37-b62b-2552ffb0aa48" />

开了NX栈不可执行，并且是64位程序

看看IDA main

<img width="915" height="793" alt="image" src="https://github.com/user-attachments/assets/f2dbe6ab-cedb-4fec-8ef6-1b465a20cd6f" />

观察伪代码运行逻辑，发现 v4==1 就可以进入 encrypt() ，v4==2 是重新选择，v4==1是exit退出

<img width="902" height="180" alt="image" src="https://github.com/user-attachments/assets/dfdd6418-aa89-4c38-9848-e5f28266f67d" />

那么点进 encrypt() 里面看看

<img width="561" height="728" alt="image" src="https://github.com/user-attachments/assets/948f2935-a965-49d4-9c5d-f54f132a1cd2" />

发现及其危险的 gets() 函数，然后就想 F12+shift 看看有没有什么 system,/bin/sh 之类的让我能利用利用

<img width="1170" height="669" alt="image" src="https://github.com/user-attachments/assets/dbc8bd6f-fa69-4637-a2e2-500f049cb005" />

发现没有

那么进行详细分析，接下来应该怎样做
 
>  [!IMPORTANT]
> **根据栈溢出，我们能想到的做法有：**
> 
> ret 2 text
> 
> ret 2 shellcode
> 
> ret 2 syscall
> 
> ret 2 libc
>
> **接下来进行可行性分析：**
>
> **ret 2 text**: 在text段有现有的函数或代码片段可以被我们利用，比如说现成的system(/bin/sh)后门函数，或gadget（pop rid,ret)等 
> 缺点是：依赖现有程序，可能找不到，优点是：不需要泄露地址，比较简单
>
> **ret 2 shellcode**: 在栈或堆上执行构造过的机器码（shellcode），但在现在的环境中极少出现，实用性不高
>
> **ret 2 syscall**: 
>
> **ret 2 libc**: 跳转到动态链接库(libc.so)中的函数(system,execve等)
>
> 原理：大多数题都动态链接
>
> 难点：ASLR(地址空间布局随机化)会导致libc基址每次运行都会改变
