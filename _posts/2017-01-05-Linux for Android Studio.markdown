---
layout:     post
title:      "在Linux系统下运行Android Studio"
subtitle:   "实用的AS插件"
date:       2017-01-05 12:00:00
author:     "foxleezh"
header-img: "img/post-bg-2017-01-05.jpg"
catalog: true
tags:
    - Android Studio
---

最近因为公司需要做视频方面的内容，找了几个第三方框架，而这些框架大部分代码都是C层的代码，而且有特别复杂的编译脚本，这些脚本都无法在windows上运行（坑），所以不得不转战到mac或linux平台，资金有限，所以放弃mac,直接装linux系统

# 一.装linux系统
* 准备工作
1.到官网下载linux的镜像文件 http://cn.ubuntu.com/download/
2.下载ultraiso 地址百度下，一搜一大把
3.准备一个8G U盘

* 制作启动盘
1.运行ultraiso,打开下载好的镜像文件
2.菜单 启动-》写入硬盘印象
3.点击格式化后写入

* 开始安装
1.windows设置U盘启动（根据不同BIOS百度去，一盘开机按F2）
2.启动后直接就进入安装，根据提示一直点下一步下一步就安装了

# 二.安装Android studio

一些坑
装好linux系统后无法上网，让我很是头疼，后来解决方案是用手机共享wifi，连接手机后一般都有这个选项，没有的自行百度
linux系统和windows安装软件区别很大，一般的开发工具都是命令行安装的，比如git,gradle等，安装方法基本是sudo apt install xxx

这里特别讲一下翻墙软件shadowsocks的安装和使用
* 1.安装pip
```sudo apt-get install python-pip```
可能会提示安装一些依赖，按提示安装即可

* 2.接着安装shadowsocks
```sudo pip install shadowsocks```
通过以上命令安装shadowsocks，为了避免权限不够，在命令行前加上sudo

* 3.配置本地文件
在任意目录下创建  shadowsocks.json 文件，将下面的内容放进去：
```
{
"server":"my_server_ip",
"server_port":8388,
"local_address": "127.0.0.1",
"local_port":1080,
"password":"mypassword",
"timeout":300,
"method":"aes-256-cfb",
"fast_open": false,
"workers": 1
}
```
基本上就是windows端的配置文件
* 4.运行sslocal -c xxx(json文件路径)
这时可能报错 "ERROR method rc4-md5 not supported"
原因是shadowsocks的版本太低，要升级下
sudo pip install shadowsocks --upgrade
升级好后基本就没问题了，运行后显示
2016-10-14 10:30:20 INFO     loading libcrypto from libcrypto.so.1.0.0
2016-10-14 10:30:20 INFO     starting local at 127.0.0.1:1080
说明已经联上vpn了

* 5.设置代理
联上vpn还不能翻墙，还需要设置代理，浏览器代理自行百度去，我这里是设置的全局代理
系统设置-》网络-》网络代理-》Socks主机设置为127.0.0.1 1080这里跟配置文件里的loacal_address要一致
参考https://aitanlu.com/ubuntu-shadowsocks-ke-hu-duan-pei-zhi.html

有了翻墙工具，很多事情就好办了
* ##### 下面开始讲android studio的安装
1.到google官网下载linux版本 https://developer.android.com/studio/index.html# downloads
2.解压下载好的文件到某一目录下 比如/home/xxx/android studio
3.运行/home/xxx/android studio/bin/studio.sh
这里可能有些机子上没配置jdk
参考http://www.cnblogs.com/kerrycode/archive/2015/08/27/4762921.html
(1)java -version查看java版本，有说明安装了jdk
(2)echo $JAVA_HOME 如果没有则说明没有配置环境
配置环境
(1).打开bashrc文件
sudo gedit ~/.bashrc
(2).在打开的文档最后添加以下内容
```
export JAVA_HOME=xxx
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
export PATH=${JAVA_HOME}/bin:$PATH
```
这里可能有些人找不到JAVA_HOME的路径
(1.which java  返回java的执行路径
(2.ls -lrt /usr/bin/java(上一个命令返回的内容)
(3.ls -lrt /etc/alternatives/java(上一个命令返回的内容)
这次返回的内容就是java的安装路径
这个方法适用于所有的命令，方便找出其安装路径
(3).刷新
source ~/.bashrc
参考http://www.cnblogs.com/samcn/archive/2011/03/16/1986248.html
配置好运行后会自动去下载sdk

# 三.一些坑
###### (1).如果是64位的机子运行项目的时候会报aapt error=2,找不到文件和目录

这是因为aapt是32位的，64位对它的兼容不好，解决方案
```
sudo apt-get -qqy install libncurses5:i386 libstdc++6:i386 zlib1g:i386
```
参考http://stackoverflow.com/questions/19523502/how-to-make-androids-aapt-and-adb-work-on-64-bit-ubuntu-without-ia32-libs-work
###### (2).升级gradle 版本
参考http://www.jianshu.com/p/707b871f9350

(1.download
```
wget https://services.gradle.org/distributions/gradle-2.14.1-bin.zip
```

(2.unzip 
```
unzip gradle-2.14.1-bin.zip
```

(3. mv 
```
mv gradle-2.14.1 /usr/local/
```

(4.vim /etc/profile(也可以是其他配置比如~/.bashrc)
```
export GRADLE_HOME=/usr/local/gradle-2.14.1
PATH=$PATH:$GRADLE_HOME/bin
```

(5.source /etc/profile
```
source /etc/profile
```

###### (3).jdk找不到tool.jar文件
由于ubuntu默认安装JAVA的openjdk版本没有装tools.jar,因此在某些情况下必须安装openjdk的完全版，如下命令即可完成：
sudo apt-get install openjdk-8-jdk

