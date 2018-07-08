---
title: 'linux环境下切换用户报java.lang.OutOfMemoryError: unable to create new native thread'
date: 2018-07-07 14:56:32
tags: linux java
---
### 起因
维护同事反映一个问题， 生产环境中有个应用原先使用root进行启动， 后面改成普通用户后， 运行一段时间不能正常对外接收请求。
<!--more-->
### 排查过程
查看后台日志， 报以下错误
```
java.lang.OutOfMemoryError: unable to create new native thread
```
而且在使用普通用户登录linux的过程中， 会报不能创建进程的错误， 导致不能登录， 于是使用不同用户查看了一下linux的`ulimit -a`，如下图：
![ulimit.png](http://pbjy9fbf1.bkt.clouddn.com/ulimit.png)
可以发现foura用户的max user processes只有1024, 说明不同用户的ulimit存在差异， 修改的办法为`vim /etc/security/limits.conf`, 加入如下行：
```
* soft noproc 50000
* hard noproc 50000
* soft nofile 65536
* hard nofile 65536
```
说明：* 代表针对所有用户，noproc 是代表最大进程数，nofile 是代表最大文件打开数
退出后重新登录机器， 再重启应用， 问题解决。
