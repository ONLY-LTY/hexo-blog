---
title: Linux常用命令
date: 2017-08-21 15:45:22
tags:
  - Linux
---
Linux 常用命令

```linux
hexdump -C binfile | less   查看二进制文件
sudo tcpdump -i eth0 -Xlnps0 port 10000 抓包


----------查看文件---------
ls -lt  查看目录中文件按照时间顺序排序
ls -ltr 查看目录中文件按照时间顺序倒序排序
ls -Slh 查看目录中文件按照大小排序
ls -Slhr  查看目录中文件按照大小倒序排序
df -lh  查看当前磁盘的使用情况
du -sh  查看当前文件夹的大小

----------解压---------
ls -lt  查看目录中文件按照时间顺序排序
tar -czvf fileName.tar dirName  tar压缩包
tar -zxvf fileName.tar  tar解压
jar xvf 解压jai包和war包
jar tvf 查看jar包内容
zipinfo -1  查看zip包中的文件

scp usreName@Host:remoteDir localDir  从远程下载文件到本地
scp localDir userName@Host:remoteDir  上传本地文件到远程

----------JVM---------
jmap -dump:format=b,,file=fileNaem.bin <pid>  导出java dump文件
jstat -gcutil <pid> 1s  监控jvm gc情况
jmap -heap <pid> 打印java heap情况

----------VIM---------
:set fileencoding 显示文件编码
:set encoding=utf-8 设置文件编码
:set number 显示行号
:set nonumber 不显示行号

---------MAC---------
sudo spctl --master-disable MAC高版本允许任何来源安装

---------LOG---------
tail -f recommend.log | grep -Eo "LHot-1500-[0-9]{1,2}" 输出grep匹配内容
cat /proc/63/status | grep "Threads" 查看进程63的线程数量
```
