简易python程序 解决linux连接steam社区错误代码：-101
===================================================

2018年09月20日 14:03:02 小程序员之死 阅读数：620

编辑

 版权声明： https://blog.csdn.net/qq\_38900288/article/details/82774462

暑假的时候把整个电脑系统由win10换成了ubuntu18.04（回头再写个安装ubuntu18.04时应该避免的坑..）

由于某些众所周知的原因，steam社区在大陆无法正常连接。

![IMG\_256](media/image1.png){width="5.645833333333333in"
height="4.489583333333333in"}

在win10系统下，其常用解决办法是用*steam*community\_*302修复工具，开启加速器等。*

而在ubuntu系统下，*steam*community\_*302修复工具和加速器通常无法运行，怎么办。*

steam社区无法正常连接是由于DNS污染造成的，电脑不能将steam社区的域名与正确的IP地址对应起来。

*有一个简易的修复办法，修改hosts文件，手动告知电脑steam社区的IP地址。*

一般/etc/hosts 的内容一般有如下类似内容：\
\
127.0.0.1 localhost.localdomain localhost\
\
192.168.1.100 linmu100.com linmu100\
\
192.168.1.120 ftpserver ftp120

而我们要在其中加入的格式为   steam社区ip  *steamcommunity.com *

*steam社区的域名是steamcommunity.com。注意前面没有www.*

*现在问题就只有一个了，我们如何去获得steam社区的ip呢？*

 

*答案 去ping store.steampowered.com *

![IMG\_257](media/image2.png){width="7.677083333333333in"
height="2.7083333333333335in"}

我们看到*store.steampowered.com *的ip地址是23.13.185.114

这个就是*steamcommunity.com的ip *

*我们在hosts文件中加入*

![IMG\_258](media/image3.png){width="9.135416666666666in"
height="0.90625in"}

 

ok！ 我们再去连接steam社区，就可以连接成功啦！

当然这个方法只适合于DNS污染导致导致的无法联通。

 

用python编写代码简化ping 再 修改问价的代码如下

1.  

*\#coding=utf-8*

1.  2.  

import os

1.  2.  3.  4.  

str="ping -c 1 store.steampowered.com"

1.  2.  3.  4.  5.  6.  

a=os.popen(str).read()

1.  2.  3.  4.  

ips=a.find('(')+1

1.  2.  

ipf=a.find(')')-1

1.  2.  

ip=a\[ips:ipf\]

1.  2.  

print (ip)

1.  2.  3.  4.  

fi=open('/etc/hosts','r+')

1.  2.  

fi.write(ip+" steamcommunity.com\\n")

1.  

这个小脚本就能轻松完成上面所说的任务啦。

用pyinstaller生成的可执行文件在这里

https://download.csdn.net/download/qq\_38900288/10678661
