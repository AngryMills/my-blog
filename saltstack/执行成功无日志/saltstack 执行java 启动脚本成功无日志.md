# saltstack部署java应用失败无日志——CICD 部署 

大家好，我是烤鸭：

​     最近在搞公司的CICD，遇到各种问题。复盘总结一下。



## CICD 架构

这篇文章写得很详细，可以看一下 https://linux.cn/article-9926-1.html

而这里只是结合现在的情况分析下：

CI 持续集成（Continuous Integration）

持续交付的目标是拥有一个可随时部署到生产环境的代码库。

CD 持续交付（Continuous Delivery）、持续部署（Continuous Deployment）

作为持续交付——自动将生产就绪型构建版本发布到代码存储库——的延伸，持续部署可以自动将应用发布到生产环境。


自动化的、高效的，并且可重复的

目前的CICD架构(封装jenkins和发布系统，简单够用就行)

![1](.\2.png)

可以看到目前的CICD部署生产和测试环境是分开的，虽然能完成上面定义的功能，但是有很大的问题。

发布测试环境可以进行自动化测试，但发布生产环境是从jenkins打包后发布的(并不是直接copy的测试时的包)。

这就有一个问题是生产环境的发包可能和测试的不一致(有时候 测试说测的时候没问题，开发说没问题，运维也是按流程来的，问题就在这)，持续交付的作用并没有发挥出来。



## 优化方式

目前的CICD 系统也只是封装了一层jenkins和发布系统，未来还会改版的。

待优化的部分：自动化测试、蓝绿部署、对多个版本的跟踪发布。

自动化测试 单元测试和持续测试(**集成测试**、**功能测试**、**验收测试**)，目前来看 单元测试的部分是缺失的，**很少写可执行的单元测试**，而且没有地方执行和查看。

**蓝绿部署指的是部分流量切换，预发环境和生产环境分开**。目前这一部分是空白的，需要测试介入。(以前做的时候用白名单拦截入口，但是白名单和指定和维护需要线下和线上同时维护，特别麻烦。销售联系到用户，要到手机号后，加入白名单。)

**对多个版本的跟踪发布指的是不同版本的协同开发，很难跟踪**，不过目前是没有类似场景的，之前倒是遇到过，一个大版本开发中，又提了对当前版本的更新维护，交付时间上交叉、代码合并出现问题，测试环境的代码维持不同的版本，对测试也是挑战。

单元测试 目前来看比较容易，强制编写单元测试，如果执行失败，则算项目失败，单元测试的标准应该以产品的需求为主，测试人员检验。打包的时候可以直接执行单元测试，只是缺少对需求的关联。（测试压力大了，规范养成了，就会容易很多）

蓝绿部署需要一套完整的预发环境(就是另一套生产环境，产生的订单数据再进行同步，对订单数据进行tag处理)。至于用户群体可以以指定的规则选择(比如根据用户画像等等，用户使用习惯)。搭建一整套环境是最复杂的，尤其是针对多个部门间的协同，每个部门的规则又不一样，缺少一个都不行。(难的不是技术，是沟通协作)

多版本的跟踪发布，这个其实遇到的场景比较少，像我上面遇到的其实是比较夸张了。如果真的遇到了，需要准备针对多套测试的多套测试环境，保证测试的分支代码没问题。其次要保证测试测过的代码和线上的完全一致，不要出现因为代码合并等问题。(如果正在开发 feature分支，master分支提一个hotfix合并，要保证提测的feature是包含这个hotfix的，上线可以不合并master，直接以feature上线，上线完成后再合并。)



## 目前的CICD框架

用的比较多的就是 jenkins和 hudson了。

- 项目页面：
  [https://jenkins.io/](https://link.zhihu.com/?target=https%3A//jenkins.io/)
- 源代码：
  [https://github.com/jenkinsci/jenkins](https://link.zhihu.com/?target=https%3A//github.com/jenkinsci/jenkins)

github用的是 Travis CI

- 项目页面：
  [https://docs.travis-ci.com/](https://link.zhihu.com/?target=https%3A//docs.travis-ci.com/)
- 源代码：
  [https://github.com/travis-ci/travis-ci](https://link.zhihu.com/?target=https%3A//github.com/travis-ci/travis-ci)

gitlab出了GitLab CI

- 项目页面：
  [https://about.gitlab.com/product/continuous-integration/](https://link.zhihu.com/?target=https%3A//about.gitlab.com/product/continuous-integration/)
- 源代码：
  [https://gitlab.com/gitlab-org/gitlab-ce/](https://link.zhihu.com/?target=https%3A//gitlab.com/gitlab-org/gitlab-ce/)

这部分已经侧重到运维研发了，现在很多企业产品也有完整的解决方案(如果有人力，参考一下，开发一款适合自己公司的也不错)。



## 遇到的问题

维护了几天，问题比想象中的多。

1. 比如saltstack部署java应用失败无日志，相同的代码启动成功脚本没问题，用saltstack启动不仅没日志，还没进程。最后发现，脚本中输出的日志路径是 nohup java -jar >nohup.out & ，输出的nohup.out 不在当前目录下，在/root 下面(注意脚本和jar包都需要写绝对路径，不要写相对路径)。找的日志就能发现问题了，其实之前遇到过，只是这次没日志，没及时发现。

   https://blog.csdn.net/Angry_Mills/article/details/112254459

2. 安装docker后，dns无法解析

   vi /etc/docker/daemon.json

   增加 bip属性， 指定容器使用的网络，避免由于dns解析后的ip和容器网络冲突导致的无法解析

   ```
   {
      "bip": "192.168.1.1/24"
   }
   ```

3. jenkins 发布docker 到harbor 仓库
   https://blog.csdn.net/Angry_Mills/article/details/114586569

