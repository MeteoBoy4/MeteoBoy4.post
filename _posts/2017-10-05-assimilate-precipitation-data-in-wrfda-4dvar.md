---
title: WRFDA四维变分同化降水资料
date: 2017-10-05 21:22:30
tags: [同化, WRFDA, Python]
---

介绍了利用WRFDA的四维变分同化（4DVAR）来同化降水资料的流程。

<!--more-->

## 动机

传统意义上，小伙伴们使用WRF对高影响天气过程进行模拟时，最为关注的要素之一便是（累积）降水量。天有不测风云，模式也不是那么容易模拟好降水过程的。这时除了调整模式参数和配置外，最为有效的办法便是同化额外的观测资料了。

WRFDA是WRF团队开发的一整套同化程序，可以同化各种常规地面、高空观测，卫星辐射、雷达和降水观测资料。既然我们很关注降水量的模拟，在模拟预热期同化降水资料是不是更直接一些呢？

WRFDA支持三维变分（3DVAR）和四维变分（4DVAR），由于降水资料一般都是随时间累积量而非瞬时量，故应采用WRFDA的四维变分来同化降水资料。

## 准备降水资料

### 资料格式

WRFDA读取降水资料时，需要降水资料已经写成特定的ASCII格式。关于此格式[官网][]并没有给出详细的说明，但是给出了一个Fortran小程序[precip_converter][]，利用它可以将NCEP的Stage IV降水资料处理成WRFDA可以读取的格式，其中给出的核心独立子程序writerainobs就是输入经纬度与降水资料而将其以WRFDA要求的格式输出到文件中的。这样再好不过，我们将其稍加修改，使用f2py将其改造成Python可以调用的子函数，便可直接构造相应格式的降水资料。

### Fortran

稍加修改后的Fortran子函数放在我的[gist][]上，其中文件名需要被命名为ob.rain.yyyymmddhh.xxh，xx代表累积降水量的时间跨度，后面的部分由SUFFIX变量传入。Fortran文件中加入了“!f2py”开头的注释，这是f2py特定可以读取的变量声明，以告诉Python哪些变量是准备输入的，哪些是准备输出的，以使得Python更好地生成文档以及使用Fortran函数。

### f2py

接下来便可通过

	f2py -c -m 外部module名 Fortran文件名
	
生成外部module供Python调用。


### Python

我们以[TRMM 3B42][]降水资料的处理为例写一个Python调用该module的小程序。该资料为3小时间隔，根据WRFDA文档的介绍，同化6小时累计是比较合理的，故我们取2个时次的平均。并手动生成经纬度一维数组以及时间后缀，然后在[Python脚本][]中使用writerainobs函数，注意外部Fortran函数的import要放在脚本最前面。

降水资料数据就生成啦！

## 同化降水资料

### 链接

接下来按照官网的[例子][]一步一步做即可。记得将wrfinput_d01链接为fg（这两个文件必须都在同化工作目录下存在，不能只把wrfinput_d01链接为fg），同时将wrfbdy_d01直接链接过来（此处不用重命名）由于4DVAR需要不断运行WRF，故边界条件必须提供（3DVAR则不需要）；此处三个文件的名字都是强制的，所以在namelist里不必指定其位置与名字，降水资料在链接到同化工作目录时有特定的命名规则，详见官网。

### namelist

同化的namelist.input开启var4d，这里var4d_bin可设为一小时，而var4d_bin_rain则应该设为6小时（降水资料的累计时间），这样跑4DVAR的时候跑满6小时后反向时会以一小时为单位swap；在wrfvar4下开启use_rainobs；wrfvar5中指定是否进行质控并指定质控的严格程度，如果你希望将资料完全进入同化系统则可以将check\_max\_iv设为False，这样生成的rej\_obs\_conv\_们（包含被拒绝入同化的数据）会全部为空；wrfvar6中控制了几层循环（天啊那不是更慢了），以及收敛的判据，WRFDA收敛是以目标函数的导数缩小为原来的百分比来判定的；时间对应到累积降水的起始与终止时刻，domain与physcis的信息完全依照wrfinput的来（即生成wrfinput的方案和配置，有些方案WRFPLUS里并没有，但是它会按照它有的来，不会报错）。具体的选项可以参考User's Guide与官网的Tutorial的PPT。

### 同化——更新初始场

