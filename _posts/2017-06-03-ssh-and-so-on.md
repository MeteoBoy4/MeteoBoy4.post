---
title: ssh那点事儿
date: 2017-06-03 21:05:21
tags: Linux
---

记录了我在Linux下ssh远程登陆的一些配置方法。

<!--more-->

## 免密登陆

大致的流程是在本地生成一对公钥私钥对，将公钥上传至远程服务器，以后每次登陆时ssh自动通过公钥（远程服务器上）私钥（本地）进行配对而验证身份:

`ssh-keygen -t rsa`, 一路回车，这会在你的~/.ssh下生成id_rsa（私钥）与id_rsa.pub（公钥）

上传你的公钥至远程服务器信任列表
```bash
cat ~/.ssh/id_rsa.pub | ssh username@remote-server 'cat >> ~/.ssh/authorized_keys'
```

如果有必要，在远程服务器上
```bash
chmod 640 .ssh/authorized_keys
chmod 700 .ssh
```

## 登陆命令简化

最直接想到的是将ssh对应的命令行一股脑地放进bashrc进行alias，但是这样只能简化ssh登陆的命令，scp的时候依旧需要输入一大串用户名和域名。

于是采用如下方法：

进入~/.ssh，编辑（或创建）config文件，加入

	Host haha
		Hostname your.host.name
		Port your_port
		User your_user_name

其中haha为你给远程服务器取的昵称，下面三行第一个词为关键字，第二个为你的对应信息。

记得修改config的权限为
```bash
chmod 600 ~/.ssh/config
```

好了，现在你登陆服务器只需要`ssh haha`，而文件传输只需要
```bash
scp haha:/path/to/your/file /local/path/
```


参考网站：[nerderati][]、[serverfault][]、[man page][]、[key management][]



[nerderati]: http://nerderati.com/2011/03/17/simplify-your-life-with-an-ssh-config-file/
[serverfault]: https://serverfault.com/questions/253313/ssh-hostname-returns-bad-owner-or-permissions-on-ssh-config
[man page]: https://linux.die.net/man/5/ssh_config
[key management]: https://docs.microsoft.com/en-us/windows-server/administration/openssh/openssh_keymanagement