---
layout: post
title: "Mac OSX vim display Chinese as garbled characters"
date: 2014-05-18 22:05:24 +0800
comments: true
categories: vim
keywords: mac,vim,garbled,chinese
description: Mac OSX vim display Chinese as garbled characters
---
### Envrionment:
1. OSX 10.9.2 English  
2. VIM 7.3  

### Problems:
1. Can display Chinese characters in file  
2. Can not display Chinese characters inputed from screen  
3. Add encoding settings to .vimrc does not work  

### Solution:
find  `Terminal-->Preferences-->Settings-->Advanced-->Escape non-ASCII input with Control-V` and uncheck this option。  

<!--more-->
If you have tried many other approach to solve garbled characters problem like this, you should try this soution.    
Figure:  
![Figure](/images/blog/2014-05/20140519-1-vim.png)

