**新手上路，欢迎交流指错**

先checksec一下发现只开了NX栈不可执行

<img width="681" height="173" alt="image" src="https://github.com/user-attachments/assets/45cf0f06-85c0-44e2-aa59-76b33f13340a" />

file看看

<img width="801" height="43" alt="image" src="https://github.com/user-attachments/assets/1543382e-d8ef-420f-b476-94321d40690e" />

放 IDA 里面看看，有个 gets 函数

<img width="1071" height="357" alt="image" src="https://github.com/user-attachments/assets/9543f28c-8757-4172-8073-b604479daf8d" />

用 Shift + F12 发现没有什么可以用的字符串不出意料是 ret 2 libc

那就开始准备呗，等等! 写着写着感觉不太对又往 function name 里面搜了搜发现漏了 get_flag

<img width="812" height="515" alt="image" src="https://github.com/user-attachments/assets/c4d9186f-e90e-4c05-bfea-347a5260fcaa" />

