**新手上路，欢迎交流指错**

这道题主要练习一下我的绕过能力

先在题目给的网址后面加上 /?ip=127.0.0.1 并按回车

<img width="667" height="332" alt="屏幕截图 2026-06-04 212630" src="https://github.com/user-attachments/assets/b2ba4879-92ba-4aad-8d16-bd22e81921d2" />

这一步可以知道网页后端可以接收参数，执行命令，返回结果

<img width="670" height="412" alt="image" src="https://github.com/user-attachments/assets/cee75cf5-3fde-4ef6-85c6-313c53044aad" />

再 ls 一下，发现有  flag.php  index.php  这两个文件，那么flag大概率就在flag.php里面了

```
/?ip=127.0.0.1|ls
```

<img width="824" height="160" alt="image" src="https://github.com/user-attachments/assets/d8462f3a-e814-4e75-9b77-4f06cf6f5dc2" />

```
/?ip=127.0.0.1|cat flag.php
```

诶，做个题怎么还骂人呢

那就用 $IFS 或更精确的 $IFS$9 试试水（是Linux的变量，因为大多数网站底层运行的服务器都是Linux系统）

```
/?ip=127.0.0.1|cat$IFS$9flag.php
```

<img width="445" height="61" alt="image" src="https://github.com/user-attachments/assets/b65925c8-e60e-4fb9-81bf-970a70792f7f" />

啊哦

那我们就看看另一个文件

<img width="491" height="279" alt="image" src="https://github.com/user-attachments/assets/ba369739-0136-4c6d-a05b-095a7bd46cd2" />

好家伙，过滤了这么多

再试试另一种反绕方法，利用 

拿到flag

<img width="550" height="317" alt="image" src="https://github.com/user-attachments/assets/56f5aa14-a372-42a7-a7dd-8027e886bf14" />
