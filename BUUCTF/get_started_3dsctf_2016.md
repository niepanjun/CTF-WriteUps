**新手上路，欢迎交流指错**

先checksec一下发现只开了NX栈不可执行

<img width="681" height="173" alt="image" src="https://github.com/user-attachments/assets/45cf0f06-85c0-44e2-aa59-76b33f13340a" />

放 IDA 里面看看

<img width="1071" height="357" alt="image" src="https://github.com/user-attachments/assets/9543f28c-8757-4172-8073-b604479daf8d" />

用 Shift + F12 发现没有什么可以用的字符串不出意料是
