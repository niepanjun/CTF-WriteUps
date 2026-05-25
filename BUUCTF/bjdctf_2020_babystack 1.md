**新手上路，欢迎交流指错**

先checksec一下

<img width="656" height="177" alt="屏幕截图 2026-05-25 211407" src="https://github.com/user-attachments/assets/0291245c-29f3-4712-8cc6-c034d2b7599f" />

发现开了NX栈堆不可执行，并且是64位程序

再IDA分析，发现逻辑其实很简单，就是程序问了名字的长度，并把长度当作read的长度参数

<img width="765" height="413" alt="屏幕截图 2026-05-25 211724" src="https://github.com/user-attachments/assets/9261d26c-7287-4f25-b94f-d55c71161a48" />

根据IDA分析的 _BYTE buf[12]; // [rsp+0h] [rbp-10h] 我们可以知道缓冲区大小是 0x10(16B) ,那么我们只需要保证输入数字大于 24(16+8) 就行了

再看看左侧，发现惊喜后门函数 backdoor，即刻点击进入

<img width="930" height="524" alt="屏幕截图 2026-05-25 212501" src="https://github.com/user-attachments/assets/71c1ebf2-ace5-4043-8e07-b126c2ff8bf2" />

果真不错

