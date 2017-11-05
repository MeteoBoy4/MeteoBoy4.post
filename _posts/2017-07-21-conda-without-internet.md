---
title: 在无法连外网的服务器上安装Python包（conda）
date: 2017-07-21 20:51:38
tags: Python
---

服务器无法联网，而我需要安装netCDF4，xarray等第三方Python包，包依赖令人却步。

Conda（Anaconda）成为了救世主。

<!--more-->

## 缘起

编写的Python程序计算耗时越来越长，在自己台式机上跑已经愈发困难（心疼），故需要将其移植到服务器上，但手头有的服务器均无法连接外网，而我的程序需要用到诸如netCDF4、xarray等第三方包，如若手动安装，复杂（不熟悉）的包依赖关系令人却步，故而想到求救Anaconda。

## 步骤

### 安装anaconda

从[官网][]上下载anaconda安装包，传至服务器后直接进行安装（可选择自定义安装目录）。

### 建立本地库镜像

直接安装单个包会有依赖冲突，故我们在可以上网的机器上使用wget将anaconda[官方库集合][]（的一部分）下载到本地，以期作为一个本地库的镜像供远程服务器使用。注意官方库并非将每个包的元信息写在单个文件中，而是将所有包的信息写在一个repodata.json以及repodata.json.bz2中。

由于暂时只用Python2.7，所以下载的时候也只下载对应版本的包以减少下载量（一些包不含有版本信息，所以不能只下2.7，而是要用排除法排除其他Python版本对应的包），下载命令如下：
```bash
wget -r --no-parent -R --regex-type pcre --reject-regex '(.*py26.*)|(.*py3[3456].*)' https://repo.continuum.io/pkgs/free/linux-64 
```
以及repo.continuum.io/pkgs/free/noarch的内容（不然conda会警告）。

按照如上方法下载下来约有18G。

### 在服务器上测试

将下载下来的包使用rsync（这样更新时会便捷些）上传至不能联外网的服务器。

```bash
rsync -av --delete -r 本地目录 远程目录
```
这样服务器上的这一目录便成为我们自己建立的一个channel，我们希望可以通过用其自动检测依赖关系，安装相应的Python包。

包的搜索：

```bash
conda search -c file://Path_to_your_channel/repo.continuum.io/pkgs/free --override-channels --offline 包名称
```

包的更新（安装）

```bash
conda update (install) -c file://Path_to_your_channel/repo.continuum.io/pkgs/free --override-channels --offline 包名称
```

注意到repo等路径是你安装官网的路径结构下载下来的结果，而指定不用深入到构架（linux64）目录下，系统会自动检测。

如果下次想要更新本地库，则可以再用rsnyc重复下上述过程。

### conda配置

每次安装更新包要输入一大堆太麻烦，我们通过修改~/.condarc来改变默认寻找的channel

在~/.condarc中加入如下内容，使其覆盖原有的default channel（repo.continuum.io/pkgs/free/linux-64），并默认不再联网进行搜索

	channels:
	  - file:///Path_to_your_channel/repo.continuum.io/pkgs/free
	 
	 offline: True
	 
以后再要安装（更新）时像平时那样即可（conda install/update 包名称）。

参考网站：[stackoverflow][]、[conda document][]




[官网]:https://www.continuum.io/downloads
[官方库集合]:https://repo.continuum.io/pkgs/free/
[stackoverflow]:https://stackoverflow.com/questions/37391824/simply-use-python-anaconda-without-internet-connection
[conda document]: https://conda.io/docs/config.html