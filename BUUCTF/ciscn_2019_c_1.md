**新手上路，欢迎交流指错**

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

