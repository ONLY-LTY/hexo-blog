---
title: Linux远程登录
date: 2017-08-21 15:46:16
tags:
  - Linux
---
Linux 远程登录 免密码

##### 第一步 将本地的 id_rsa.pub 文件上传到远程服务器

```linux
scp /Users/LTY/.ssh/id_rsa.pub userName@Host:~/

#输入密码 回车
```

##### 第二步 登录远程服务器

```linux
ssh usreName@Host

#输入远程服务器密码 回车
```

##### 第三步 生成秘钥(如果远程服务器存在 用户目录下存在.ssh 可以跳过)

```Linux
ssh-keygen

#一直回车
```
##### 第四步 将第一步上传的本地秘钥复制到 .ssh/authorized_keys

```linux
cat id_rsa.pub >> .ssh/authorized_keys

###如果 authorized_keys 文件不存在 新建一个
```
##### 第五步 修改authorized_keys 文件权限 为600

```linux
chmod 600 .ssh/authorized_keys
```
##### 第六步 删除你的本地 秘钥 然后退出服务器 再次登录
