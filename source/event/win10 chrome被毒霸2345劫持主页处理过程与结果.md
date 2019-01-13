win10 chrome被毒霸2345劫持主页处理过程与结果
============================================

被纂改之后，经历很多方法都没改回来，后来看到一个博主讲了这个事情  http://blog.sina.com.cn/s/blog\_58d52ec30102wbzs.html

在chrome地址栏内输入：chrome://version

显示下面信息里面“命令行”后多出了个网址，。

这说明问题并不是出在chrome本身，而是出在explorer.exe上面，桌面的图标还有快速启动栏点击运行后是由explorer.exe调用指定路径程序来启动的，所以在启动chrome的时候，有某个进程加载到explorer.exe，并在运行chrome程序的时候附带上了这个被篡改的主页，因此不管怎么重装chrome都无用，问题不在chrome上面。

该进程在win7系统下位于“C:\\Users\\Administrator\\AppData\\Roaming”目录下，目前所知名字可能是kingoft，可能是pntelliTrace，共同点是都为dll文件。

后来我研究自己的win10系统，使用procexp64查看进程时发现有金山毒霸的dll文件参与启动，然后我把毒霸删了，然后正常了。。。。。。。
