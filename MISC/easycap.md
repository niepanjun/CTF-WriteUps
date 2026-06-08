**新手上路，欢迎交流指错**

解压后发现是 .pcap 文件放 wrieshark 里面瞅瞅

> [!NOTE]
> 看到 .pcap .pcapng .cap 或者 .dmp 文件，直接用 Wireshark 打开,因为大概率是 抓包文件
>
> .pcap文件 打开是十六进制形式，里面有文件头和很多数据包
>
> .pcap文件 相当于一份记录程序运行的录像带，而 wrieshark 可以将录像带回放

<img width="2121" height="988" alt="屏幕截图 2026-06-07 214907" src="https://github.com/user-attachments/assets/7c795bd8-8132-4d4f-8a94-4d25c7e214ff" />

发现都是 TCP数据包 ，随便抓一个追踪

<img width="952" height="812" alt="屏幕截图 2026-06-07 214935" src="https://github.com/user-attachments/assets/80cabb54-834d-4326-b4bb-483e288b9f95" />

获得flag

<img width="506" height="176" alt="屏幕截图 2026-06-07 214850" src="https://github.com/user-attachments/assets/e1bba01e-c572-4286-8c96-aa019ba1f73d" />
