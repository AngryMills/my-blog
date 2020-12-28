日志 中文乱码、nacos 中文乱码、saltstack 中文乱码、docker中文乱码

大家好，我是烤鸭：

​     今天分享一个 saltstack 中文乱码 的问题。



1.  问题说明

   由于项目之前没有接入公司的发布系统，今天接入之后发现日志乱码，不仅如此，从nacos获取到的中文参数也是乱码。于是猜想是发布系统遗留了一个bug。如下图。

   ![1](.\1.png)

   发布系统使用的是 saltstack，网上查了下，saltstack 导致的中文乱码，下文说的是2015.8 之后的版本修复了，而我们这的版本是2019.2.5。

   https://blog.csdn.net/hnhuangyiyang/article/details/50421738

   即便如此我们还是去看一下现在的源码。找到salt依赖的 cmd脚本。

   ```
   more /usr/lib/python2.7/site-packages/salt/modules/cmdmod.py
   ```

   ![1](.\2.png)

   关于上边参数的说明，可以看一下 https://www.iteye.com/blog/yintech-397380，这里重点看一下 LANGUAGE。

   **locale的设定：**
   LC_ALL和LANG优先级的关系：LC_ALL > LC_* >LANG
   1、如果需要一个纯中文的系统的话，设定LC_ALL= zh_CN.XXXX，或者LANG=zh_CN.XXXX都可以。
   2、如果只想要一个可以输入中文的环境，而保持菜单、标题，系统信息等等为英文界面，那么只需要设定 LC_CTYPE＝zh_CN.XXXX，LANG=en_US.XXXX就可以了。
   3、假如什么也不做的话，也就是LC_ALL，LANG和LC_*均不指定特定值的话，系统将采用POSIX作为lcoale，也就是C locale。

   **LANG和LANGUAGE的区别：**
   LANG - Specifies the default locale for all unset locale variables
   LANGUAGE - Most programs use this for the language of its interface
   LANGUAGE是设置应用程序的界面语言。而LANG是优先级很低的一个变量，它指定所有与locale有关的变量的默认值。

   可以看到启动应用程序没有设置 lc_all  ,会按照 language处理，而这里language设置的C。

   C 是啥编码呢。看下这篇http://www.gnu.org/software/libc/manual/html_node/Standard-Locales.html#Standard-Locales，也就是 ISO 编码。

   编码乱了，为什么会导致nacos和日志乱码，new String的时候不指定编码格式，会按照当前环境变量的编码格式创建，所以乱码了。

   其实docker也是一样的，可以参考解决方案的第一个。

   

2. 解决方案

   - 修改应用程序编码，比如我们是java应用，在启动脚本中增加 -Dfile.encoding=utf-8

   - 修改 saltstack ，因为salt实际调用用的是SLS，需要把env参数找位置加进去，如下图

     ![1](.\3.png)

   - 当然也可以修改 上边说到的 cmdmod.py



3. 总结：

   问题出现还是有一定影响的，总结下发布系统使用挺久的了，出现这个问题没人反馈过(没人想到是发布系统的问题)，要不自己改脚本，要不想别的办法处理(new String的时候，指定编码格式)。

   这种问题最好还是由上游处理，解决方案的1、3都不太友好。

   还有就是测试和生产环境并不一致，发布系统只在生产环境使用(如果发布系统出bug了，很难查，而且测试验收通过了?)

   坑无处不在，希望天下无坑。

