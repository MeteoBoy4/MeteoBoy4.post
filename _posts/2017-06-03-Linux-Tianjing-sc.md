---
title: Linux下登陆天津超算
date: 2017-06-03 12:39:33
tags: Linux
---

记录我在CentOS下登陆天津超算中心的步骤。

<!--more-->

## 缘起

日常工作平台是CentOS, 有了超算账号以后自然也想在Linux底下登陆。

## 步骤

由于缺少像Windows平台下的VPN客户端，需要用脚本进行VPN登陆，参考[勇敢的心][]的方法：

* 用firefox连接nscc-tj.gov.cn
* 用超算VPN账号密码登陆
* 下载弹出的文件sslvpn_jnlp.cgi（这个文件会在/etc/hosts中添加重导向链接，只对每次登陆有效，默认绑定22端口）
* 修改sslvpn_jnlp.cgi中端口为2222（因为22端口默认被系统的ssh占用）
* 使用javaws运行sslvpn_jnlp.cgi（需要sudo，因为修改hosts的权限）```bash
sudo javaws sslvpn_jnlp.cgi
```
* 脚本登陆VPN完成，此时便可用命令行ssh到天津超算上
```bash
ssh -Y username@TH-1A-LN* -p 2222
```

## 工作相关

* 想要用scp传文件，指定端口的时候要加-P 2222
* 也可以使用GNOME下的files来作为图形化的FTP工具，点击右下角的Connect to Server，键入```sftp://username@TH-1A-LN*:2222```即可





[勇敢的心]: http://goodluck1982.blog.sohu.com/251615859.html