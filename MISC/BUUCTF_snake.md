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

