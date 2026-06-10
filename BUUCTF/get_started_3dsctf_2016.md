**新手上路，欢迎交流指错**

先checksec一下发现只开了NX栈不可执行

<img width="681" height="173" alt="image" src="https://github.com/user-attachments/assets/45cf0f06-85c0-44e2-aa59-76b33f13340a" />

file看看(之前就是忘了file出了大问题)

<img width="801" height="43" alt="image" src="https://github.com/user-attachments/assets/1543382e-d8ef-420f-b476-94321d40690e" />

> [!NOTE]
><img width="959" height="697" alt="image" src="https://github.com/user-attachments/assets/1d2327b9-3d82-4c29-ad54-e5c268d14a9d" />
>
> 因为没查是静态链接，所以用了 elf 导致错误


放 IDA 里面看看，有个 gets 函数可以利用

<img width="1071" height="357" alt="image" src="https://github.com/user-attachments/assets/9543f28c-8757-4172-8073-b604479daf8d" />

用 Shift + F12 发现没有什么可以用的字符串

那就开始准备呗，等等! 写着写着感觉不太对又往 function name 里面搜了搜发现漏了 get_flag

<img width="812" height="515" alt="image" src="https://github.com/user-attachments/assets/c4d9186f-e90e-4c05-bfea-347a5260fcaa" />

核心逻辑就是让 a1 == 814536271 && a2 == 425138641

get_flag 地址是0x080489A0

<img width="632" height="161" alt="image" src="https://github.com/user-attachments/assets/92110c84-7144-4320-bd75-0ffb9a911dc8" />

再把 a1 a2 的值换为十六进制

a1=0x308CD64F
a2=0x195719D1

基本信息差不多了，但还差一个返回地址，这地方也是个重点

> [!IMPORTANT]
> 在C语言中，putchar(printf)并不是直接把字符打印到屏幕上，而是放在缓冲区里面，只有遇到\n，程序“正常结束”或缓冲区满了才会打印在屏幕上
>
> <img width="1206" height="1108" alt="屏幕截图 2026-06-10 202953" src="https://github.com/user-attachments/assets/2af8d49e-720e-44c5-a4fb-4a2217645d11" />
>
> <img width="1197" height="717" alt="屏幕截图 2026-06-10 203018" src="https://github.com/user-attachments/assets/d307c36c-71ea-4e67-9c2c-0f3c4b811e45" />




char 变量名[大小]; // [相对于esp的位置] [相对于ebp的位置或栈帧边界]
