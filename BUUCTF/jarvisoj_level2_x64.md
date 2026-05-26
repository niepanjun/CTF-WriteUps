**新手上路，欢迎交流指错**

依旧先checksec

<img width="367" height="153" alt="image" src="https://github.com/user-attachments/assets/a7ba4c56-61f7-42d1-94c4-b0b46901970d" />

发现只开了 NX 栈不可执行，64位程序

放 IDA 里面看看，有点眼熟啊，是之前做的 jarvisoj_level2 的64位形态

<img width="524" height="147" alt="image" src="https://github.com/user-attachments/assets/99b1b709-60dc-4994-83bd-ed1c759bd42f" />

read依旧开的够大，可以覆盖缓冲区，造成栈溢出，那么估计也有 system 和 /bin/sh，那就 shift+F12 看看

<img width="1048" height="426" alt="image" src="https://github.com/user-attachments/assets/ee536da2-1415-4360-92d4-ab8ea02ee0d2" />

<img width="712" height="123" alt="image" src="https://github.com/user-attachments/assets/e0e49ca3-1738-4a72-a926-878736ecd5fd" />

那么system与/bin/sh地址都找到了
