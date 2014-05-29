---
layout: post
title: "Mac OSX vim 输入中文乱码问题"
date: 2014-05-18 22:05:24 +0800
comments: true
categories: vim 
---
###出现问题的环境:
1. OSX 10.9.2 英文  
2. VIM 7.3  

###问题描述:
1. 可以正常显示文件中得中文字符  
2. 输入中文字符会出现乱码  
3. 通过在.vimrc中设置编码格式无效  

###解决方案:
找到 `Terminal-->Preferences-->Settings-->Advanced-->Escape non-ASCII input with Control-V` 取消该选项即可。  

<!--more-->
如果你尝试了设置各种编码格式都无法解决乱码问题的话，可以尝试一下该方法  
如图:  
![示例](/images/blog/2014-05/20140519-1-vim.png)

