---
layout: post
title: "netbeans debug hotspot"
date: 2014-06-21 18:02:59 +0800
comments: true
categories: jvm
keywords: jvm,hotspot,netbeans,debug
description: Netbeans debug hotspot 
---
学习JVM的过程中肯定不能少了对JVM的调试，进行就学习一下怎样用Netbeans调试Hotspot。  
####编译过程
环境：  
>Ubuntu12.04  
>OpenJdk 7u  
>Netbeans7.0.1(c/c++)   
 
1.安装编译所需要的工具。  
```java
apt-get install ant mercurial gawk g++ libcups2-dev libasound2-dev libfreetype6-dev libx11-dev libxt-dev libxext-dev libxrender-dev libxtst-dev libfontconfig1-dev
```
<!--more-->
2.clone openjdk
```java
hg clone http://hg.openjdk.java.net/jdk7u/jdk7u/ jdk7u
```
3.进入jdk7u目录，执行下面的脚本下载openjdk源代码  
```java
./get_source.sh
```
4.开始编译OpenJdk,为了方便写个小脚本(build.sh),该脚本的Gist的地址:[查看](https://gist.github.com/zarue/0c6dd39d3e271888f02d#file-1-build-sh),将该脚本放在jdk7u目录下面  
```java
#!/bin/bash
unset JAVA_HOME
export LANG=C
#必须开启，jdk在编译过程中会联网下载一些openjdk本身未包含的第三方库
export ALLOW_DOWNLOADS=true
export USE_PRECOMPILED_HEADER=true
export SKIP_DEBUG_BUILD=false
export SKIP_FASTDEBUG_BUILD=true
export DEBUG_NAME=debug
#ALT_BOOTDIR 是你本机jdk的目录
export ALT_BOOTDIR=/home/cheney/Downloads/jdk1.6.0_45
source jdk/make/jdk_generic_profile.sh
make sanity && make
```
5.执行build.sh 开始编译过程，大约耗时20-30分分钟  
```java
./build.sh
```
####可能遇到的问题:  
1、  
```java
.src/share/vm/runtime/interfaceSupport.hpp:430:0: error: "__LEAF" redefined [-Werror] 
/usr/include/x86_64-linux-gnu/sys/cdefs.h:44:0: note: this is the location 
of the previous definition 
```
有两种解决方法:   
1.参考这个：http://hg.openjdk.java.net/hsx/hotspot-comp/hotspot/rev/a6eef545f1a2    
2.这个问题在jdk7u中已经修复，直接使用jdk7u版本的源码就可以了。   
 
2、"*** This OS is not supported:" 'uname -a'; exit 1;
解决方法:  
uname -r  
\#查看当前的内核版本：3.11.0-15-generic  
找到下面的文件：/hotspot/make/linux/Makefile   
\#在这行最后加上当前的内核版本3.11%，  
 SUPPORTED_OS_VERSION = 2.4% 2.5% 2.6% 2.7% 3.11%  

3、Error occurred during initialization of VM java/lang/NoClassDefFoundError: java/lang/invoke/AdapterMethodHandle  解决方法:  
这是因为编译Openjdk的所用的Jdk版本不符合要求导致的，我这里用的`jdk1.6.0_45`  
如果还遇到其它问题可以自行Google,一般都能解决。  
####使用Netbeans调试  
1.安装Netbeans7.0.1 我尝试了8.0，7.4 都不能正常进行Debug，最后换了7.0.1就正常了，这里仅供参考。  
2.新建一个项目，选择“基于现有源代码的C/C++项目”，在“源代码文件夹目录”选择openjdk下的hotspot目录，“选择配置模式”中选择“定制”。  

3.下一步，“使用现有的makefile”：选择hotspot/make目录下的Makefile文件。  

4.构建：“构建命令”：
```java
${MAKE} -f Makefile clean jvmg ALT_BOOTDIR=/home/cheney/Downloads/jdk1.6.0_45 ARCH_DATA_MODEL=64 LANG=C   ZIP_DEBUGINFO_FILES=0 
```
 如果你是64位系统那么需要指定ARCH_DATA_MODEL=64，另外如果不指定ZIP_DEBUGINFO_FILES=0，那么需要在编译完成后到jvmg目录下面执行unzip libjvm.diz 解压出调试需要的符号信息。否则将不能进行调试。   

5.运行-运行命令
```java
"/home/cheney/soft/jdk7u/hotspot/build/linux/linux_amd64_compiler2/jvmg/gamma"   -XX:StopInterpreterAt=1 Test
```
-XX:StopInterpreterAt=1的作用是当遇到序号为<n>的字节码指令时，便会中断程序执行，进入断点调试，但是我不指定这个参数也照样可以进行调试。Test 是我自己写的测试类，以后如果想调试哪个类就在这里更换。  

6.运行-环境变量
```java
LD_LIBRARY_PATH /home/cheney/soft/jdk7u/hotspot/build/linux/linux_amd64_compiler2/jvmg
JAVA_HOME /home/cheney/soft/jdk7u/build/linux-amd64/j2sdk-image
CLASSPATH=${JAVA_HOME}/lib/dt.jar:${JAVA_HOME}/lib/tools.jar:/home/cheney
```
注意：这里要把Test所在的目录添加到环境CLASSPATH里面。  

7.接下来就是等待编译过程了，编译完成之后就可以进行调试了。