然后开始同化！

	./da_wrfvar.exe &> wrfda.log &
	
可以看到利用WRFPLUSV3（一个简化的非线性WRF版本），同化系统不断地从降水累积起始时刻到终止时刻，来回循环地跑简化的WRF（正着跑，倒着跑..），并以最大梯度方向调整初始场使得WRF跑的结果向观测资料靠近，cost_fn与grad_fn可以看到梯度和目标函数的情况，还有很多文件可以查看WRFDA的具体诊断输出量，由于每一次循环相当于跑了二次（6小时）的WRF，完成4DVAR的速度会非常慢。当然从这里也可以看到4DVAR的好处，可以利用模式自身的动力对初始场进行调整，并且可以同化“未来”的观测值。

同化成功后将生成新的初始场，名为wrfvar_output，其格式与wrfinput_d0们完全一样，可以通过一些画图软件对比wrfinput与wrfvar_output的增量情况。

### 同化——更新侧边界条件

更新了初始场以后，边界条件也必须更新以使得其与新的初始场保持协调，当然这一步只需对最外层进行，因为内层的侧边界条件是由外层提供的。

这一步是由da_update_bc.exe完成的，它也需要配一个namelist称为parame.in，将其中的update_lateral_bdy设为true，然后告诉它wrf_bdy的地址（将会覆盖原来的边界条件）和da_file（即上一步生成的wrfvar_output），加上一些别的选项（见User's Guide）运行，更新！

至此，生成的新初边值就可以链接回原来WRF的运行目录，正常跑WRF即可。

## 关于同化内层区域

上面提到4DVAR在更新初始场的时候必须提供边界条件，这就给同化内层区域带来了问题（3DVAR不存在这一问题）：在运行real.exe或wrf.exe的时候内层区域只会生成初始场，并不会生成边界条件（wrfbdy_d02）。那么这个文件从哪里来呢，以下是我询问[wrfhelp](mailto:wrfhelp@ucar.edu)的回复：

>As you say, "producing exactly the wrfbdy_02 manually is not so easy". But unfortunately this is the only way to do 4DVAR assimilation on a nested domain. If you have any questions on how to do this we are happy to help (there is enough information in the nested WRF file to create this domain exactly), but unfortunately there is no way around this inconvenience.

也就是说，需要自己手动生成内层的边界条件！

哎~~

不过办法还是有的，就是麻烦点：

1.在WRF运行目录，把原来跑多层的namelist.input先备份以下；
2.将WPS目录下的met_em.d02们链接到WRF目录，但是链接为对应时间的met_em.d01们；
3.修改namelist.input：time_control部分不用动，在domains部分将max_dom修改为1，然后将time_step、e_we、e_sn、dx、dy的原第一列（对应最外层）删除，第二列（对应domain2）顺势成为第一列；注意此处grid_id、parent_id、i_parent_start、j_parent_start、parent_grid_ratio、parent_time_step_ratio保留原第一列的值（即都为1），虽然这样生成的wrfbdy_d02中的netCDF属性中这些值会为1，与其应有值冲突，但是并不会使得WRF、WRFDA报错；
4.运行real.exe，生成内层的边界条件文件（虽然名字叫wrfbdy_d01）；
5.按照与最外层一样的方法运行4DVAR。

上述办法不适合4DVAR循环同化（cycling）：wrf.exe不会生成边界文件。

试试看用新的初边值条件跑出来的结果，降水上是不是与你想要的结果更接近了呢？








[官网]: http://www2.mmm.ucar.edu/wrf/users/wrfda/
[precip_converter]: http://www2.mmm.ucar.edu/wrf/users/wrfda/download/precip_converter.tar.gz
[gist]: https://gist.github.com/MeteoBoy4/132197f0135d11d0680db33bb7a7a794#file-write_rainobs-f90
[TRMM 3B42]: http://apps.webofknowledge.com/full_record.do?product=UA&search_mode=GeneralSearch&qid=1&SID=N1Qh2M3LsfbRqvA82T1&page=1&doc=1
[例子]: http://www2.mmm.ucar.edu/wrf/users/docs/user_guide_V3.8/users_guide_chap6.htm#_Precipitation_Data_Assimilation
[Python脚本]: https://gist.github.com/MeteoBoy4/132197f0135d11d0680db33bb7a7a794#file-nc2wrfda-py