# 关于容器docker #

## 1.为什么会出现 ##
&nbsp;&nbsp;&nbsp;&nbsp;软件开发部署需要重复的在不同的操作系统或者不同的环境下搭建同样的环境，环境搭建需要添加各种依赖，可能还需要配置各种环境变量。希望能 安装的时候，把原始环境一模一样地复制过来。<br>

**1、虚拟机**<br>
虚拟机（virtual machine）就是带环境安装的一种解决方案。它可以在一种操作系统里面运行另一种操作系统，虽然可以通过虚拟机还原软件原始环境，但是这个方案有一下几个缺点：<br>

- 资源占用多：虚拟机会独占一部分内存和硬盘空间。它运行的时候，其他程序就不能使用这些资源了。哪怕虚拟机里面的应用程序，真正使用的内存只有1MB，虚拟机依然需要几百MB的内存才能运行。
- 冗余步骤多：虚拟机是完整的操作系统，一些系统级别的操作步骤，往往无法跳过，比如用户登录。
- 启动慢:启动操作系统需要多久，启动虚拟机就需要多久。可能要等几分钟，应用程序才能真正运行。

**2、Linux容器**<br>
https://baijiahao.baidu.com/s?id=1694108417119974229&wfr=spider&for=pc