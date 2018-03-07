---
title: WRF格点下计算涡度散度谜团——公式篇
date: 2017-10-03 17:24:08
tags: [诊断分析, WRF, Fortran]
---
描述了WRF原生格点下计算涡度散度的迷惑和思考。

<!--more-->

## 迷惑

涡度和散度算是最常用、最普遍的风场诊断量了，在再分析资料的等经纬度网格下，计算涡度和散度只需采用简单的中央差分，但是到了WRF的Arakawa C水平网格并且考虑地图投影时，我就被涡度和散度的计算给搞糊涂了。

## WRF格点

WRF采用Arakawa C交错格点，水平面上，UV风速在其垂直方向相对于动热力量被交错了半个格距，热动力量所在格点称为质量格点（mass point），u（v、w）所在位置称为u（v、w）格点，这样做的好处是风速采用中央差分空间求导时只需要用相邻u（v、w）格点的量进行，格距为一倍模式格距，对应导数点正好落在质量格点上，在不需要增加计算量的情况下提高了求导的空间精度。

![C_Grid][]

另外，为了将地球球面转换为WRF的计算平面，需要采用地图投影，由于地球球面无法完美展开到连续平面上，故任何地图投影都会产生变形。WRF中采用了保角变换，保证地图投影的角度关系不变，但是空间距离会变形。变形的程度用地图放大系数（map scale factor）来表示

$$m = \frac{WRF格距}{地球球面距离}$$

在WRF的输出文件中会包含各个格点上的地图放大系数，可以直接用NCL读取。

## 正交曲线坐标系

上述的地图投影坐标系其实是一种[正交曲线坐标系][]，球坐标系也是它的一员。在数学家的口中，常常用拉密系数来表达放大系数。坐标变元$dq_j$要乘以拉密系数才等于坐标线元。**在数值上拉密系数等于地图放大系数的倒数**。

在正交曲线坐标系下，其矢量运算有着一套自有的方法，其数学推导不详，现给出求涡度和散度的公式

![vorticity][]

![divergence][]

由于我们一般首先关注水平散度和垂直涡度，故将上述公式的二维形式导出。

\begin{equation}
\triangledown_{h}\times \boldsymbol{V} = \frac{1}{H_1H_2}
(\frac{\partial(H_2u_2)}{\partial q_1} - 
\frac{\partial(H_1u_1)}{\partial q_2})\boldsymbol{e_3}
\end{equation}

\begin{equation}
\triangledown_{h}\centerdot \boldsymbol{V} = \frac{1}
{H_1H_2H_3}(\frac{\partial(H_2H_3u_1)}{\partial q_1} + 
\frac{\partial(H_3H_1u_2)}{\partial q_2})
\end{equation}

这里值得注意的是，在垂直方向上未做投影，故$H_3=1$。而WRF中*Lambert conformal，polar stereographic，和Mercator*投影是**各向同性**的，即一个格点上x、y方向的地图放大系数是相同的，故$H_1 = H_2$。

## 交错格点

既然WRF使用了交错格点，我们就希望可以尽量利用到其优势，使得中央差分求导的空间精度尽量地高。另外，由于涡度与散度常常需要与热动力量进行进一步计算，故我们希望求出的点对应在质量格点的位置。

对于散度来说情况比较明朗，因为空间的求导方向与格点的交错方向正好是一致的（见WRF格点说明），直接对相邻格点进行一倍格距的求导，所求点就对应着质量格点上的导数值，直接套用正交曲线坐标系的公式即可。

对于涡度来说，情况就要复杂一些了，我们希望求出的涡度值也对应在模式质量格点上，而求导方向与格点交错方向是正交的（见WRF格点说明），如果直接使用一倍格距对相邻格点的值进行求导，对应导数的位置在格点的四个角上，既不是质量格点，也不对应u、v格点，而且很难再通过平均等手段将其转换到质量格点上。不得已我们只好采取2倍格距，使求得导数值位置对应到“中央的”相应u（v）格点上去，然后再通过质量格点两侧u（v）格点上导数值的平均，求出质量格点上的对应导数值。注意到，这样求出的涡度空间精度低，还进行过一次平均，与散度相比粗略了很多。

## 参考代码

通过以上的讨论，求涡度与散度的思路基本明确，下面先给出较简单的散度的Fortran参考代码片段，再给出涡度参考代码片段。
```Fortran
DO k = 1,nz
    DO j = 1,ny
        DO i = 1,nx
            mm = msft(i,j)*msft(i,j)
            dudx = ( u(i+1,j,k)/msfu(i+1,j) - u(i,j,k)/msfu(i,j) )/dx*mm
            dvdy = ( v(i,j+1,k)/msfv(i,j+1) - v(i,j,k)/msfv(i,j) )/dy*mm
            diver = dudx + dvdy
            div(i,j,k) = diver*1.D5
        END DO
    END DO
END DO
```
注意代码中msft(u,v)对应着质量、u、v格点的地图放大系数。

```Fortran
DO k = 1,nz
    DO j = 1,ny
        jp1 = MIN(j+1, ny)
        jm1 = MAX(j-1, 1)
        dsy = (jp1 - jm1) * dy
        DO i = 1,nx
            ip1 = MIN(i+1, nx)
            im1 = MAX(i-1, 1)
            dsx = (ip1 - im1) * dx
            mm = msft(i,j)*msft(i,j)
            dudy = 0.5D0*(u(i,jp1,k)/msfu(i,jp1) + u(i+1,jp1,k)/msfu(i+1,jp1) - &
                  u(i,jm1,k)/msfu(i,jm1) - u(i+1,jm1,k)/msfu(i+1,jm1))/dsy*mm
            dvdx = 0.5D0*(v(ip1,j,k)/msfv(ip1,j) + v(ip1,j+1,k)/msfv(ip1,j+1) - &
                  v(im1,j,k)/msfv(im1,j) - v(im1,j+1,k)/msfv(im1,j+1))/dsx*mm
            vorter = dvdx - dudy
            vor(i,j,k) = vorter*1.D5
        END DO
    END DO
END DO
```
代码中jp1,jm1,ip1,im1的出现是为了在模式边界上采用one-sided差分求导而中间采用二倍格距中央差分求导而设计的。

## 参考网站

[wrf-python][]、[南信大段明铿老师][]课件

特别感谢南京大学的[weinihou][]的答疑解惑和耐心指导，没有你便不会有这篇博客。


[C_Grid]: /images/Arakawa_C.png
[vorticity]: /images/vorticity.png
[divergence]: /images/divergence.png
[正交曲线坐标系]: https://en.wikipedia.org/wiki/Curvilinear_coordinates
[wrf-python]: https://github.com/NCAR/wrf-python/blob/develop/fortran/wrf_pvo.f90
[weinihou]: https://github.com/weinihou
[南信大段明铿老师]: https://baike.baidu.com/item/%E6%AE%B5%E6%98%8E%E9%93%BF/16543016?fr=aladdin