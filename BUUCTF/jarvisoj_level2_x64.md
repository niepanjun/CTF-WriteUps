**新手上路，欢迎交流指错**

依旧先checksec

<img width="367" height="153" alt="image" src="https://github.com/user-attachments/assets/a7ba4c56-61f7-42d1-94c4-b0b46901970d" />

发现只开了 NX 栈不可执行，64位程序

放 IDA 里面看看，有点眼熟啊，是之前做的 jarvisoj_level2 的64位形态

<img width="524" height="147" alt="image" src="https://github.com/user-attachments/assets/99b1b709-60dc-4994-83bd-ed1c759bd42f" />

read依旧开的够大，可以覆盖缓冲区，造成栈溢出，那么估计也有 system 和 /bin/sh，那就 shift+F12 看看

<img width="1048" height="426" alt="image" src="https://github.com/user-attachments/assets/ee536da2-1415-4360-92d4-ab8ea02ee0d2" />

<img width="1110" height="885" alt="image" src="https://github.com/user-attachments/assets/2a3af57d-78d9-4e9d-b841-5654d3b334b3" />

<img width="712" height="123" alt="image" src="https://github.com/user-attachments/assets/e0e49ca3-1738-4a72-a926-878736ecd5fd" />

那么system与/bin/sh地址都找到了那就再梳理一下思路：

64位程序的函数调用与32位函数调用有很大区别，32位在调用的时候会直接把参数压入栈，而64位会把前6位参数分别放进6个寄存器里面（rdi,rsi,rdx,rcx,r8,r9），而多余的参数就会像32位一样压入栈

那我们就可以先找到并放入 pop rdi,ret 然后是/bin/sh，最后是system

因为pop edi 先把栈顶元素出栈压入rdi寄存器，然后返回到system的地址，这样就是一条完整逻辑链用来获得shell

<img width="521" height="873" alt="image" src="https://github.com/user-attachments/assets/ff996813-1669-4a96-aa31-9b10bda5fc52" />

大概长这样

接下来就是获取 pop rdi,ret 指令地址然后构造脚本了

```python
ROPgadget --binary ./level2_x64 --only "pop|ret"
```

<img width="838" height="370" alt="屏幕截图 2026-05-26 215941" src="https://github.com/user-attachments/assets/4b5f750f-854b-4f62-a8bf-2c2f3535b15e" />

看来我要找的地址就决定是你了! 0x4006b3

构造脚本
```python
