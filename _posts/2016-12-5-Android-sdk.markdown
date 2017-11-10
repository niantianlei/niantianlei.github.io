---
layout: post
title: 安装Android sdk遇到的问题
subtitle:   " \"the questions of installing Android sdk\""
author:     "Nian Tianlei"
header-img: "img/post-bg-2016.jpg"
tags:
    - Android
---

> 下滑这里查看更多内容

解决*sdk manager*链接google失败或者下载速度太慢的问题。    
首先打开sdk manager.exe然后依次点击tools->options，    
将Proxy Settings 里的HTTP Proxy Server和HTTP Proxy Port分别设置成     
mirrors.neusoft.edu.cn和80。  
然后在others选项卡中选中Force http…（就是第一个）勾上。    
最后重新启动。    


若遇到  
*Warning  
A folder failed to be renamed or moved. On Windows this typically means that a program is using that folder (for example Windows Explorer or your anti-virus software.)
Please momentarily deactivate your anti-virus software or close any running,programs that may be accessing the directory... When ready, press Yes to try again.*    
问题    
首先进入sdk根目录，打开*temp*文件夹，下载好会有一个    
*tools——r25.2.3-windows.zip*压缩文件（我下载的版本），解压此文件。  
然后将里面的文件复制覆盖到sdk根目录的tools文件夹中的文件。  
重新打开*sdk manager.exe*，即可完美运行。  

