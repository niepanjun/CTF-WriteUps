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

然后用 binwalk 分析，再用电脑自带的 Power shell 把隐藏的压缩包截断出来

有两个文件，先看看key

<img width="2560" height="1600" alt="image" src="https://github.com/user-attachments/assets/c8100f56-f26d-4d5c-a40e-6b1bd8e15794" />

用笔记本打开，发现需要对base64进行编码

有两种方式，一种是在本地 CMD 里运行
```
powershell -Command "[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String('你的Base64字符串'))"
```

<img width="843" height="62" alt="image" src="https://github.com/user-attachments/assets/2ae88d77-0059-48ba-bfdc-d3e4b1ee58e7" />

第二种方法是用在线工具 https://base64.us/ 

<img width="773" height="437" alt="image" src="https://github.com/user-attachments/assets/1eaa189d-34f4-470f-80ff-5539d9bd6d90" />

