---
title: Linux常用命令
date: 2017-08-21 15:45:22
tags:
  - Linux
---
Linux 常用命令
<!--more-->

```linux
hexdump -C binfile | less   查看二进制文件

ls -lt  查看目录中文件按照时间顺序排序
ls -ltr 查看目录中文件按照时间顺序倒序排序
ls -Slh 查看目录中文件按照大小排序
ls -Slhr  查看目录中文件按照大小倒序排序

tar -czvf fileName.tar dirName  tar压缩包
tar -zxvf fileName.tar  tar解压

zipinfo -1  查看zip包中的文件

jar xvf 解压jai包和war包
jar tvf 查看jar包内容

jmap -dump:format=b,,file=fileNaem.bin <pid>  导出java dump文件

scp usreName@Host:remoteDir localDir  从远程下载文件到本地
scp localDir userName@Host:remoteDir  上传本地文件到远程

----------VIM---------
:set fileencoding 显示文件编码
:set encoding=utf-8 设置文件编码
:set number 显示行号
:set nonumber 不显示行号


---------HEXO---------
hexo server
hexo new article
heox g -d

```
