---
title: Android开发环境搭建
tags:
  - Android
categories:
  - Android
id: 109
date: 2016-04-15 09:58:41
---

Android是用Java编写的，所以安装JDK是一件事。据说由于版权问题2016年开发者大会Google会宣布放弃使用OpenJdk，这之前仍然使用Oracle的版本。

#### 安装JDK

1.下载链接[OracleJDK](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)，安装并配置环境变量.
2.  配置环境变量：JAVA_HOME: &nbsp; &nbsp;...<span class="color-new">**\jdk1.x.x_xx； &nbsp; &nbsp;&nbsp;**</span>PATH : &nbsp; &nbsp;...<span class="color-new">**\jdk1.x.x_xx\bin；**</span>
3.  之后再命令行输入&quot;**java -version**&quot;,查看是否成功。
注意：安装JDK和JRE的时候不要安装在同一个目录，一般默认路径安装是安装在同一文件夹。

#### 安装Android SDK

1.打开Android Developer官网的[工具下载](http://developer.android.com/intl/zh-cn/sdk/index.html#Other)页面，下拉找到&quot;**Get just the command line tools**&quot;对应下载选项并且下载Windows版本的SDK工具。
2.由于天朝的一些原因，SDK工具里的SDK Manager是无法下载SDK的,所以我们需要代理。打开国内一个Android工具网站[AndroidDevTools](http://www.androiddevtools.cn/)，下拉找到&quot;**Android SDK在线更新镜像服务器**&quot;，按照所给的镜像地址和端口对SDK Manager配置代理。
3. 配置完成之后，会立马刷新SDK列表，可以看到未下载的各种版本的SDK，SDK并不是所有版本都要安装。**Tools**下的全部下载，**Tools(Pre Channel)**预览版不用下载，**Android x.x.x(API XX)**从目前最新的Andorid N一直到Android 4.1都下载下来（只需要下载&quot;SDK Platform&quot;，&quot;Google APIs&quot; &quot;Sources for Android SDK&quot;）,**Extras**里的都可以下载下来。
4.  配置环境变量：PATH: ...\android-sdk-windows\platform-tools &nbsp;, &nbsp; ...\android-sdk-windows\tools.
5.  打开命令行工具，输入&quot;adb&quot;查看是否配置成功。

#### 安装Android Studio

1. 打开Android Developer官网的[工具下载](http://developer.android.com/intl/zh-cn/sdk/index.html#Other)页面，下拉找到&quot;**Select a different platform**&quot;对应下载选项并且下载Windows版本的不包含SDK的Android Studio,建议免安装版。
2. Android Studio 比较吃内存，所以建议到Android Studio的bin目录找到&quot;**studio64.exe.vmoptions**&quot;（64位系统），打开找到非注释的第二行&quot;**-Xmx**&quot;修改为2048或者4096.&quot;**-Xmx&quot;** 参数是 Java 虚拟机启动时的参数，用于限制最大堆内存。Android Studio 启动时设置了这个参数，并且默认值很小，没记错的话，只有 1028m。 一旦你的工程变大，IDE 运行时间稍长，内存就开始吃紧，频繁触发 GC，自然会卡。
3. 启动Android Studio，如果JDK配置好了，会进入下载SDK的页面，选择取消下载，完成，之后会直接进入Android Stuio开始页面，找到&quot;**Project Structute**&quot;设置选项，打开添加对应的SDK路径和JDK路径。
4.  新建项目，运行测试。

