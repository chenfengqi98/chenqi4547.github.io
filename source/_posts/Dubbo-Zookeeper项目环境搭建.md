---
title: Dubbo+Zookeeper项目环境搭建
date: 2019-08-18 08:53:45
tags:
- Dubbo
- Zookeeper
categories:
- Dubbo
- Zookeeper
---
# Dubbo+Zookeeper环境搭建

## JDK的安装

> 下载

在官网下载完JDK后，在`usr`目录下新建一个`java`目录，把JDK解压到这个目录。

百度云下载：[百度云链接](https://pan.baidu.com/s/1Qk7puB904aQugozWuRrJJQ )，提取码：9wej

> 配置环境变量

编辑`/etc/profile`

```bash
sudo vim /etc/profile
```

在文件末尾输入

```bash
JAVA_HOME=usr/java/jdk_version
JRE_HOME=usr/java/jdk_version/jre
PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin
CLASSPATH=:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
export JAVA_HOME JRE_HOME PATH CLASSPATH
```

然后输入`source /etc/profile` 加载一下配置

```bash
[root@localhost ~]# source /etc/profile
[root@localhost ~]# java -version
java version "1.8.0_141"
Java(TM) SE Runtime Environment (build 1.8.0_141-b15)
Java HotSpot(TM) 64-Bit Server VM (build 25.141-b15, mixed mode)
[root@localhost ~]# 
```

每次换一个`terminal`可能环境变量不生效，需要重新`source /etc/profile`,可以配置一下。进入`root`根目录，然后修改一下`.bashrc`文件，在文件末尾加入`source /etc/profile`。

```bash
[root@localhost etc]# cd ~
[root@localhost ~]# ls -a
.                .bash_logout   .config     .local               .ssh
..               .bash_profile  .cshrc      .npm                 .tcshrc
anaconda-ks.cfg  .bashrc        .gitconfig  .npminstall_tarball  .viminfo
.bash_history    .cache         .lesshst    .oracle_jre_usage    .wget-hsts
[root@localhost ~]# vim .bashrc
```

## Tomcat的安装

把tomcat下载到`/opt`目录下并解压。

进入到tomcat的`bin`目录下运行命令

```bash
[root@localhost opt]# cd apache-tomcat-9.0.22/
[root@localhost apache-tomcat-9.0.22]# cd bin
[root@localhost bin]# ./startup.sh 
Using CATALINA_BASE:   /opt/apache-tomcat-9.0.22
Using CATALINA_HOME:   /opt/apache-tomcat-9.0.22
Using CATALINA_TMPDIR: /opt/apache-tomcat-9.0.22/temp
Using JRE_HOME:        /usr/java/jdk1.8.0_141/jre
Using CLASSPATH:       /opt/apache-tomcat-9.0.22/bin/bootstrap.jar:/opt/apache-tomcat-9.0.22/bin/tomcat-juli.jar
Tomcat started.
[root@localhost bin]# 
```

然后到浏览器查看是否运行成功。

运行`./startup.sh`,停止`./shutdown.sh`

## Dubbo下载与运行

把`dubbo-admin-2.6.0.war`下载到`/opt`目录下并解压，重命名为`dubbo`

```bash
[root@localhost opt]# unzip dubbo-admin-2.6.0.war -d dubbo
```

然后配置`tomcat` 运行`dubbo`项目.

```bash
[root@localhost opt]# cd apache-tomcat-9.0.22/
[root@localhost apache-tomcat-9.0.22]# cd conf
[root@localhost conf]# ls
Catalina             context.xml           logging.properties  tomcat-users.xsd
catalina.policy      jaspic-providers.xml  server.xml          web.xml
catalina.properties  jaspic-providers.xsd  tomcat-users.xml
[root@localhost conf]# vim server.xml
```

找到下面代码段

```xml
      <Host name="localhost"  appBase="webapps"
            unpackWARs="true" autoDeploy="true">

        <!-- SingleSignOn valve, share authentication between web applications
             Documentation at: /docs/config/valve.html -->
        <!--
        <Valve className="org.apache.catalina.authenticator.SingleSignOn" />
        -->

        <!-- Access log processes all example.
             Documentation at: /docs/config/valve.html
             Note: The pattern used is equivalent to using pattern="common" -->
        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="localhost_access_log" suffix=".txt"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />
        <Context path="/dubbo" docBase="/opt/dubbo" debug="" privileged="true"/>
      </Host>

```

配置一下Context，`path`配置项目访问路径，`docBase`配置项目文件位置。

启动tomcat,访问`localhost:8080/dubbo`，出现`dubbo`界面就可以了。

## zookeeper下载与安装

下载zookeeper到`/opt`并解压。

进入zookeeper目录，新建一个data文件，然后进入到conf文件，发现有一个zoo_sample.cfg，copy这个文件到当前目录重命名为zoo.cfg,然后配置一下。

把dataDir=/tmp/zookeeper改成刚才得data目录dataDir=/opt/zookeeper-3.4.11/data

进入到bin目录下,用./zkServer.sh start
./zkServer.sh status命令启动zookeeper，两个命令都需要启动

## Tomcat与zookeeper的自启动

> Tomcat自启动

```bash
vim /etc/init.d/dubbo-admin
#!/bin/bash
#chkconfig:2345 20 90
#description:dubbo-admin
#processname:dubbo-admin
CATALANA_HOME=/opt/apache-tomcat-9.0.22
export JAVA_HOME=/usr/java/jdk1.8.0_141
case $1 in
start)  
    echo "Starting Tomcat..."  
    $CATALANA_HOME/bin/startup.sh  
    ;;  
  
stop)  
    echo "Stopping Tomcat..."  
    $CATALANA_HOME/bin/shutdown.sh  
    ;;  
  
restart)  
    echo "Stopping Tomcat..."  
    $CATALANA_HOME/bin/shutdown.sh  
    sleep 2  
    echo  
    echo "Starting Tomcat..."  
    $CATALANA_HOME/bin/startup.sh  
    ;;  
*)  
    echo "Usage: tomcat {start|stop|restart}"  
    ;; esac
```

```bash
chkconfig --add dubbo-admin
chmod 777 dubbo-admin 
service dubbo-admin start
```

> zookeeper自启动

```bash
vim /etc/init.d/zookeeper
#!/bin/bash
#chkconfig:2345 20 90
#description:zookeeper
#processname:zookeeper
ZK_PATH=/opt/zookeeper
export JAVA_HOME=/opt/jdk1.8.0_152
case $1 in
         start) sh  $ZK_PATH/bin/zkServer.sh start;;
         stop)  sh  $ZK_PATH/bin/zkServer.sh stop;;
         status) sh  $ZK_PATH/bin/zkServer.sh status;;
         restart) sh $ZK_PATH/bin/zkServer.sh restart;;
         *)  echo "require start|stop|status|restart"  ;;
esac
```

```bash
chkconfig --add zookeeper
chmod 777 zookeeper
service zookeeper start
```

