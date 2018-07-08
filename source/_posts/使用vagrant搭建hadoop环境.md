---
title: 使用vagrant搭建hadoop环境
date: 2018-07-06 16:22:32
tags: hadoop
---
# 环境准备
1. 下载vagrant和virtualbox，并安装
- vagrant:https://www.vagrantup.com/
- virtualbox:https://www.virtualbox.org/
2. 虚拟机配置
- 1台master: memory 2048m 硬盘20G
- 2台slave: memory 1024m 硬盘10G
<!--more-->
# 安装步骤

从官网下载centos镜像：
    ```
    vagrant box add centos/7
    ```
如果box下载速度慢，可以拷贝控制台上的下载链接用迅雷等下载工具下载到本地 ![box下载.png](https://upload-images.jianshu.io/upload_images/12713048-383e7d30e5c37c95.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
创建vagrantfile所在文件夹，并创建两个文件Vagrantfile和init.sh
```
mkdir /project
touch VagrantFile
touch init.sh
```
其中VagrantFile是vagrant的启动配置文件， init.sh是初始环境的安装脚本
编辑VagrantFile文件， 内容如下
```
Vagrant.configure("2") do |config|
    config.vm.define :master1, primary: true do |master|
        master.vm.provider "virtualbox" do |v|
            v.customize ["modifyvm", :id, "--name", "hadoop-master1", "--memory", "512"]
       end
       master.vm.box = "centos/7"
       master.vm.hostname = "hadoop-master1"
       master.vm.network :private_network, ip: "192.168.10.10"
    end

   (1..2).each do |i|
    config.vm.define "slave#{i}" do |node|
        node.vm.box = "centos/7"
        node.vm.hostname = "hadoop-slave#{i}"
        node.vm.network :private_network, ip: "192.168.10.1#{i}"
        node.vm.provider "virtualbox" do |vb|
          vb.memory = "1024"
        end
     end
   end

  #manage hosts file 
  config.hostmanager.enabled = true
  config.hostmanager.manage_host = true
  config.hostmanager.manage_guest = true

   #provision
   config.vm.provision "shell", path: "init.sh", privileged: false
end
```
可以看到， 我们一共创建了4个虚拟机环境 ，分别是master1, slave1, slave2

启动虚拟机， 验证配置是否正确
```
vagrant up
```
 启动过程中如果有打印如下信息， 不需要理会
 ```
slave3: Warning: Connection aborted. Retrying...
slave3: Warning: Connection reset. Retrying...
slave3: Warning: Connection aborted. Retrying...
slave3: Warning: Connection reset. Retrying...
 ```
正常启动后，我们就可以使用以下命令登录到虚拟机了
```
vagrant ssh master1
```
我们可以运行以下命令， 测试host-manager
```
[vagrant@hadoop-master1 /]$ ping hadoop-slave1
PING hadoop-slave1 (192.168.10.11) 56(84) bytes of data.
64 bytes from hadoop-slave1 (192.168.10.11): icmp_seq=1 ttl=64 time=0.453 ms
64 bytes from hadoop-slave1 (192.168.10.11): icmp_seq=2 ttl=64 time=0.377 ms
64 bytes from hadoop-slave1 (192.168.10.11): icmp_seq=3 ttl=64 time=0.429 ms
64 bytes from hadoop-slave1 (192.168.10.11): icmp_seq=4 ttl=64 time=0.387 ms
```
可以看到， 我们并没有配置host文件，主机可以自动标识

**使用securecrt做为ssh客户端**
一般情况下， 我们习惯使用securecrt等工具进行主机的操作，但是如果你直接使用securecrt的时候，会报找不到public key的问题：
![securecrt问题.png](https://upload-images.jianshu.io/upload_images/12713048-63fcdcbadaadc8c9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个时候，我们可以做如下配置
>1.下载安装puttygen
https://the.earth.li/~sgtatham/putty/latest/w64/puttygen.exe
2.打开Putty Key Generator，点击"Load"按钮，然后选择主机的privatekey, 路径为.vagrant\machines\master1\virtualbox\private_key。
3.Load成功后，选择菜单中的"Conversions”—>"Export OpenSSH key"
4.然后会弹出保存文件对话框，选择一个你需要的名字，比如"openssh-key",保存到~/.ssh目录中去
注意：这一步保存的文件名不能有任何后缀，按照原文作者所述，如果用了比如openssh-key.pub的公钥文件，则SecureCRT会在同样目录下寻找名为"openssh-key"的私钥
5.在puttygen的界面，还需要选择save public key，保存publickey到~/.ssh目录中
6.这样， 我们同时生成了新的公钥和私钥

打开securecrt，登好主机IP和用户名后，需要做如下操作：
![securecrt设置.png](https://upload-images.jianshu.io/upload_images/12713048-c87a55f2b0cd2387.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这样，就可以使用securecrt进行登录了

编写provision文件
前面安装vagrant的时候说到，provision的作用是帮助我们进行主机环境的初始化工作，现在我们来编写init.sh，具体内容根据实际情况进行删减。在provision里，我只是安装了linux环境必需的一些组件，具体hadoop集群需要的组件我会使用ansible进行安装。
```
sudo yum install -y epel-release

sudo yum install -y  lrzsz.x86_64
sudo yum install -y nmap-ncat.x86_64
sudo yum install -y net-tools
sudo yum install -y vim-enhanced.x86_64
sudo yum install -y sshpass
```
编写完后，运行命令进行生效
```
vagrant provision
```

**设置ssh互信**
参考[编写ssh互信脚本](./编写ssh互信脚本)
在ssh登录过程中，可能会报Permission denied (publickey,gssapi-keyex,gssapi-with-mic)，解决方法如下：`vim /etc/ssh/sshd_config`, 修改如下配置为yes
```
PubkeyAuthentication yes
PasswordAuthentication yes
```
重启
```
systemctl restart sshd
```
**安装JDK**
```
yum install -y java-1.8.0-openjdk-devel
```
安装完Jdk后， 记得要到`hadoop-env.sh`中配置java_home
```
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.171-8.b10.el7_5.x86_64
```
**hadoop配置文件**
一共需要配置4个文件， core-site.xml hdfs-site.xml yarn-site.xml mappr-site.xml workers, 以上文件路径都位于etc/hadoop/中
- core-site.xml
```
<configuration>
<!-- 设置 resourcemanager 在哪个节点-->
<!-- Site specific YARN configuration properties -->
        <property>
                <name>yarn.resourcemanager.hostname</name>
                <value>hadoop-master1</value>
        </property>
        <property>
          <name>yarn.resourcemanager.webapp.address</name>
          <value>192.168.10.10:8088</value>
         </property>
         <!-- reducer取数据的方式是mapreduce_shuffle -->
        <property>
                <name>yarn.nodemanager.aux-services</name>
                <value>mapreduce_shuffle</value>
        </property>
</configuration>
```
- hdfs-site.xml
```
<configuration>
        <!-- 设置namenode的http通讯地址 -->
        <property>
                <name>dfs.namenode.secondary.http-address</name>
                <value>hadoop-master1:50090</value>
        </property>
        <!-- 设置hdfs副本数量 -->
        <property>
                <name>dfs.replication</name>
                <value>2</value>
        </property>
         <!-- 设置namenode存放的路径 -->
        <property>
                <name>dfs.namenode.name.dir</name>
                <value>file:/home/vagrant/hadoop-3.0.3/tmp/dfs/name</value>
        </property>
         <!-- 设置datanode存放的路径 -->
        <property>
                <name>dfs.datanode.data.dir</name>
                <value>file:/home/vagrant/hadoop-3.0.3/tmp/dfs/data</value>
        </property>
        <property>
          <name>dfs.namenode.datanode.registration.ip-hostname-check</name>
          <value>false</value>
        </property>
</configuration>
```
- yarn-site.xml
```
<configuration>
        <property>
                <name>yarn.resourcemanager.hostname</name>
                <value>hadoop-master1</value>
        </property>
        <property>
          <name>yarn.resourcemanager.webapp.address</name>
          <value>192.168.10.10:8088</value>
         </property>
         <!-- reducer取数据的方式是mapreduce_shuffle -->
        <property>
                <name>yarn.nodemanager.aux-services</name>
                <value>mapreduce_shuffle</value>
        </property>
</configuration>
```
- mapred-site.xml
```
<configuration>
        <!-- 通知框架MR使用YARN -->
        <property>
                <name>mapreduce.framework.name</name>
                <value>yarn</value>
        </property>
</configuration>
```
- workers
```
hadoop-slave1
hadoop-slave2
```
worker文件只需要在master节点中进行配置，workdrs中默认配置为localhost，此时为伪分布式配置
配置环境变量
```
export HADOOP_HOME=/home/vagrant/hadoop-3.0.3
```
同步hadoop文件夹到slave节点相同路径下
**启停集群**
启动运行`sbin/start-all.sh`, 停止运行`sbin/stop-all.sh`
访问namenode的web管理界面: `http://hadoop-master1:9870`
访问yarn的web管理界面: `http://192.168.10.10:8088`

