---
title: 使用ERA-Interim驱动WRF3.9
date: 2017-05-30 20:24:42
tags: WRF
---

介绍了如何使用ERA-Interim资料驱动WRF3.9进行模拟。

记录了我遇到的困难和解决方法。

<!--more-->

## 缘起

ERA-Interim是欧洲中心一套再分析资料，鉴于欧洲中心在数字天气预报上的领先地位，其预报场受到天气预报员的青睐，所在在利用WRF-ARW进行模拟时自然也想用其再分析场作为驱动，希望得到好的模拟效果，但是毕竟欧洲中心和WRF的开发者（UCAR/NCAR）不是同源，其整合程度也不像北美的几家产品一样好，需要摸索一下。

## 步骤

首先参考[气象家园：夏朗的芒果][]，使用Python脚本下载高空场和地面场变量（其将需要的下载量压缩到了最小），并下载了不随时间变化的geopotential.grib和landsea-mask.grib（保持四个文件分辨率一致）。

其次也是按照上述帖子，将ungrib/Variable_Tables/Vtable.ERA-interim.pl链接到WPS目录下，分别对高空场和地面场进行ungrib，PREFIX分别命名为‘3D’与‘SFC’；然后将start_date与end_date改成1989-01-01_12（这是两个不随时间变化的场所带自描述的时间），将PREFIX分别改为‘Z’与‘LSM’，再次ungrib；最后修改namelist.wps为

	constants_name = 'LSM:1989-01-01_12', 'Z:1989-01-01_12',
	fg_name = '3D','SFC',

然后执行metgrib.exe，生成对应的met_em.nc

## 错误

在进入WRF的时候，real.exe报以下错误


	----------FATAL CALLED--------------
	FATAL CALLED FROM FILE: <stdin> LINE: 2948
	mismatch_landmask_ivgtyp
	------------------------------------


我一度怀疑是一开始没有并入不随时间变化的两个场，[Computering.io][]也证实了这两个场很重要（包括后面提到的SST，如果需要更新的话也需要包括进来）。

## 解决方法

根据[WRFforum][]的讨论，问题出在SST上

{% blockquote @pau: http://forum.wrfforum.com/viewtopic.php?f=6&t=10010  Re: real.exe mismatch_landmask_ivgtyp %}
Taking a look to the code and the errors I figured out that the problem was the SST after running the metgrid.exe, so I removed SST from the ERA-Interim files I downloaded and real.exe showed no errors. 
{% endblockquote %}

但是我不甘心将SST移出驱动场，于是顺着[wrf-users的mailing list][]发现想要保留SST可以，但是需要修改METGRID table。于是我又找到了[/dav/null][]的文章，里面指明了修改METGRID.TBL的办法：


	SST
	  interp_option=sixteen_pt+four_pt+wt_average_4pt+wt_average_16pt+search
	  missing_value=-1.E30
	  masked=land
	  interp_mask=LANDMASK(1)
	  fill_missing=0.
	  flag_in_output=FLAG_SST 


经过这样修改后的SST场（在met_em.nc中）不仅局限在大洋上，而且在内陆的河面、湖面上也会进行插值，从而得到正确的插值结果。

再次运行real.exe正常，运行wrf.exe也没有问题。

## 另外一种解决方法

根据[气象家园：tbag][]的跟帖，这一问题还可以通过修改namelist.input中的surface_input_source来解决。根据WRF官网的说明：

![surface_input_source][]

看来在3.8以后模式会默认直接拿着geogrid里的mask进行插值，不会再重新调整（推论是如果要做地形敏感性实验，得把这个参数设置成3），这造成了SST的插值错误，如果将其修改成1，那么不会报错（3.8之前该参数默认值不同，所以可能不会报错）。






[气象家园：夏朗的芒果]: http://bbs.06climate.com/forum.php?mod=viewthread&tid=30962
[WRFforum]: http://forum.wrfforum.com/viewtopic.php?f=6&t=10010
[wrf-users的mailing list]: http://mailman.ucar.edu/pipermail/wrf-users/2016/004235.html
[Computering.io]: http://computing.io/wp/2012/12/running-wrf-with-ecmwf-data/
[/dav/null]: https://dvalters.github.io/2016/12/23/ECMWF-data-in-WRF.html
[气象家园：tbag]: http://bbs.06climate.com/forum.php?mod=viewthread&tid=46183
[surface_input_source]: /images/surface_input_source.png
 