**新手上路，欢迎交流指错**

先解压，得到一张绿蛇🐍的图片

在正式开始之前，先了解下我总结的相关知识(我也是边学边查的，欢迎交流指错）

> [!IMPORTANT]
> *MISC隐写术：图片中隐藏ZIP文件详解
>
> 1.文件结构基础知识
>
> <img width="258" height="388" alt="image" src="https://github.com/user-attachments/assets/f12aa385-04f7-4edd-a630-2c288123f4c0" />
>
> 关键点：文件查看器只看到文件尾，文件尾之后的数据会被忽略，且不会破坏文件，所以可以添加任意数据
>
> 2.文件头标识
>
>  <img width="544" height="363" alt="image" src="https://github.com/user-attachments/assets/52e25b19-02b7-40af-9ffd-1d5534c27617" />
>

搞懂原理之后开始实践，把图片放进 010Editor ，看看到了PK(50 4B 03 04)，就说明图片中隐藏了压缩包

<img width="553" height="355" alt="image" src="https://github.com/user-attachments/assets/e281bfff-b3d5-4723-b8fa-0fcfb3115ceb" />

然后用 binwalk 分析，根据返回结果可以得出 binwalk 扫描出了 ZIP 文件藏在第 278260 个字节的位置，再用电脑自带的 Power shell 把隐藏的压缩包截断出来

```
binwalk 你的文件名
```

直接在CMD里面运行：

```
powershell -Command "Get-Content snake.jpg -Encoding Byte | Select-Object -Skip 278260 | Set-Content hidden.zip -Encoding Byte"
```

> [!NOTE]
> 翻译：powershell -Command""            |告诉电脑要用 powershell 来执行" "里面的命令
>
> Get-Content snake.jpg -Encoding Byte             |把 snake.jpg 打开，不是当作图片，而是用数字的形式
>
> Select-Object -Skip 278260              |扔掉前 278260 字节，保留剩下的
>
> Set-Content hidden.zip -Encoding Byte            |把剩下的命名为 hidden.zip 保存

然后解压：
```
powershell Expand-Archive hidden.zip -DestinationPath extracted
```

> [!NOTE]
> 翻译：“把刚才生成的 hidden.zip 解压，放到一个叫 extracted 的文件夹里。

有两个文件，先看看key

<img width="808" height="207" alt="ec283371488520afb599c17ca75b906c" src="https://github.com/user-attachments/assets/8ed58cc1-336f-41fa-a964-ecd20c779127" />

用笔记本打开，发现需要对base64进行编码

有两种方式，一种是在本地 CMD 里运行
```
powershell -Command "[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String('你的Base64字符串'))"
```

<img width="843" height="62" alt="image" src="https://github.com/user-attachments/assets/2ae88d77-0059-48ba-bfdc-d3e4b1ee58e7" />

第二种方法是用在线工具 https://base64.us/ 

<img width="773" height="437" alt="image" src="https://github.com/user-attachments/assets/1eaa189d-34f4-470f-80ff-5539d9bd6d90" />

得到答案  What is Nicki Minaj's favorite song that refers to snakes?

<img width="1007" height="270" alt="image" src="https://github.com/user-attachments/assets/016e0363-132d-496c-b79c-d013b42bd82d" />

是 Anaconda

再用记事本打开另一个文件

<img width="437" height="96" alt="image" src="https://github.com/user-attachments/assets/3bd88188-8660-4103-94ea-71eba6de75d1" />

发现和想像的有点不一样啊，这是因为这是一个二进制文件，而记事本一般用于打开纯文本

所以在打开文件之前一般需要查看一下类型，这里我直接在 windows 的 cmd 里面看类型了

```
powershell -Command "Get-Content cipher -Encoding Byte -TotalCount 16 | ForEach-Object { '{0:X2}' -f $_ }"
```

<img width="1697" height="507" alt="image" src="https://github.com/user-attachments/assets/9d391e58-959a-4c1a-8b74-a6f954cdc526" />

发现好像不是任何常见类型

有点没有思路，查了查别人的做题步骤，才发现还需要用到一个在线工具 http://serpent.online-domain-tools.com/

<img width="1126" height="1014" alt="ae0fe6a8c16706856fb94f5e48e1d7ee" src="https://github.com/user-attachments/assets/787d27c2-83ad-4f17-9cc9-0e5148a937bd" />

key 就用之前解出的 anaconda

下载并打开得到的文件

<img width="528" height="193" alt="image" src="https://github.com/user-attachments/assets/2096709c-1678-4b79-8a68-3fea798e1d9a" />

得到flag

```
flag{who_knew_serpent_cipher_existed}
```
