# 关于idea maven 编译报错的问题。Perhaps you are running on a JRE rather than a JDK

1. 先检查是否环境变量的问题

参考这篇 https://blog.csdn.net/mingjie1212/article/details/106963143

2.  如果确定环境变量没问题
  执行maven install的时候报错，Perhaps you are running on a JRE rather than a JDK
  由于我是在控制台执行的，所以不考虑配置的问题。

3.   检查环境变量 jdk 和maven

   ![2](E:\my\blog\maven\找不到jdk\2.png)

   ![1](E:\my\blog\maven\找不到jdk\1.png)

4.   问题猜想

   由于之前环境变量是jre11，发现有问题后换了jdk 8。还是不行，感觉是这个问题。idea 控制台查看maven 版本。果然是这个问题，可以看到idea的环境变量还是之前的。

   ![4](E:\my\blog\maven\找不到jdk\4.png)

   

5.   问题解决

   重启和 *Invalidate* *Caches* / *Restart* 都试过了，没办法。这个缓存比较强劲，只能去C盘下手动删除了。

   删除目录：

   C:\Users\xxxxxx\AppData\Local\JetBrains\IntelliJIdea2020.1\caches

