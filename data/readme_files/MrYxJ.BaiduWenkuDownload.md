# 格格百度文库下载程序1.0 及2.0                                                                                                            						                                                                 														

## 前言

总体言之，此软件功能相较于已有的冰点文库，最大创新点可能在于我先实现了**多线程的批量下载**。 大部分功能也来自于我之前爬虫实习积累的经验。

最后考虑更好用户交互性，我开发2.0带GUI界面的版本。这个过程同时解决了很多未知的技术和微小bug，所以还是耗费我的一段**时间心血和劳动力**。

基于**崇尚互联网自由和分享**的精神，源代码和软件都全部放出。但希望后来使用和学习者能**注明出处，尊重原创！。**

2018年，对于很多人是不平凡的一年，在快到来的12月里，希望你一切顺利~

最后如果觉得对您有帮助，欢迎**github Start** 加**Fork**。

还有更多使用细节和问题反馈可以github issues 

## 百度文库下载器1.0说明 

### 程序运行注意

- 运行该程序，必须保证电脑里面有Firefox浏览器。

- 同时在运行该程序时，尽量不要打开和操作火狐浏览器（Firefox）。



### 实现的意义

支持对百度文库能浏览的学习，资料，文章（pdf,word,txt,ppt）便利（免登陆，消费劵)和稳定的（支持复杂图表word文档）的下载到本地,并应对大量的文章支持多进程高效的批量下载。

**声明：本程序原理仅是模拟浏览器截图到本地，开放的初衷是方便使用者下载文档资料到本地离线学习使用，无盈利目的，更请勿利用此程序下载的文档牟取盈利。同时百度文库上文档也都是百度账号的用户自愿上传，不存在侵权行为。**



### 实现的主要功能
- 将多张图片合成pdf文件。

- 文件下载到本地的路径可自由设置。

- 对下载PDF文件,可设置其分辨率的大小即文件大小。

- word,pdf文件，下载为超高清的pdf文件；txt文件，还原下载为txt文件。ppt文件，下载为高清图集。

- 对多个文件实现多进程的批量下载，可设置多进程同时下载的进程数（默认为4），极大加快大量文件下载速率和和简化了操作。

- 加入sqlite3数据库，对下载的历史记录进行保存，删除，方便回顾。


### 与他文库下载器和爬虫程序的对比

##### 优点：
- 相比Github上目前其他爬文库程序，它们简单只对txt,word等文档只有文字内容的解析，遇到带有格式或者表格的文档无法还原，几乎没有做什么处理。而此程序完美的还原word,pdf,ppt,txt,文档。

- 目前网络上类似文库爬取文档下载的软件口碑最好是冰点文库，相比冰点文库，实现了和它一样的技术同时也先实现多进程的批量下载，完善高效的下载功能。
- 相比冰点文库的广告，此程序原创，无广告，绿色，安全。

##### 缺点:
- 冰点文库已经有很长迭代式开发周期，功能更完善，运行更稳定，几乎支持所有的主流文库，相比之下此程序支持网站目前只有百度文库。

- 冰点文库使用C语言进行开发，程序运行效率好于此程序。


### 目前有待改进地方
- 设计一个简洁（陋）的UI界面，使交互更友好。（2.0版本写了一个窗口，提升交互性，易用性）

- 支持对其他开放的文库文档实现下载功能，比如知网，道客，360cc（待补充）。

- 多测试一下程序八阿哥（BUG），使其更稳定。

  ​    

         1.0主界面 

  ![](https://github.com/MrYxJ/GeGeWenkuDownload/raw/master/%E6%A0%BC%E6%A0%BC%E7%99%BE%E5%BA%A6%E6%96%87%E5%BA%93%E4%B8%8B%E8%BD%BD%E5%99%A81.0/%E6%BA%90%E4%BB%A3%E7%A0%81%E7%AD%89/picture/Main.png)

### 主界面部分功能的说明：

【2】.批量下载百度文库文档功能：

       使用此功能前，需要编辑与软件同一目录的 ```urls.txt```文件,没有则新建该文件，在里面写入需要批量下载网页链接，一行一个网页链接。

【3】.设置下载PDF文件分辨率：

       不同分辨率的参数，决定下载下pdf文件的分辨率参数与大小，分辨率越高，文件越清晰，文件容量越大。

【7】.设置批量下载时多进程的数量

        理论上，进程设置数最多为使用者电脑CPU的核数，保证同时有最多不同进程在同时下载不同文件，但考虑程序性能内存占用，推荐设置为4,表示批量下载时最多有四个不同文件同时下载。

---



## 格格百度文库下载程序2.0说明

在1.0黑白Dos命令行运行界面基础上，利用PYQT技术撸了更为美观便于使用的界面。但由于没能解决PYQT里线程池的问题，2.0版本的批量下载是单线程按照队列顺序一个一个下载，批量下载速度没有1.0高效，其余功能都和1.0一样。具体请下载体验吧。

                                         格格百度文库下载器2.0   程序主界面

![程序主界面](https://raw.githubusercontent.com/MrYxJ/GeGeWenkuDownload/master/%E6%A0%BC%E6%A0%BC%E7%99%BE%E5%BA%A6%E6%96%87%E5%BA%93%E4%B8%8B%E8%BD%BD%E5%99%A82.0/%E6%BA%90%E4%BB%A3%E7%A0%81%E7%AD%89/picture/main.png)

                                         格格百度文库下载器2.0  批量下载界面
                                         
![批量下载](https://github.com/MrYxJ/GeGeWenkuDownload/blob/master/%E6%A0%BC%E6%A0%BC%E7%99%BE%E5%BA%A6%E6%96%87%E5%BA%93%E4%B8%8B%E8%BD%BD%E5%99%A82.0/%E6%BA%90%E4%BB%A3%E7%A0%81%E7%AD%89/picture/download.png)

​                                                                  

​            



​                    