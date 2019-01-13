ubuntu系统装机流程与相关软件安装 备忘
=====================================

前段时间说有机会把ubuntu装机的流程贴上来，让大家避免一些坑，今天电脑系统又崩了，就有了重装系统的机会，虽然浏览器的收藏夹导出了但没保存成功心都在流血，但还是要保持微笑。

下面就是流程啦。

装一个系统盘重启电脑什么的就不说啦，注意的是装的时候在最后一布把登陆需要密码给关掉，不然容易出现卡登陆界面的情况。

那咱们就先解决卡登陆界面的问题叭。首先，进设置

![IMG\_256](media/image1.png){width="10.208333333333334in"
height="5.760416666666667in"}

把自动登陆打开。然后为了一劳永逸不卡死在启动页面。然后

打开终端，去安装N卡的官方驱动。

 

sudo add-apt-repository ppa:graphics-drivers/ppa

sudo apt-get update

 

这是添加apt-get源，然后通过

ubuntu-drivers devices

寻找合适的驱动版本，如图

![IMG\_257](media/image2.png){width="6.5in"
height="1.7916666666666667in"}

后面带有recommended的就是合适的版本，然后

sudo apt-get install nvidia-driver-410

安装，这里要挑符合自己情况的版本哦。

然后通过

sudo reboot

sudo nvidia-smi

来重启和查看新nvidia版本

![IMG\_258](media/image3.png){width="7.520833333333333in"
height="4.09375in"}

![IMG\_259](media/image4.png){width="7.177083333333333in"
height="5.84375in"}

ok,第一阶段告成，到这个阶段往后只要不用最后一个nouveau驱动基本上就不担心电脑会卡logo卡死了。

 

相关软件安装：

1，视频播放器

![IMG\_260](media/image5.png){width="12.479166666666666in"
height="8.791666666666666in"}

![IMG\_261](media/image6.png){width="12.5in" height="8.8125in"}

 

安装smplayer，这个播放器比ubuntu自带的vlc好一点。

2，搜狗输入法

https://pinyin.sogou.com/linux/

根据电脑选择32位或者64位，

3，网易云音乐

参考网址：https://www.ubuntukylin.com/ukylin/forum.php?mod=viewthread&tid=32543

根据这个帖子下载。

链接: https://pan.baidu.com/s/1pLOGwLt 密码: g9rv
这里面有一个deb文件，下载后双击就能安装。

 

4，qq，百度云，tim，微信

参考网址：https://www.lulinux.com/archives/1319

在这个网址下通过教程安装。

 

5,chrome

参考网址：https://blog.csdn.net/qq551551/article/details/78885704/

终端中执行下述命令：

 

 

sudo wget http://www.linuxidc.com/files/repo/google-chrome.list -P
/etc/apt/sources.list.d/

 

wget -q -O - https://dl.google.com/linux/linux\_signing\_key.pub | sudo
apt-key add -

 

sudo apt-get update

 

sudo apt-get install google-chrome-stable

 

/usr/bin/google-chrome-stable

 

 

ok，这次教程到此为止啦，我以后系统又崩的时候会回来看的
