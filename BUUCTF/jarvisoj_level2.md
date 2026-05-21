##[BUUCTF]jarvisoj_level2_writeup

哈哈哈哈哈哈，我终于弄懂怎么插入图片啦



首先checksec一下

<img width="257" height="90" alt="image" src="https://github.com/user-attachments/assets/712cd4d7-6ec8-4100-a49d-11c9d5d47d47" />

发现只开了NX（栈不可执行）那就应该考虑怎么构造攻击链

下一步打开IDA，看看main

<img width="635" height="317" alt="image" src="https://github.com/user-attachments/assets/a6c27015-6eeb-49fa-b29f-7ead8dbceaea" />


发现没什么特别的，那就先看看vulnerable_function()里面有没有什么漏洞


<img width="280" height="115" alt="image" src="https://github.com/user-attachments/assets/ddd98090-cb2c-4fdb-ac8f-c29b1c9068e2" />

发现了一个危险的read，读取了0x100（256B），但缓冲区大小是0x88（136），存在缓冲区溢出

然后再用shift加F12看看有没有什么可用的字符串

<img width="253" height="163" alt="image" src="https://github.com/user-attachments/assets/267e3ba0-6781-4354-ad68-4c5f004b20c4" />

可以，收获颇丰，这样我们可以获得/bin/sh的地址
